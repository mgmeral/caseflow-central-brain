# Frontend Agent Instructions

## Purpose
Defines frontend-specific agent context and boundaries.

## What belongs here
Frontend repository usage rules, UI contract expectations, and cross-repository integration notes.

## Who should use this
Agents operating on `caseflow-fe`.

## What should NOT be stored here
Backend/mobile/AI-service-only implementation guidance.

## Current guidance

Read before implementing UI that touches the API:
1. [../repos/backend/frontend-contract.md](../repos/backend/frontend-contract.md) — the frozen, authoritative integration contract (auth flow, permission catalog, pagination, error shape, endpoints, enums). Treat this, not assumption or memory, as ground truth; verify against Swagger UI (`http://localhost:8080/swagger-ui.html`) when in doubt.
2. [../repos/frontend/README.md](../repos/frontend/README.md) — tech stack, mock vs. real mode, project structure, known V2-deferred limitations.

Rules:
- **Gate every feature on `permissionCodes`, never on `roleCode`/`roleName`.** `roleCode`/`roleName` are display-only.
- Never call `caseflow-ai-service` directly — AI features are backend-mediated only ([ADR-0002](../decisions/0002-ai-service-no-auth-p1.md)).
- Do not silently work around a missing/501 backend endpoint by faking richer behavior client-side (e.g. simulating outbound email reply success) — surface the real limitation, or switch to mock mode for local development, and treat it as a roadmap item (see [../docs/product/roadmap.md](../docs/product/roadmap.md)).
- If a change requires a backend contract change, propose the contract update in `caseflow-be`/here first (see [../agents/BACKEND.md](BACKEND.md)) rather than shaping the frontend around undocumented backend behavior.
- Outlook/IMAP mailbox admin screens: see [../repos/frontend/outlook-oauth2-imap-app-only.md](../repos/frontend/outlook-oauth2-imap-app-only.md) for the fields the backend expects (`oauthTenantId`, `oauthClientId`, `oauthClientSecret`, etc.) and which values must never be shown in list/detail views.
