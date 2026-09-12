# CaseFlow — Current State

**Last updated:** 2026-03-29 (P4 — IMAP hardening: safe onboarding, validation, multi-instance lease, attachment ingestion, connection test)

---

## Build Status

- **Main compile:** PASSING
- **Tests:** 287/287 PASSING (+38 from P4: MailboxValidationServiceTest 17 + ImapMailboxPollerTest 14 + MailboxControllerTest 14 + EmailIngressServiceTest 10 + other updates)
- **Docker image:** Builds successfully (multi-stage, eclipse-temurin:21-jre-alpine)
- **docker-compose config:** VALID (app + postgres + mongo + minio)
- **CI pipeline:** GitHub Actions (`.github/workflows/ci.yml`) — build/test/docker
- **Spring Boot version:** 3.3.0
- **Java:** 21

---

## Implemented Layers

### Domain / Persistence
- All JPA entities: Ticket, History, AttachmentMetadata (+ sourceType), Customer, Contact, User, Group, Note, Assignment, Transfer
- V5 JPA entities: EmailMailbox, EmailIngressEvent, OutboundEmailDispatch, CustomerEmailSettings, CustomerEmailRoutingRule
- MongoDB document: EmailDocument (+ direction, mailboxId, customerId, providerEventId, bodyPreview)
- V9 migration: email platform tables (mailboxes, ingress events, dispatches, customer email settings, routing rules, attachment source_type column)
- **V10 migration:** mailbox operational metadata columns (displayName, defaultGroupId, defaultPriority, lastSuccessfulInboundAt, lastSuccessfulOutboundAt), customer_email_settings extended columns (allowSubdomains, defaultGroupId, defaultPriority), V6 permission seeds for all starter roles
- **V11 migration:** `ticket_no_seq` PostgreSQL sequence for collision-free sequential ticket numbers (`TKT-0000001` format)
- **V12 migration:** `updated_at` column on `customer_email_routing_rules` + `@PreUpdate` lifecycle hook
- **V13 migration:** IMAP columns on `email_mailboxes` (imapHost/Port/Username/Password/UseSsl/Folder, pollingEnabled, pollIntervalSeconds, lastSeenUid, lastPollAt, lastPollError); threading/body columns on `email_ingress_events` (inReplyTo, rawReferences, rawReplyTo, rawCc, textBody, htmlBody, envelopeRecipient); DROP contact-centric columns from `customer_email_settings` (matchingStrategy, trustedContactsOnly, autoCreateContact)
- All Spring Data repositories in place including SKIP LOCKED queries for workers

### Service Layer
- TicketService, TicketQueryService — full CRUD + state transitions
- CustomerService, ContactService — CRUD + activate/deactivate + findAll
- UserService, GroupService — CRUD + activate/deactivate + findAll
- NoteService — add, getById, listByTicket
- AssignmentService — assign/reassign/unassign/getActive
- TransferService — transfer/history
- TicketHistoryService — all history recording methods
- TicketStateMachineService — validated state transitions
- ~~EmailProcessingServiceImpl~~ — **deleted in V8** (was using ContactRepository for routing; replaced by two-stage pipeline)
- EmailDocumentQueryService — read-only email queries
- AttachmentService — metadata management + binary upload/download via ObjectStorageService
- **V5 email platform:**
  - EmailIngressService (two-stage pipeline: receiveEvent → processEvent, idempotent)
  - EmailRoutingService (customer-based routing: thread → exact-email rule → domain rule → policy; **no contact lookup**)
  - EmailThreadingService (In-Reply-To → References → messageId chain)
  - LoopDetectionService (auto-reply/bounce/mailer-daemon detection)
  - EmailMailboxService (mailbox CRUD)
  - EmailIngressEventQueryService (read queries, replay trigger)
  - CustomerEmailSettingsService (per-customer unknown-sender policy + routing rules)
  - SmtpEmailSender (optional JavaMailSender, permanent failure if unconfigured)
  - EmailDispatchService (durable outbound queue management)
  - EmailReplyService (orchestrates outbound customer replies)
  - EmailMetrics (Micrometer counters for inbound/outbound)
  - EmailIngressRetryScheduler (SKIP LOCKED batch worker, processes RECEIVED + retries FAILED)
  - OutboundDispatchScheduler (SKIP LOCKED batch worker, sends PENDING + retries FAILED)
  - **V6 additions:** `quarantineEvent(id, reason)` + `releaseEvent(id)` on EmailIngressService; `EmailMailboxService.activate/deactivate`; `CustomerEmailSettingsService.updateRule`; `EmailDispatchService.getById`; `EmailIngressEventQueryService.findByMailboxId`
  - **V7 additions:** `EmailIngressServiceImpl` stamps `lastSuccessfulInboundAt` on mailbox after CREATE_TICKET/LINK_TO_TICKET; `OutboundDispatchScheduler` stamps `lastSuccessfulOutboundAt` via `findByAddress`; `EmailIngressEventQueryService.searchPaged()` for paginated list; `TicketRepository.nextTicketSeq()` native query
  - **V8 additions:** `IngressEmailData` record (13-field data carrier for receiveEvent); `EmailIngressEvent` enriched with threading headers (inReplyTo, rawReferences, rawReplyTo, rawCc) and body fields (textBody, htmlBody, envelopeRecipient); `EmailMailbox` IMAP fields (11 columns); `ImapMailboxPoller` (UID-based polling, duplicate-safe, parses multipart); `ImapPollingScheduler` (@Scheduled, per-mailbox interval check); `EmailMailboxService.findPollingEnabled()`; routing threading bug fixed (was passing null,null — now passes actual headers); `EmailIngressServiceImpl.applyCustomerDefaults()` applies defaultPriority/defaultGroupId from CustomerEmailSettings
  - **P4 additions (IMAP hardening):**
    - `InitialSyncStrategy` enum (START_FROM_LATEST default, BACKFILL_ALL) — controls first-poll onboarding behaviour; stored on `EmailMailbox`; exposed in API
    - `MailboxValidationService` — enforces: polling requires IMAP+POLLING mode; WEBHOOK/SMTP_RELAY cannot enable polling; IMAP polling requires host/port/username/password/folder; `pollIntervalSeconds` 30–86400
    - `InvalidMailboxConfigException` → 422 in `GlobalExceptionHandler`; validation called at create/update/activate
    - `EmailMailbox` — added `pollLockedBy` + `pollLeasedUntil` columns; `ImapPollingScheduler` claims DB-level lease before each poll via `tryClaimMailbox` JPQL UPDATE; lease TTL configurable (default 10min); lease always released in `ImapMailboxPoller.pollMailbox` finally block
    - `ImapMailboxPoller` — first-poll logic for START_FROM_LATEST (advance cursor to current max UID, no history); attachment extraction from MIME parts; binaries stored in object storage; lock release in finally
    - `IngressEmailData` — added 14th field `List<IngressAttachmentData> attachments`; propagated through Stage 1 (serialized to `attachments_json` TEXT column on `EmailIngressEvent`) and Stage 2 (populates `EmailDocument.attachments` + creates JPA `AttachmentMetadata` records)
    - `POST /api/admin/mailboxes/{id}/test-connection` — proactive IMAP health check; returns `MailboxConnectionTestResponse(success, message, testedAt)`; 200 always, success/failure in body; no DB side effects; passwords never exposed
    - **V14 migration** — `initial_sync_strategy`, `poll_locked_by`, `poll_leased_until` on `email_mailboxes`; `attachments_json` on `email_ingress_events`

### API Layer
- 15 REST controllers (Auth, Ticket, Customer, Contact, User, Group, Note, Assignment, Transfer, EmailDocument, Attachment, MailboxController, IngressEventController, TicketEmailController, CustomerEmailSettingsController)
- All controllers use constructor injection, @Valid, ResponseEntity
- All controllers tagged with @Tag for OpenAPI grouping
- Audit fields (createdBy, performedBy, assignedBy, transferredBy) removed from request bodies — resolved from SecurityContext
- **V6 permission model:** MailboxController → PERM_EMAIL_CONFIG_VIEW/MANAGE; IngressEventController → PERM_EMAIL_OPERATIONS_VIEW/MANAGE; CustomerEmailSettingsController → PERM_EMAIL_CONFIG_VIEW/MANAGE; TicketEmailController → PERM_TICKET_EMAIL_VIEW / PERM_TICKET_EMAIL_REPLY_SEND (via TicketAuthorizationService)
- **V7:** `GET /api/admin/mailboxes?activeOnly=true` filter added; `GET /api/admin/ingress-events` returns `PagedResponse<IngressEventResponse>` (size=20, sort=receivedAt DESC)

### DTOs
- ~50 Java record DTOs across all modules
- Full Jakarta validation annotations on request DTOs

### Mappers
- 11 MapStruct interfaces across all modules
- unmappedTargetPolicy = ERROR, unmappedSourcePolicy = IGNORE

---

## Infrastructure Added (this session)

### Application Entry Point
- `CaseFlowApplication.java` — main Spring Boot application class

### Configuration
- `application.properties` — PostgreSQL, MongoDB, Flyway, Springdoc, storage, CORS config
- `application-dev.properties` — debug logging, dev-only flags

### Global Exception Handling
- `ErrorResponse` record — standard JSON error shape (timestamp, status, error, code, message, path, details, requestId)
- `FieldViolation` record — per-field validation error (field, message, rejectedValue, code)
- `GlobalExceptionHandler` (@RestControllerAdvice) — covers: not found, conflict, invalid state, validation, malformed request, missing params, type mismatch, auth exceptions, unexpected errors

### Security (V2 — JWT)
- `SecurityConfig` — Spring Security, stateless JWT Bearer auth
- `JwtAuthenticationFilter` — reads `Authorization: Bearer` header, sets Long principal
- `JwtTokenService` — JJWT 0.12.6 HS256, access token (1h), refresh token (7d)
- `RefreshToken` JPA entity + `RefreshTokenRepository` — DB-backed refresh tokens (stored as SHA-256 hash)
- `CaseFlowUserDetails` + `CaseFlowUserDetailsService` — backed by `users` table
- `AuthService` + `AuthController` — login, refresh (rotation), logout, /me
- `SecurityContextHelper` — `requireCurrentUserId()` for controllers
- `CorrelationIdFilter` — sets MDC `correlationId`, echoes `X-Correlation-Id` response header
- Role-based access: VIEWER=GET, AGENT=POST/PUT/PATCH, ADMIN=DELETE
- `AuthenticationEntryPoint` returns 401 for unauthenticated requests
- CORS configured for localhost:3000 and localhost:5173
- Swagger/OpenAPI + actuator endpoints publicly accessible

### OpenAPI / Swagger
- `OpenApiConfig` — API info, bearerAuth JWT security scheme, module tags
- All controllers annotated with @Tag
- Swagger UI available at /swagger-ui.html

### Pagination & Search
- `PagedResponse<T>` record — wraps `List<T> items, page, size, totalElements, totalPages`
- `TicketSpecification` — JPA Specifications for status, priority, userId, groupId, customerId, search, date range
- `TicketRepository` implements `JpaSpecificationExecutor<Ticket>`
- `TicketQueryService.search(...)` — builds dynamic spec chain, returns `Page<Ticket>`
- `TicketController.listTickets(...)` — 12 optional query params, returns `PagedResponse<TicketSummaryResponse>`

### Attachments (V2)
- `AttachmentController` — `POST /api/attachments/upload` (multipart, 25 MB limit), GET metadata/by-ticket, GET download (streams binary), DELETE
- `AttachmentService` — `upload()`, `download()`, `getById()`, `delete()` methods
- Object key pattern: `tickets/{ticketId}/{uuid}_{sanitizedName}`

### Email Ingest (V2)
- `POST /api/emails/ingest` — accepts `IngestEmailRequest`, idempotent on `messageId` (409 on dup)
- Real thread resolution via `inReplyTo` / `references` header chain
- `DuplicateEmailException` → 409 Conflict
- `EmailDocumentRepository.findByMessageId()` for idempotency check

### Storage
- `ObjectStorageService` interface — store/retrieve/delete/exists
- `LocalFileStorageService` — filesystem implementation with path traversal protection
- `StorageProperties` (@ConfigurationProperties) — rootPath, provider
- `AttachmentService` updated to accept and use ObjectStorageService
- New methods: `upload()` (store binary + save metadata), `download()` (retrieve stream)

### Database Migrations
- `db/migration/V1__init_schema.sql` — complete baseline schema for all 11 relational tables
- Partial unique index on assignments for active assignment constraint
- Indexes on all FK/lookup paths

### Seed Data
- `DevDataLoader` (@Component @Profile("dev")) — idempotent ApplicationRunner
- Seeds: 3 groups, 3 users, 2 customers, 3 contacts, 3 tickets, 2 notes, 1 sample email

### Tests
- **Service tests (26):**
  - TicketServiceTest — 7 tests (create, close, invalid state, not found, update, changeStatus, reopen)
  - TicketQueryServiceTest — 6 tests (getById found/not found, getByTicketNo, listByStatus, findAll)
  - AssignmentServiceTest — 6 tests (assign, conflict, not found, unassign, reassign, getActive empty)
  - NoteServiceTest — 4 tests (add+history, getById, not found, listByTicket)
  - EmailDocumentQueryServiceTest — 5 tests (findById, not found, findByTicketId empty/found, findByThreadKey)
- **Controller tests (20):**
  - TicketControllerTest — 7 tests (getById 200/404/401, list 200, create 201/400/403)
  - CustomerControllerTest — 5 tests (getById 200/404, list 200, create 201, 401 unauth)
  - NoteControllerTest — 6 tests (add 201/400, getById 200/404, getByTicket 200/401)
  - ContactControllerTest — 10 tests (create 201/400/401, getById 200/404, list, by-email 200/404, by-customer, update 200)
  - AssignmentControllerTest — 7 tests (assign 200/403/401, reassign 200, unassign 204, getActive 200/404)
  - TransferControllerTest — 6 tests (transfer 201/403/401/400, history 200/403)
  - AttachmentControllerTest — 9 tests (upload 201/401/403, getMetadata 200/404, getByTicket, download, delete 204/403)

### FE Integration
- `docs/frontend-contract.md` — compact frontend contract (auth flow, all endpoints, shapes, enums)
- `docs/api-endpoints.md` — full per-endpoint request/response examples
- `docs/api-models.ts` — TypeScript types matching backend Java records
- `docs/api-notes.md` — endpoint overview, error shapes, enum values, CORS, seed data (JWT auth)
- Consistent JSON error responses with FE-friendly field violation structure
- CORS pre-configured for localhost:3000 and localhost:5173
- Enum serialization uses string names by default (Spring Boot default)

---

## Module Boundaries (preserved)

```
customer     → none
identity     → none
ticket       → customer, identity (via FK only, no service cross-calls)
workflow     → ticket, identity (AssignmentService, TransferService, HistoryService)
note         → ticket, workflow.history
email        → ticket (ticketId FK), storage (optional)
storage      → none (AttachmentService depends on ticket.repository)
common       → exception classes + api (ErrorResponse, GlobalExceptionHandler) + security + config + dev
```

---

## Containerisation (added 2026-03-27)

- `Dockerfile` — multi-stage (maven:3.9.6-eclipse-temurin-21-alpine builder → eclipse-temurin:21-jre-alpine runtime), non-root user, `MaxRAMPercentage=75`
- `.dockerignore` — excludes target/, .git, IDE files, logs, storage-data, .env, docs, .ai
- `docker-compose.yml` — app + postgres:16-alpine + mongo:7-jammy, named volumes, health checks, `depends_on: service_healthy`
- `.env.example` — documents all overridable env vars
- `README.md` — added quick-start instructions
- `docs/deployment-local.md` — full local Docker guide
- `application.properties` — all backing-service URLs now env-var driven with localhost defaults preserved

One-command local start: `docker compose up -d`
Seed data: `SPRING_PROFILES_ACTIVE=dev docker compose up -d`
Health check path: `GET /actuator/health` (public, no auth)

## CI/CD (added 2026-03-27)

- `.github/workflows/ci.yml` — GitHub Actions pipeline: build+test (Java 21, Maven) → Docker build → GHCR push on main/master
- `Dockerfile.ci` — CI-specific runtime-only Dockerfile (copies pre-built JAR; faster than multi-stage in CI)
- `Dockerfile.ci.dockerignore` — companion ignore file that does NOT exclude `target/`
- `docs/ci-cd.md` — full CI/CD guide (pipeline overview, registry config, secrets, local commands, env vars)
- `.gitignore` — added `.env`, `*.log`, `logs/`, `storage-data/` exclusions
- `README.md` — added CI badge placeholder and CI/CD section

Pipeline jobs:
1. `test` — `mvn verify` (no real DB needed; all tests are unit/@WebMvcTest), uploads test reports + JAR
2. `docker` — downloads JAR, builds with Dockerfile.ci, pushes to GHCR on main/master (uses GITHUB_TOKEN, no extra secrets)

Image tagging: `sha-<short-sha>`, `latest` (main/master only), branch name

## External Verification Pass — 2026-09-12

The entries above (through P4, 2026-03-29) are the last first-party changelog update to this file. A read-only source inspection on 2026-09-12 (for a central-brain sync, not an implementation change) confirmed the backend has grown substantially since, with **44 Flyway migrations now present (V1–V44)** vs. the V14 baseline this file last documented. Confirmed additions not previously logged here, at a summary level (no version-by-version detail available without the intervening changelog entries this file doesn't have):

- **SLA**: policy config + breach-checking scheduled job, SLA fields on `Ticket`, full CRUD controller (`/api/admin/sla/policies`).
- **Tags**: `Tag`/`TicketTag` entities, tag CRUD + per-ticket assignment endpoints.
- **Automation rules**: a rules engine module (`automation/`) — exact trigger/action model not verified in this pass.
- **In-app notifications**: `UserNotification` entity + controller, polled by both FE and mobile.
- **Jira integration**: issue creation/linking from tickets, admin config, via the durable `IntegrationJob` queue.
- **Notification channels**: Slack/Teams/generic-webhook outbound delivery, same job-queue mechanism.
- **AI integration** (`ai/` module): REST client to the separate `caseflow-ai-service` with circuit breaker + retry + Postgres response cache, plus an **optional Kafka producer** (3 topics, disabled by default) as an async ingestion lane — see [ADR-0004](../../decisions/0004-optional-kafka-ai-ingestion-lane.md).
- **Ticket merge/split**: `parentTicketId`, `mergedAt`, `mergedBy` fields.
- **Dashboard/reporting**: `DashboardController`, `AdminReportController`, `CustomerReportController`, `QueueController`, `BulkTicketActionController`.
- **Mail templates + scheduled email**: reusable reply templates with preview; delayed-send outbound replies.
- **Security hardening**: rate limiting (Bucket4j, in-process), account lockout, security audit log table, HSTS/CSP/security headers.
- **Testing**: Testcontainers (PostgreSQL + MongoDB) now present in the build (~95 test files total) — though see the CI note below.

Confirmed still true / unchanged since P4:
- No Redis anywhere (verified by repo-wide grep).
- No multi-tenancy mechanism (`Customer` remains a business entity, not a tenant-isolation boundary).
- Modular monolith, one deployable unit — not microservices.

New open question surfaced by this pass: CI (`.github/workflows/ci.yml`) comments that tests require no real databases, which is in tension with the Testcontainers (PostgreSQL + MongoDB) dependencies and multiple `*IntegrationTest` classes present in the build — not resolved by this inspection; flagged for the backend team.

Full detail on all of the above: [module-map.md](module-map.md#modules-added-since-the-original-map-verified-2026-09-12), [../../docs/architecture/backend.md](../../docs/architecture/backend.md), [../../docs/architecture/system-overview.md](../../docs/architecture/system-overview.md), [../integration-map.md](../integration-map.md), [../dependency-map.md](../dependency-map.md).

## Remaining Gaps / Known Issues

See [remaining-issues.md](remaining-issues.md).
