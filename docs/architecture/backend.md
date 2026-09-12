# Backend Architecture

## Purpose
This document defines backend architecture context relevant to other repositories.

## What belongs here
Backend responsibilities, interfaces, and constraints that affect shared contracts and cross-repo behavior.

## Who should use this
Backend contributors and any AI agent integrating with backend capabilities.

## What should NOT be stored here
Application source code, endpoint implementation details, or internal-only refactoring notes — see [repos/backend/](../../repos/backend/) for the full internal documentation set (module map, rules, domain specs, prompts, skills).

## Responsibility
`caseflow-be` is the single source of truth for the ticket/customer/user domain, email ingestion/dispatch, SLA tracking, tagging, automation rules, and third-party integrations (Jira, Slack/Teams/webhooks). Every other repository (`caseflow-fe`, `caseflow-mobil`, `caseflow-ai-service`'s caller relationship) is a client of its API.

## Components
Modular monolith (`com.caseflow.*`), feature-based packages: `ticket` (central aggregate — includes tags, dashboard/reporting, queue, bulk actions), `email` (ingestion/parsing/routing/dispatch/templates/scheduled-send), `identity` (user/role/group), `customer` (customer/contact + email routing rules), `workflow` (assignment/transfer/state/history), `note`, `sla` (policy config + breach detection), `automation` (rules engine — `UNKNOWN: exact trigger/action model`), `notification` (in-app user notifications), `integration` (`jira` sub-module: issue creation/linking; `notification` sub-module: Slack/Teams/webhook channel delivery — both via a shared durable job queue, `IntegrationJob`), `ai` (client to `caseflow-ai-service`: REST + optional Kafka producer, circuit breaker, retry, Postgres response cache — **not** an embedded LLM client, no Spring AI dependency), `storage` (object storage abstraction), `auth` (JWT), `common` (cross-cutting exceptions/security/config). Full detail: [repos/backend/module-map.md](../../repos/backend/module-map.md).

## Dependencies
PostgreSQL (JPA/Flyway, 44 migrations), MongoDB (email documents), pluggable object storage (local filesystem / MinIO / S3-compatible), optional SMTP relay + IMAP for email, outbound calls to `caseflow-ai-service` (REST mandatory, Kafka optional/disabled-by-default), outbound calls to Jira/Slack/Teams/webhooks (optional, per-tenant/channel config). **No Redis** — verified absent from dependencies, config, and k8s manifests.

## Communication
Exposes REST/JSON over HTTP, JWT Bearer auth (stateless, `SecurityConfig`). Full contract: [repos/backend/frontend-contract.md](../../repos/backend/frontend-contract.md) (current for auth/ticket/email core; newer controller groups — Jira, SLA, tags, automation, dashboard/reports, mail templates, notifications — are `TODO: Verify` in per-field detail, see [repos/integration-map.md](../../repos/integration-map.md)) and [contracts/api/README.md](../../contracts/api/README.md).

## Data ownership
Owns all relational (PostgreSQL) and email-document (MongoDB) CaseFlow data — see [system-overview.md](system-overview.md#data-ownership). No other repository owns or duplicates this data. Single-tenant data model: `Customer` is a business entity, not an infrastructure isolation boundary — no `tenantId`/schema-per-tenant mechanism exists.

## Security considerations
JWT Bearer auth (access 1h + rotating, DB-backed refresh 7d), permission-code-based authorization (`permissionCodes`, ~29 codes, not role name) plus a bean-based ticket-visibility layer (`@ticketAuth`, `TicketScope` enum: `ALL`/`OWN_GROUPS`/`OWN_AND_OWN_GROUPS`/`ASSIGNED_ONLY`). IMAP/SMTP credentials write-only (never returned in API responses or logs). IP-based rate limiting (Bucket4j, in-process — auth 10/60s, `/contacts/by-email` 20/60s, AI endpoints 20/60s, general 300/60s), account lockout support, and a dedicated security audit log table. Full detail: [docs/security/security-overview.md](../security/security-overview.md).

## Observability
`X-Correlation-Id` on every response (MDC-based), Micrometer metrics (email counters + Prometheus registry), `/actuator/health` (+ `/health/readiness`, `/health/liveness` in containerized deployments), OpenAPI/Swagger at `/swagger-ui.html`.

## Failure handling
Calls to `caseflow-ai-service` are wrapped in a circuit breaker (Resilience4j, 50% failure threshold / 10-call window / 30s open wait) + retry (Spring Retry); every failure mode collapses to a single `AiServiceUnavailableException` and the client always gets a graceful `available:false` response, never a raw error. Email dispatch/ingress and integration-job delivery (Jira/Slack/Teams/webhook) use durable, retryable Postgres-backed queues with `SKIP LOCKED`/`PESSIMISTIC_WRITE` claiming, not fire-and-forget calls.

## Testing
JUnit 5 + Testcontainers (PostgreSQL + MongoDB) present in the build; ~95 test files. **Open question**: CI's own comment claims "no real databases required" for the test job, which is in tension with the Testcontainers dependencies and multiple `*IntegrationTest` classes present — not resolved by this pass, flagged for the backend team.

## Known limitations
- **No Redis anywhere** — rate limiting and the AI response cache are single-node/in-process/Postgres-backed. `k8s/hpa.yaml` (autoscaling) exists, which is in tension with several `@Scheduled` jobs (IMAP poll, SLA breach check, email retry/dispatch, integration job worker, AI ingest retry) having no distributed-lock guard beyond DB-level claiming on the job-queue-shaped ones — a real risk if this deployment is ever scaled beyond 1 replica.
- API rate limiting is per-node only for the same reason.
- Jira integration's exact outbound REST contract was not verified in depth (`UNKNOWN`).

## Open questions
- Whether Testcontainers-based integration tests actually run in CI (see Testing above).
- Automation-rules engine's exact trigger/condition/action model.
- Whether/when to enable the Kafka async-ingestion lane (built, disabled by default) — see [contracts/events/README.md](../../contracts/events/README.md).
