# CaseFlow — Remaining Issues

**Last updated:** 2026-03-29 (P4 — IMAP hardening: safe onboarding, validation, multi-instance lease, attachment ingestion, connection test)

---

## Resolved in V2

- ✅ JWT auth replaces HTTP Basic
- ✅ Attachment upload/download endpoints
- ✅ `POST /api/emails/ingest` — idempotent email ingestion
- ✅ `createdBy`/`performedBy` removed from request bodies — resolved from JWT SecurityContext
- ✅ `GET /api/tickets/{id}/detail` — full response with attachments and history
- ✅ `GET /api/tickets` — pagination + JPA Specifications
- ✅ `CorrelationIdFilter` — MDC-based correlation IDs

## Resolved in V5

- ✅ Durable two-stage email ingress pipeline (receiveEvent → processEvent)
- ✅ EmailMailbox, EmailIngressEvent, OutboundEmailDispatch JPA entities
- ✅ EmailRoutingService — deterministic routing (exact-email → domain → contact → policy)
- ✅ EmailThreadingService — In-Reply-To / References header chain resolution
- ✅ LoopDetectionService — auto-reply/bounce/mailer-daemon detection
- ✅ SmtpEmailSender — optional JavaMailSender, permanent failure if unconfigured
- ✅ EmailDispatchService + EmailReplyService — durable outbound queue
- ✅ SKIP LOCKED scheduler workers (EmailIngressRetryScheduler, OutboundDispatchScheduler)
- ✅ CustomerEmailSettings + CustomerEmailRoutingRule entities and services
- ✅ Micrometer email metrics

## Resolved in V6

- ✅ New permissions: EMAIL_CONFIG_VIEW/MANAGE, EMAIL_OPERATIONS_VIEW/MANAGE, TICKET_EMAIL_VIEW, TICKET_EMAIL_REPLY_SEND — added to Permission enum + seeded in V10 migration
- ✅ `GET /api/auth/me` exposes `permissionCodes[]` — FE uses these, not role names
- ✅ EmailMailbox: displayName, defaultGroupId, defaultPriority, lastSuccessfulInboundAt, lastSuccessfulOutboundAt fields
- ✅ CustomerEmailSettings: trustedContactsOnly, autoCreateContact, allowSubdomains, defaultGroupId, defaultPriority fields
- ✅ MailboxController: PATCH activate/deactivate; permissions → PERM_EMAIL_CONFIG_VIEW/MANAGE
- ✅ IngressEventController: POST quarantine/release; list filters (status, mailboxId, ticketId); permissions → PERM_EMAIL_OPERATIONS_VIEW/MANAGE
- ✅ CustomerEmailSettingsController: PUT /rules/{ruleId}; permissions → PERM_EMAIL_CONFIG_VIEW/MANAGE
- ✅ TicketEmailController: GET /thread (primary FE contract), GET /inbound/{id}, GET /outbound/{id}; permissions → TICKET_EMAIL_VIEW / TICKET_EMAIL_REPLY_SEND via TicketAuthorizationService
- ✅ EmailIngressService: quarantineEvent, releaseEvent
- ✅ EmailDispatchService: getById
- ✅ RoutingRuleNotFoundException, DispatchNotFoundException — GlobalExceptionHandler updated
- ✅ V10 Flyway migration for new columns and permission seeds
- ✅ 202/202 tests passing (33 new: MailboxControllerTest expanded, IngressEventControllerTest, TicketEmailControllerTest, CustomerEmailSettingsControllerTest, EmailIngressServiceTest quarantine/release)
- ✅ docs/frontend-contract.md fully updated with V6 permission model and all email endpoints

## Resolved in V8

- ✅ **Contact-based routing removed** — `EmailRoutingService` no longer calls `ContactRepository.findByEmail()`; routing is purely customer-rule-based (CustomerEmailRoutingRule)
- ✅ **Threading null/null bug fixed** — routing and ingress services now pass `event.getInReplyTo()` + `event.getReferencesList()` to `threadingService`; threading headers are persisted on `EmailIngressEvent`
- ✅ **`IngressEmailData` record** — replaces 6-param `receiveEvent` signature; carries all 13 fields (headers, body, envelope)
- ✅ **`EmailIngressEvent` enriched** — added inReplyTo, rawReferences, rawReplyTo, rawCc, textBody, htmlBody, envelopeRecipient; helper methods `getReferencesList()` and `effectiveReplyTo()`
- ✅ **Contact-centric `CustomerEmailSettings` fields removed** — matchingStrategy, trustedContactsOnly, autoCreateContact dropped from entity, DTOs, mapper, service, and V13 migration
- ✅ **`MatchingStrategy` enum deleted** — was obsolete after contact-based routing was removed
- ✅ **Legacy `EmailProcessingServiceImpl` deleted** — was using `ContactRepository` for routing; entirely replaced by two-stage pipeline
- ✅ **IMAP MVP** — `EmailMailbox` extended with 11 IMAP columns; `ImapMailboxPoller` (UID-based, duplicate-safe, multipart parsing); `ImapPollingScheduler` (@Scheduled, per-mailbox interval)
- ✅ **Customer defaults applied to inbound tickets** — `applyCustomerDefaults()` reads `CustomerEmailSettings.defaultPriority` + `defaultGroupId` when creating tickets from email
- ✅ **V13 Flyway migration** — IMAP columns, ingress enrichment columns, drop of 3 contact-centric settings columns
- ✅ **Test suite updated** — `EmailRoutingServiceTest` (removed ContactRepository mock, 8 new routing tests), `EmailIngressServiceTest` (IngressEmailData signature), `CustomerEmailSettingsControllerTest` (no MatchingStrategy), `ImapMailboxPollerTest` (8 guard + failure tests)

---

## Resolved in P4

- ✅ **Safe first-time onboarding** — `InitialSyncStrategy.START_FROM_LATEST` (default): first poll advances cursor to current max UID without ingesting history. `BACKFILL_ALL` available for intentional historical import.
- ✅ **Mailbox validation** — `MailboxValidationService` enforces polling→IMAP+POLLING mode; WEBHOOK/SMTP_RELAY cannot enable polling; complete IMAP credentials required; `pollIntervalSeconds` 30–86400. Called at create/update/activate. Returns 422 `INVALID_MAILBOX_CONFIG`.
- ✅ **Multi-instance safe polling** — DB-level lease (`poll_locked_by`, `poll_leased_until`). `tryClaimMailbox` JPQL UPDATE (CAS). Only one pod polls a mailbox at a time. Lease TTL 10min for crash recovery. Lock always released in `ImapMailboxPoller.pollMailbox` finally block.
- ✅ **IMAP attachment ingestion** — `ImapMailboxPoller` extracts MIME attachments (>25MB skipped), stores binaries in object storage. Metadata flows through `IngressEmailData` → `attachments_json` column → `EmailDocument.attachments` + JPA `AttachmentMetadata` records on ticket.
- ✅ **Connection test / health check** — `POST /api/admin/mailboxes/{id}/test-connection` returns `{success, message, testedAt}`; no DB writes; passwords never in response.
- ✅ **V14 migration** — 4 new columns: `initial_sync_strategy`, `poll_locked_by`, `poll_leased_until` on `email_mailboxes`; `attachments_json` on `email_ingress_events`.
- ✅ **Test coverage** — 287/287 passing. Added: `MailboxValidationServiceTest` (17 tests), `ImapMailboxPollerTest` expanded (14 tests), `MailboxControllerTest` expanded (14 tests).

---

## P1 — Blockers for Production Use

(None)

---

## Resolved in V7

- ✅ `lastSuccessfulInboundAt` stamped by `EmailIngressServiceImpl` after CREATE_TICKET / LINK_TO_TICKET
- ✅ `lastSuccessfulOutboundAt` stamped by `OutboundDispatchScheduler` after successful SMTP send (via `findByAddress`)
- ✅ Ticket numbers are now sequential — `V11__ticket_no_sequence.sql` + `ticket_no_seq` PostgreSQL sequence; format `TKT-%07d`; both `TicketService` and `EmailIngressServiceImpl.createTicketFromEvent` use it
- ✅ MongoDB indexes activated — `spring.data.mongodb.auto-index-creation=true` added; `@CompoundIndex(ticket_received)` added to `EmailDocument`
- ✅ Ingress event list paginated — `GET /api/admin/ingress-events` now returns `PagedResponse<IngressEventResponse>`; defaults `size=20, sort=receivedAt DESC`; `searchPaged()` method on `EmailIngressEventQueryService`
- ✅ Mailbox list `?activeOnly=true` filter added to `GET /api/admin/mailboxes`
- ✅ `CustomerEmailRoutingRule.updatedAt` — `V12__routing_rule_updated_at.sql` + `@PreUpdate` lifecycle hook; exposed in `RoutingRuleResponse`

---

## P2 — Important but Non-Blocking

(None)

---

## P3 — Quality / Nice to Have

### Integration tests
- All 202 tests are unit / @WebMvcTest — no real DB
- Schema errors only surface at deploy time
- Fix: `@SpringBootTest` + Testcontainers for PostgreSQL + MongoDB

### Controller coverage gaps
- ✅ All controllers now have @WebMvcTest coverage (ContactControllerTest, AssignmentControllerTest, TransferControllerTest, AttachmentControllerTest added — 32 new tests)

### Validation messages
- Jakarta defaults ("must not be blank") — consider `ValidationMessages.properties`

### Storage: cloud provider
- `LocalFileStorageService` only; S3/MinIO/Azure Blob via `caseflow.storage.provider=s3`

### API rate limiting
- Not implemented; consider Bucket4j or API gateway

---

## CI/CD — External Dependencies

### No integration tests in CI
- Flyway migration regressions only surface at deployment time
- Add `@SpringBootTest` + Testcontainers profile

### GHCR visibility
- First push may fail if `Settings → Actions → Workflow permissions` is not "Read and write"

### CI badge URL placeholder
- `README.md` contains `your-org/caseflow` — replace once repo is pushed

### Maven not wrapped
- No `mvnw` — run `mvn wrapper:wrapper -Dmaven=3.9.6`

---

## Known Assumptions

- `groups` table name is used as-is in PostgreSQL (not a reserved keyword at table level)
- `performed_by` in `ticket_history` is nullable for system-generated events
- `flyway.baseline-on-migrate=true` — database may have tables from prior `ddl-auto=create` runs
- V10 permission seeds use `WHERE r.code = 'ADMIN'` etc. — safe no-ops if those role codes don't exist
