# CaseFlow Repository Context

Fast-orientation summary for an AI agent about to work across CaseFlow repositories. Verified against source as of **2026-09-12**. This file links to detail rather than duplicating it — read the linked doc before making a change in its area.

## Repositories

| Repo | Role | Detail |
|---|---|---|
| `caseflow-be` | Backend — domain/API source of truth | [repository-map.md](repository-map.md#caseflow-be-backend), [backend/](backend/) |
| `caseflow-fe` | Web UI | [repository-map.md](repository-map.md#caseflow-fe-frontend), [frontend/](frontend/) |
| `caseflow-ai-service` | AI orchestration, called only by `caseflow-be` | [repository-map.md](repository-map.md#caseflow-ai-service-ai-service), [ai-service/](ai-service/) |
| `caseflow-mobil` | Mobile client, same backend contract as web | [repository-map.md](repository-map.md#caseflow-mobil-mobile), [mobile/](mobile/) |
| `caseflow-central-brain` | This repo — no application code | [../README.md](../README.md) |

## System Architecture

`caseflow-be` is a **modular monolith** (not microservices) that is the dependency root for the whole system. `caseflow-fe` and `caseflow-mobil` are pure REST clients of it with no domain data of their own. `caseflow-ai-service` is a second, independently-deployed service called only by `caseflow-be` (REST always; Kafka optionally, disabled by default). Full detail: [docs/architecture/system-overview.md](../docs/architecture/system-overview.md).

## Repository Responsibilities

`caseflow-be` owns all relational (PostgreSQL) and email-document (MongoDB) data, plus attachment binaries (MinIO/local). `caseflow-ai-service` owns only its Qdrant vector index (a derived copy) and a Postgres ingestion-job log — never ticket/customer data of record. `caseflow-fe`/`caseflow-mobil` are stateless clients (session/UI state only, in `localStorage`/`expo-secure-store` respectively). See [repository-map.md](repository-map.md) for full per-repo module lists.

## Communication

REST/JSON + JWT Bearer is the default for everything. An optional Kafka lane exists between `caseflow-be` (producer) and `caseflow-ai-service` (consumer) for async AI ingestion, **disabled by default on both sides**. Full detail: [dependency-map.md](dependency-map.md), [integration-map.md](integration-map.md).

## Data Ownership

- PostgreSQL, MongoDB, object storage → `caseflow-be` only.
- Qdrant vector index, `ai_ingestion_job` Postgres table → `caseflow-ai-service` only (derived data, not system of record).
- No repository other than these two persists CaseFlow domain data.
- **No multi-tenancy anywhere** — `Customer` is a business concept (the external company a ticket belongs to), not a SaaS isolation boundary. One shared database serves all customers.

## Authentication

JWT Bearer issued by `caseflow-be` (`POST /api/auth/login`); access token 1h, refresh token 7d (DB-backed, rotated on refresh). `caseflow-fe` and `caseflow-mobil` use the identical contract, but **only mobile actually implements silent refresh** — the web frontend stores the refresh token and never uses it, so any 401 force-logs-out a web user immediately. `caseflow-ai-service` has no authentication as shipped (scaffolding exists, disabled by default) — see [ADR-0002](../decisions/0002-ai-service-no-auth-p1.md). Authorization everywhere is `permissionCodes`-based, never role name.

## AI Architecture

Two services cooperate: `caseflow-be`'s `ai` module (REST client with circuit breaker + retry + Postgres response cache, no Spring AI) calls `caseflow-ai-service` (Spring AI `ChatClient` → Ollama, `VectorStore` → Qdrant). **Only 2 of the 4 AI-assist capabilities are actually RAG** — similar-cases (pure retrieval) and policy-guidance (retrieval + generation, with an anti-hallucination guard). Ticket-summary and reply-draft are plain LLM completions built from caller-supplied data, despite living on the same API surface. The web frontend only calls summary and reply-draft today; similar-cases/policy-guidance are fully implemented on both backend repos but explicitly not yet wired into any client UI. Full detail: [docs/architecture/ai-service.md](../docs/architecture/ai-service.md).

## Event Architecture

Kafka exists (as of this pass) but is **optional and disabled by default** — 3 topics for async AI ingestion (ticket/policy/template), producer in `caseflow-be`, consumer in `caseflow-ai-service`, no idempotency/dedup on the consumer side yet. The dominant async mechanism *inside* `caseflow-be` is a Postgres-backed durable job queue (`IntegrationJob`, used for Jira/Slack/Teams/webhook delivery and AI-ingest retry), not a pub/sub bus. Full detail: [contracts/events/README.md](../contracts/events/README.md).

## API Boundaries

`caseflow-be` is the only repository any client (FE, mobile) may call directly. `caseflow-ai-service` is called only by `caseflow-be` — never by FE/mobile, by convention and code comment on both clients, not by an enforced network boundary visible in either repo. Full contract: [contracts/api/README.md](../contracts/api/README.md), [backend/frontend-contract.md](backend/frontend-contract.md) (current for auth/ticket/email core; `TODO: Verify` for Jira/SLA/tags/automation/dashboard endpoint groups — see [integration-map.md](integration-map.md)).

## Important Business Flows

- **Email → ticket**: durable two-stage pipeline (webhook or IMAP poll → `EmailIngressEvent` RECEIVED → async routing/threading/ticket-creation). Customer-rule-based routing only, no Contact lookup ([ADR-0001](../decisions/0001-customer-based-email-routing.md)). Full flow: [backend/email-flow.md](backend/email-flow.md).
- **Ticket lifecycle**: manual + system-triggered status transitions validated against an explicit matrix, SLA due-date tracking, optional Jira linking, tagging, automation rules (`UNKNOWN: exact trigger/action model`). Full flow: [backend/ticket-rules.md](backend/ticket-rules.md).
- **AI assist**: backend-mediated only, always degrades gracefully (`available:false`) rather than propagating an AI-service failure to a client.

## Common Cross-Repository Changes

See [../agents/CROSS-REPOSITORY-CHANGE.md](../agents/CROSS-REPOSITORY-CHANGE.md) for the standard workflow. Most common triggers: a backend DTO/enum/permission-code change (must update `contracts/api/`, `backend/frontend-contract.md`, and flag breaking changes); a new AI-assist capability wired into a client (FE/mobile consuming `similar-cases`/`policy-guidance` would be the next likely one, since both already exist server-side); enabling the Kafka async lane in either/both of `caseflow-be`/`caseflow-ai-service` (currently off by default in both — enabling one without the other is a deployment misconfiguration, not a supported state).

## Known Constraints

- `caseflow-ai-service` must never be reachable from anything other than `caseflow-be` (no auth as shipped).
- No Redis anywhere — rate limiting and AI response caching are single-node/in-process/Postgres-backed; this is a real scaling constraint if `caseflow-be` is ever run with >1 replica (its own `k8s/hpa.yaml` implies that's anticipated).
- `@Scheduled` jobs in `caseflow-be` (IMAP poll, SLA breach check, AI-ingest retry) have no distributed-lock guard beyond DB-level claiming on the job-queue-shaped ones.

## Known Gaps

- Frontend never uses its issued refresh token (mobile does).
- Frontend's `/admin/sla-policy` page is a non-functional stub despite the backend having full SLA policy CRUD.
- Frontend's AI surface omits `similar-cases`/`policy-guidance` despite both existing end-to-end server-side.
- Mobile: push notifications and biometric unlock are both UI/flag-only, not functionally wired; ticket workflow is entirely read-only.
- Kafka consumer side (`caseflow-ai-service`) has no idempotency/dedup; `RagSearchService` is dead code there.
- `caseflow-be`'s Jira integration endpoints/config were not verified in per-field detail this pass (`UNKNOWN`).

## Open Questions

- Whether Testcontainers-based backend integration tests actually run in CI, given a CI comment claiming otherwise.
- The automation-rules engine's exact trigger/condition/action model.
- P2 service-to-service auth timeline for `caseflow-ai-service` (scaffolding exists on the AI-service side; `caseflow-be`'s client does not send the header yet).
