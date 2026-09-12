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
REST/JSON over HTTP. OpenAPI/Swagger is already in use (`/swagger-ui.html`, `/v3/api-docs` on `caseflow-be`; `/swagger-ui.html`, `/api-docs` on `caseflow-ai-service`) — treat Swagger UI as the live, always-current reference and the documents below as the human-curated contract summary.

## Authoritative documents
- **caseflow-be ↔ caseflow-fe / caseflow-mobile**: [repos/backend/frontend-contract.md](../../repos/backend/frontend-contract.md) is the frozen, primary integration contract (auth flow, permission catalog, pagination, error shape, all endpoints, enums). [repos/backend/api-endpoints.md](../../repos/backend/api-endpoints.md) has full per-endpoint request/response examples; [repos/backend/api-notes.md](../../repos/backend/api-notes.md) is a lighter endpoint/error/enum overview.
- **caseflow-be → caseflow-ai-service**: [repos/ai-service/README.md](../../repos/ai-service/README.md) lists the 4 ticket-AI endpoints (`summary`, `reply-draft`, `similar-cases`, `policy-guidance`) plus ingest/health endpoints. This is a service-to-service contract only — `caseflow-fe`/`caseflow-mobile` must never call it directly ([ADR-0002](../../decisions/0002-ai-service-no-auth-p1.md)).

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
