# API Contracts

## Purpose
Defines HTTP/API-level shared agreements across repositories.

## What belongs here
Endpoint definitions, request/response shapes, authentication expectations, versioning, and compatibility notes.

## Who should use this
Backend, frontend, mobile, and AI service contributors when changing API behavior.

## What should NOT be stored here
Endpoint implementation code or private service internals.

## Format direction
REST/JSON over HTTP. OpenAPI/Swagger is already in use (`/swagger-ui.html`, `/v3/api-docs` on `caseflow-be`; `/swagger-ui.html`, `/api-docs` on `caseflow-ai-service`) — treat Swagger UI as the live, always-current reference and the documents below as the human-curated contract summary. [openapi-core.yaml](openapi-core.yaml) provides a git-reviewable, machine-readable subset (auth, core ticket, AI-assist) for the highest-traffic cross-repository surface — explicitly partial, not a full spec.

## Authoritative documents
- **caseflow-be ↔ caseflow-fe / caseflow-mobil**: [repos/backend/frontend-contract.md](../../repos/backend/frontend-contract.md) is the primary integration contract, **current and verified for the auth flow and core ticket/email endpoints** as of 2026-09-12. It does **not yet** cover the newer controller groups confirmed to exist in the backend (Jira integration, SLA policy CRUD, tags, automation rules, dashboard/reports, mail templates, scheduled email, notification-channel config, in-app notifications) — those are listed at a summary level in [repos/integration-map.md](../../repos/integration-map.md) and marked `TODO: Verify` for per-field detail pending a dedicated pass against the controllers/DTOs directly. [repos/backend/api-endpoints.md](../../repos/backend/api-endpoints.md) and [repos/backend/api-notes.md](../../repos/backend/api-notes.md) predate that same set of additions and carry the same caveat.
- **caseflow-be → caseflow-ai-service**: [repos/ai-service/README.md](../../repos/ai-service/README.md) lists the 4 ticket-AI endpoints (`summary`, `reply-draft`, `similar-cases`, `policy-guidance`) plus ingest/health endpoints. This is a service-to-service contract only — `caseflow-fe`/`caseflow-mobil` must never call it directly ([ADR-0002](../../decisions/0002-ai-service-no-auth-p1.md)). **Important:** only `similar-cases` and `policy-guidance` are retrieval-augmented; `summary` and `reply-draft` are plain LLM completions — see [docs/architecture/ai-service.md](../../docs/architecture/ai-service.md). Also note the frontend currently only calls `summary`/`reply-draft` — `similar-cases`/`policy-guidance` are implemented but unconsumed by any client today.
- **caseflow-be → caseflow-ai-service (async)**: an optional Kafka lane also exists for ingestion — see [contracts/events/README.md](../events/README.md). Disabled by default on both sides.

## Cross-cutting conventions (all endpoints)
- Auth: JWT Bearer, `Authorization: Bearer <token>`.
- Authorization: gate on `permissionCodes`, never on `roleCode`/`roleName`.
- Pagination: `{ items, page, size, totalElements, totalPages }`, query params `page`/`size`/`sort`/`direction`, defaults `page=0`, `size=20`.
- Errors: consistent `ErrorResponse` shape (`timestamp, status, error, code, message, path, details, requestId`) plus `X-Correlation-Id` response header on every request. Full error code table: [repos/backend/frontend-contract.md](../../repos/backend/frontend-contract.md#error-response).
- Never send `createdBy`/`performedBy`/`assignedBy`/`transferredBy` in request bodies — resolved server-side from the JWT.
- Write-only fields (e.g. `smtpPassword`) are never present in GET responses.

## Contract governance rules
- Do not silently change a shared contract.
- Breaking changes must be explicitly documented (update the affected document above **and** flag it in the PR/task description) and must identify affected repositories.
- Contract changes that reflect a real architectural decision (not just a shape tweak) should get an ADR — see [decisions/](../../decisions/).

## Cross-repository endpoint inventory
See [repos/integration-map.md](../../repos/integration-map.md) for the full verified list of cross-repository REST integrations (FE/mobile → backend, backend → AI service), including which endpoint groups are fully documented here vs. `TODO: Verify`.
