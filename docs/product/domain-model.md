# Domain Model

## Purpose
This document defines shared domain language and business entities used across repositories.

## What belongs here
Canonical terms, entities, relationships, and lifecycle definitions at product/domain level.

## Who should use this
All repository contributors and AI agents producing cross-repository behavior.

## What should NOT be stored here
Database schemas, ORM mappings, or implementation-specific model classes — see [repos/backend/domain-specs/](../../repos/backend/domain-specs/) for full per-entity field-level specs (currently covers the original 11 core entities; the additions below are `TODO: Verify` for field-level detail).

## Domain glossary
- **Ticket** — the central aggregate. Not equal to an email; a ticket may have multiple linked email documents. Has a stable external `publicId` (UUID) in addition to its sequential `ticketNo` (`TKT-0000001`) — see [ADR-0003](../../decisions/0003-sequential-ticket-numbers-public-uuid.md). Supports merge/split (`parentTicketId`, `mergedAt`, `mergedBy`).
- **Customer** — the external organization/account a ticket belongs to. Owns email routing configuration. **Not a multi-tenancy/isolation boundary** — a business-data concept only; all customers share one database.
- **Contact** — a person at a customer. Does **not** drive email routing (see [ADR-0001](../../decisions/0001-customer-based-email-routing.md)).
- **User** — an internal actor (agent/admin/supervisor/viewer) who can be assigned tickets.
- **Group** — an internal queue/team a ticket or user can belong to.
- **Assignment** — the current owning user/group of a ticket. Only one active assignment per ticket.
- **Transfer** — a change of ticket ownership between groups/users; does not automatically change ticket status.
- **Note** — an internal comment on a ticket. Not an email reply. Supports @-mentions.
- **EmailDocument / EmailIngressEvent / OutboundEmailDispatch** — the email platform's record of inbound/outbound messages linked to a ticket.
- **Attachment** — binary content (in object storage) with metadata owned by the ticket.
- **Tag** — a label a ticket can carry (many-to-many via `TicketTag`); tags have their own management/activation lifecycle. *(Added since the last full doc pass — field-level spec is `TODO: Verify`.)*
- **SLA Policy / SLA Event Log** — a configured due-date policy (first-response, resolution) applied to a ticket at creation; breaches are detected by a scheduled job and logged. First-response/resolution due dates and actuals live directly on `Ticket`.
- **Automation Rule** — a backend-configured rule that can act on tickets automatically. `UNKNOWN: exact trigger/condition/action model` — not verified in depth.
- **Notification** — an in-app, per-user notification record (distinct from a Note or an email); read/unread state, surfaced via polling on both web and mobile.
- **Jira Link (`TicketJiraLink`) / Jira Config** — a ticket's linked external Jira issue, and the per-deployment Jira connection configuration.
- **Notification Channel Config** — a configured outbound delivery target (Slack, Teams, or generic webhook) for system-originated notifications, distinct from the in-app Notification above.
- **Mail Template** — a reusable, previewable canned-response template for composing email replies.
- **Integration Job** — a durable, retryable unit of asynchronous work (Jira issue creation, Slack/Teams/webhook delivery, AI-ingest retry) processed by a Postgres-backed job queue — an implementation mechanism, not a user-facing domain concept, but referenced here because it's the shared pattern behind several of the concepts above.

Full field-level specs: [repos/backend/domain-specs/](../../repos/backend/domain-specs/) (ticket, customer, contact, user, group, note, assignment, transfer, attachment, email-documents — the original core set; tag/SLA/automation/notification/Jira/notification-channel specs are not yet written there — `TODO: Define`).

## Core entities
See glossary above. Central aggregate: **Ticket**. Everything else either belongs to a ticket (notes, attachments, assignments, transfers, linked emails, tags, SLA dates, Jira link) or provides context for one (customer, contact, user, group, SLA policy, automation rule).

## Entity relationships
```
Customer 1──* Contact
Customer 1──* CustomerEmailRoutingRule
Ticket   *──1 Customer
Ticket   1──* Assignment (only one active at a time)
Ticket   1──* Transfer
Ticket   1──* Note
Ticket   1──* Attachment (metadata; binary in object storage)
Ticket   1──* EmailDocument (via EmailIngressEvent / OutboundEmailDispatch)
Ticket   *──* Tag (via TicketTag)
Ticket   1──1 TicketJiraLink (optional)
Ticket   *──1 Ticket (optional parentTicketId, for merge)
Ticket   1──* SlaEventLog
User     *──* Group
User     1──* Notification
Assignment *──1 User or Group (assignee target)
```

## Lifecycle/state concepts
Ticket status enum: `NEW, TRIAGED, ASSIGNED, IN_PROGRESS, WAITING_CUSTOMER, RESOLVED, CLOSED, REOPENED`.

**Manual transitions are not purely linear** — the authoritative matrix (`TicketStateMachineService`) is:
```
NEW              → TRIAGED, ASSIGNED, IN_PROGRESS, CLOSED
TRIAGED          → ASSIGNED, IN_PROGRESS, WAITING_CUSTOMER, RESOLVED, CLOSED
ASSIGNED         → IN_PROGRESS, WAITING_CUSTOMER, RESOLVED, CLOSED
IN_PROGRESS      → WAITING_CUSTOMER, RESOLVED, ASSIGNED, CLOSED
WAITING_CUSTOMER → IN_PROGRESS, RESOLVED, CLOSED
RESOLVED         → IN_PROGRESS, REOPENED, CLOSED
CLOSED           → REOPENED
REOPENED         → ASSIGNED, IN_PROGRESS, WAITING_CUSTOMER, RESOLVED, CLOSED
```
Invalid transitions are rejected (`InvalidTicketStateException`). Full rules and the **system-triggered** transitions (inbound/outbound email auto-moves a ticket between `WAITING_CUSTOMER`/`RESOLVED`/`CLOSED`/`IN_PROGRESS`) are in [repos/backend/ticket-rules.md](../../repos/backend/ticket-rules.md).

Email ingress lifecycle: `RECEIVED → PROCESSING → PROCESSED | FAILED | QUARANTINED`. Outbound dispatch lifecycle: `PENDING → SENDING → SENT | FAILED | PERMANENTLY_FAILED`. Full flow: [repos/backend/email-flow.md](../../repos/backend/email-flow.md).

## Open questions
- Automation rule engine's exact condition/action model.
- Whether Tag, SLA Policy, Notification Channel Config, and Jira Link deserve dedicated `repos/backend/domain-specs/` entries (recommended — not yet written).
