# Module Map

This document defines the module boundaries of the CaseFlow backend.
Project architecture is a modular monolith with feature-based packaging.

Base package:

`com.caseflow`

---

## 1. common

Package:
- `com.caseflow.common`

Purpose:
- Shared cross-cutting concerns
- Common exceptions
- Shared infrastructure helpers that do not belong to a single domain

Contains:
- `exception`

Rules:
- Keep this package small
- Do not move business logic here
- Do not use common as a dumping ground

---

## 2. ticket

Package:
- `com.caseflow.ticket`

Purpose:
- Core ticket aggregate
- Ticket lifecycle data
- Ticket querying
- Attachment metadata and history records anchored to ticket

Contains:
- `domain`
- `repository`
- `service`
- `api`
- `dto`

Main domain objects:
- `Ticket`
- `AttachmentMetadata`
- `History`
- `TicketStatus`
- `TicketPriority`

Responsibilities:
- Create and update tickets
- Store core ticket data
- Query ticket list/detail views
- Own ticket-centric history and attachment metadata
- Enforce ticket-level invariants

Rules:
- Ticket is the central aggregate of the system
- Ticket is not equal to email
- Ticket may have multiple related email documents
- Ticket state changes must be validated
- Detail responses may aggregate data from workflow, note, and email modules

Depends on:
- `customer`
- `identity`
- `workflow` (for assignment/transfer context)
- `note` (for detail view composition)
- `email` (for linked email records)

---

## 3. customer

Package:
- `com.caseflow.customer`

Purpose:
- Customer and contact management
- Email routing configuration per customer

Contains:
- `domain`
- `repository`
- `service`
- `api`
- `dto`

Main domain objects:
- `Customer`
- `Contact`
- `CustomerEmailSettings` (unknownSenderPolicy, allowSubdomains, defaultPriority, defaultGroupId)
- `CustomerEmailRoutingRule` (EXACT_EMAIL / DOMAIN sender matching rules)

Responsibilities:
- Manage customer master data
- Manage customer contacts
- Own email routing rules that map incoming senders to customers
- Provide customer context (defaultPriority, defaultGroupId) for inbound ticket creation

Rules:
- Customer owns contact relationship
- Contact data does NOT drive email routing — routing is rule-based (CustomerEmailRoutingRule)
- Keep customer module CRUD-focused
- `MatchingStrategy` enum and contact-centric settings (trustedContactsOnly, autoCreateContact) are removed

Depends on:
- no business dependency required
- may be referenced by `ticket` and `email`

Used by:
- `ticket`
- `email`

---

## 4. identity

Package:
- `com.caseflow.identity`

Purpose:
- Internal actor model
- Users and groups for assignment/routing

Contains:
- `domain`
- `repository`
- `service`
- `api`
- `dto`

Main domain objects:
- `User`
- `Group`
- `GroupType`

Responsibilities:
- Manage internal users
- Manage internal groups/queues
- Support ticket assignment targets
- Represent operational ownership

Rules:
- Identity does not own ticket workflow
- Identity only provides assignee/owner references
- Authentication/authorization can evolve later without polluting core domain

Depends on:
- no business dependency required

Used by:
- `ticket`
- `workflow`

---

## 5. workflow

Package:
- `com.caseflow.workflow`

Purpose:
- Process-oriented ticket actions
- Assignment, transfer, and state orchestration

Subpackages:
- `assignment`
- `transfer`
- `state`
- `history`
- `repository`
- `domain`

Main domain objects:
- `Assignment`
- `Transfer`

Responsibilities:
- Assign ticket to a user or group
- Reassign tickets
- Transfer tickets between groups/users
- Validate ticket state transitions
- Record workflow-driven history events

Rules:
- Workflow is process-driven, not simple CRUD
- Only one active assignment per ticket
- Transfer does not automatically change status unless explicitly required
- Assignment and transfer actions must be written to history
- State machine logic belongs here, not in controllers

Depends on:
- `ticket`
- `identity`

Used by:
- `ticket` detail orchestration
- controllers handling assignment/transfer/state actions

---

## 6. note

Package:
- `com.caseflow.note`

Purpose:
- Internal notes attached to tickets

Contains:
- `domain`
- `repository`
- `service`
- `api`
- `dto`

Main domain objects:
- `Note`
- `NoteType`

Responsibilities:
- Add internal notes
- Store operational comments
- Support ticket investigation and collaboration

Rules:
- Notes belong to tickets
- Notes are not email replies
- Note module should remain focused and simple

Depends on:
- `ticket`
- optionally `identity` for author information

Used by:
- `ticket` detail views

---

## 7. email

Package:
- `com.caseflow.email`

Purpose:
- Email ingestion and parsing
- Persist raw/parsed email document
- Resolve email-to-ticket linkage

Contains:
- `document`
- `service`
- `api`
- `dto`

Main domain objects:
- `EmailDocument` (MongoDB)
- `EmailIngressEvent` (JPA — enriched with threading headers, body fields)
- `EmailMailbox` (JPA — includes IMAP polling config)
- `OutboundEmailDispatch` (JPA)

Responsibilities:
- Ingest emails via webhook (`POST /api/emails/ingest`) or IMAP polling (`ImapMailboxPoller`)
- Durable two-stage pipeline: Stage 1 receives/stores event; Stage 2 routes + creates/links ticket
- Resolve thread via In-Reply-To / References headers (`EmailThreadingService`)
- Route incoming sender to customer via `CustomerEmailRoutingRule` — **no contact lookup**
- Create or link tickets; apply customer defaults (priority, group) from `CustomerEmailSettings`
- Manage outbound replies via durable dispatch queue

Rules:
- Email is an external input channel
- Email must not become the primary aggregate
- One ticket can have multiple email documents
- Routing is CUSTOMER-based; Contact records are not in the routing path
- IMAP passwords are write-only (never returned in responses, never logged)
- Large raw content stays in email/document storage shape, not in every API response

Depends on:
- `ticket`
- `customer`
- optionally `storage`

---

## 8. storage

Package:
- `com.caseflow.storage`

Purpose:
- Object storage abstraction for binary content

Contains:
- `service`

Responsibilities:
- Upload/download file content
- Provide storage abstraction for attachments
- Keep storage provider details isolated

Rules:
- Storage does not own business metadata
- Binary content belongs here
- Attachment metadata belongs to `ticket`
- Do not leak provider-specific API to upper layers

Used by:
- `email`
- `ticket`
- future attachment upload endpoints

---

# Modules Added Since the Original Map (verified 2026-09-12)

The sections above (`common` through `storage`) were the original 8-module map. Source inspection on 2026-09-12 confirmed the backend has grown substantially beyond them. These additions are documented here at a summary level; they have not yet received the same per-module rules/depends-on treatment as the original 8 — treat that as a documentation gap to close, not evidence the modules themselves are undocumented elsewhere (see [current-state.md](current-state.md) and [../integration-map.md](../integration-map.md) for what is independently confirmed about each).

## 9. sla
Package: `com.caseflow.sla`. SLA policy configuration (`SlaPolicyConfig`) and breach detection (`SlaEventLog`, `SlaBreachCheckerJob` — a `@Scheduled` job, default 5 min interval). First-response/resolution due dates and actuals are stamped directly onto `Ticket` at creation from the applicable policy. `SlaPolicyController` exposes full CRUD at `/api/admin/sla/policies` plus a `/backfill` endpoint. **The frontend does not yet build a UI on this CRUD** — see [../../docs/architecture/frontend.md](../../docs/architecture/frontend.md).

## 10. automation
Package: `com.caseflow.automation`. A rules engine (`AutomationRuleController`, `AutomationRule` entity, `automation/engine`) that can act on tickets automatically. `UNKNOWN: exact trigger/condition/action model` — not inspected in depth in the 2026-09-12 pass.

## 11. notification
Package: `com.caseflow.notification`. In-app, per-user notification records (`UserNotification` entity) — distinct from `integration/notification` below. `NotificationController` (`/api/notifications`): list, unread-count, mark-read, mark-all-read. Polled (not pushed) by both `caseflow-fe` and `caseflow-mobil`.

## 12. integration
Package: `com.caseflow.integration`. Two sub-modules, both delivered via a shared durable job queue:
- `integration/jira` — `JiraController` (`/api/tickets/{ticketPublicId}/jira`: status/create/retry), `JiraAdminController` (config), `JiraConfig`/`TicketJiraLink` entities. `UNKNOWN: exact outbound Jira REST endpoints` — not verified in depth.
- `integration/notification` — Slack/Teams/generic-webhook channel config (`NotificationChannelConfig` entity, `NotificationChannelConfigController`) and delivery processors (`{Slack,Teams,Webhook}NotificationJobProcessor`).
- Shared mechanism: `IntegrationJob` entity — a Postgres-backed outbox/worker-queue pattern (`PESSIMISTIC_WRITE` + `SKIP LOCKED` claiming, idempotency keys, attempt/backoff bookkeeping), processed by `integration/scheduler/IntegrationJobWorker` (`@Scheduled`, default 30s). This is the dominant async mechanism in the codebase — not Kafka, not a pub/sub bus.

## 13. ai
Package: `com.caseflow.ai`. Client to the separate `caseflow-ai-service` — **not** an embedded LLM client (no Spring AI dependency here). `AiAssistantController` (`/api/tickets/{ticketId}`: `ai-summary`, `ai-reply-draft`, `ai-similar-cases`, `ai-policy-guidance`) enforces: FE never calls the AI service directly; responses are backend-owned stable DTOs, not raw AI payloads; AI failures degrade gracefully (`200` with `metadata.available=false`); AI must never auto-send email, change ticket status, or assign tickets (documented invariants in the controller's own Javadoc — treat as binding). `CaseflowAiClient` wraps the synchronous REST call in a circuit breaker (Resilience4j) + retry (Spring Retry). `TicketAiResponseCache` (Postgres) caches responses. An optional Kafka producer (`KafkaAiEventPublisher`, disabled by default) provides an async ingestion lane — see [ADR-0004](../../decisions/0004-optional-kafka-ai-ingestion-lane.md).

## Ticket module additions
The original `ticket` module description above predates: tags (`TicketTagController`, `Tag`/`TicketTag` entities), dashboard/reporting (`DashboardController`, `AdminReportController`, `CustomerReportController`), queue views (`QueueController`), bulk actions (`BulkTicketActionController`), and ticket merge/split (`parentTicketId`, `mergedAt`, `mergedBy` fields on `Ticket`). These live in the `ticket` package alongside the original core aggregate — no new top-level module was created for them.

---

# High-Level Dependency Direction

Allowed dependency direction:

- `customer` → none
- `identity` → none
- `ticket` → `customer`, `identity`
- `workflow` → `ticket`, `identity`
- `note` → `ticket`, optionally `identity`
- `email` → `ticket`, `customer`, optionally `storage`
- `storage` → none
- `common` → shared by all

Avoid:
- `customer` depending on `ticket`
- `identity` depending on `workflow`
- `note` containing assignment logic
- `email` containing full ticket business rules
- `common` becoming a hidden business module

---

# Package Placement Guide

Use these package rules when generating code:

- Core entity for ticketing → `ticket.domain`
- Ticket repository/query → `ticket.repository`
- Ticket business logic → `ticket.service`
- Ticket API contract → `ticket.api` / `ticket.dto`

- Customer/contact entities → `customer.domain`
- User/group entities → `identity.domain`

- Assignment/transfer/state logic → `workflow.*`
- Internal notes → `note.*`
- Email document + parsing → `email.*`
- File/object storage abstraction → `storage.service`

- Shared exceptions/utilities → `common.*`

---

# API and DTO Convention

Each feature module should prefer this structure:

- `domain` → JPA/domain model
- `repository` → persistence access
- `service` → business logic
- `api` → controllers
- `dto` → request/response models

Important:
- Controllers should never expose entities directly
- Cross-module API responses should use DTOs
- Aggregated detail views may compose multiple module DTOs in controller/assembler/query layer

---

# Current Central Flows

## Email → Ticket flow
- email arrives (webhook or IMAP poll)
- Stage 1: `IngressEmailData` persisted as `EmailIngressEvent` (RECEIVED) — idempotent on messageId
- Stage 2 (async): loop detection → customer-based routing (thread → exact-email rule → domain rule → policy)
- Routing creates or links ticket; customer defaults (priority, group) applied from `CustomerEmailSettings`
- `EmailDocument` saved to MongoDB; history recorded
- Contact records are NOT consulted during routing

## Ticket assignment flow
- ticket created or triaged
- assignment created for user/group
- active assignment uniqueness enforced
- history recorded

## Ticket transfer flow
- transfer initiated
- ownership target changes
- status may stay same unless business rule says otherwise
- history recorded

## Ticket detail flow
- ticket core data from `ticket`
- assignments/transfers from `workflow`
- notes from `note`
- email summaries from `email`
- attachment metadata from `ticket`

---

# Generation Guidance for AI

When generating code:
1. Respect module boundaries
2. Do not move workflow logic into ticket CRUD services
3. Do not place email parsing into ticket module
4. Do not place binary storage concerns into ticket entity
5. Prefer small focused services
6. Use mappers/assemblers for cross-module detail responses