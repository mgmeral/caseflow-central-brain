Email to Ticket flow (V8 — customer-based routing, IMAP MVP):

## Inbound ingress paths

### Path A: Webhook (POST /api/emails/ingest)
1. Controller receives `IngestEmailRequest` and builds `IngressEmailData` (13 fields)
2. `EmailIngressService.receiveEvent(data)` — Stage 1

### Path B: IMAP polling
1. `ImapPollingScheduler` runs every N ms (default 30s)
2. For each active polling-enabled mailbox whose `pollIntervalSeconds` has elapsed:
   - Scheduler claims a **DB-level lease** via `tryClaimMailbox` UPDATE (CAS) — only one pod may poll at a time
   - `ImapMailboxPoller.pollMailbox(mailbox)` opens IMAP store
   - **First-time onboarding** (lastSeenUid=null):
     - `START_FROM_LATEST` (default): advances cursor to current max UID — no historical messages ingested
     - `BACKFILL_ALL`: starts from UID 1 — ingests entire inbox history
   - Fetches messages with UID > `lastSeenUid` (UID-based duplicate safety)
   - Parses each message headers + body into `IngressEmailData`
   - Extracts MIME attachments (≤25MB), stores binaries in object storage; passes metadata as `IngressAttachmentData` list
   - Calls `EmailIngressService.receiveEvent(data)` per message
   - Finally block: clears `pollLockedBy`/`pollLeasedUntil` (lock release), updates `lastSeenUid`, `lastPollAt`, `lastPollError`

## Stage 1 — receiveEvent
- Check messageId idempotency (second layer of duplicate safety; returns existing event if found)
- Persist `EmailIngressEvent` with all threading headers and body content:
  - inReplyTo, rawReferences, rawReplyTo, rawCc
  - textBody, htmlBody, envelopeRecipient
  - mailboxId, rawFrom, rawTo, rawSubject
  - `attachments_json` — JSON-serialized list of `IngressAttachmentData` (fileName, objectKey, contentType, size); populated by IMAP poller; null for webhook
- Status: RECEIVED
- `metrics.inboundReceived()`

## Stage 2 — processEvent (async via EmailIngressRetryScheduler)
1. Load event; skip if already PROCESSED
2. Loop detection (`LoopDetectionService.isLoop(from, subject, headers)`)
   - If loop: mark PROCESSED, `metrics.inboundIgnored()`, return
3. Routing (`EmailRoutingService.route(event)`)
   - Passes `event.getInReplyTo()` and `event.getReferencesList()` to threading
   - **Routing precedence (customer-based — no contact lookup):**
     1. Thread: `threadingService.resolveTicketId(inReplyTo, references)` → `LINK_TO_TICKET`
     2. Exact-email rule: `CustomerEmailRoutingRule` with `EXACT_EMAIL` type (case-insensitive, display name stripped)
     3. Domain rule: `CustomerEmailRoutingRule` with `DOMAIN` type (leading `@` stripped)
     4. Unknown sender policy: MANUAL_REVIEW → `QUARANTINE`; IGNORE → `IGNORE`; default → `QUARANTINE`
4. Dispatch on routing result:
   - `QUARANTINE` → mark QUARANTINED, `metrics.inboundQuarantined()`
   - `IGNORE` → mark PROCESSED, `metrics.inboundIgnored()`
   - `LINK_TO_TICKET` → save EmailDocument (with attachment list); create JPA `AttachmentMetadata` records linked to ticket; stamp `lastSuccessfulInboundAt`
   - `CREATE_TICKET` → create new ticket (with `applyCustomerDefaults()` for priority/group); save EmailDocument; create JPA `AttachmentMetadata` records; stamp `lastSuccessfulInboundAt`

## Customer defaults on ticket creation
- `applyCustomerDefaults(ticket, customerId)` looks up `CustomerEmailSettings` for the resolved customer
- Applies `defaultPriority` and `defaultGroupId` if configured

## Thread resolution detail
- `EmailThreadingService.resolveTicketId(inReplyTo, references)`:
  - Looks up `EmailDocument` records matching any of the provided message-ids
  - Returns the ticketId of the most recently matched document
- `EmailThreadingService.resolveThreadKey(inReplyTo, references, messageId)`:
  - Returns a stable thread key for grouping related email documents

## Quarantine management
- `quarantineEvent(id, reason)` — operators quarantine suspicious events
- `releaseEvent(id)` — resets QUARANTINED → RECEIVED for reprocessing

## Mailbox connection test
- `POST /api/admin/mailboxes/{id}/test-connection` — proactive IMAP health check
- Opens IMAP store and folder, reads message count, returns `{success, message, testedAt}`
- Always returns HTTP 200; success/failure communicated in body
- Read-only: no DB writes, no lastPollAt update
- Passwords never exposed in response or logs

## Rules
- Routing is CUSTOMER-based: Contact records are NOT consulted for routing decisions
- Do not rely only on subject for thread matching
- Avoid duplicate ticket creation (UID + messageId two-layer idempotency)
- Email is source of truth for inbound communication
- IMAP passwords are never logged or returned in API responses
- Multi-instance safety: only one pod may poll a mailbox at a time (DB lease + CAS UPDATE)
- First-time onboarding default is START_FROM_LATEST — historical inbox not ingested unless operator explicitly sets BACKFILL_ALL
