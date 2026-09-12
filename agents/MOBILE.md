# Mobile Agent Instructions

## Purpose
Defines mobile-specific agent context and boundaries.

## What belongs here
Mobile repository usage rules, integration expectations, and cross-repository constraints.

## Who should use this
Agents operating on `caseflow-mobil`.

## What should NOT be stored here
Backend/frontend/AI-service-only implementation guidance.

## Current guidance

Read before implementing mobile features that touch the API:
1. [../repos/backend/frontend-contract.md](../repos/backend/frontend-contract.md) — the same authoritative contract the web frontend uses; mobile follows it exactly, with no separate mobile-only auth flow.
2. [../repos/mobile/README.md](../repos/mobile/README.md) — current Phase 1 scope, repository structure, environment variables, auth model, known deferred items.

Rules:
- Do not implement OIDC/PKCE or any auth flow beyond `POST /api/auth/login` + `/refresh` + `/logout` + `GET /api/auth/me` — the backend does not expose anything else, and inventing a mobile-only flow would break the shared contract.
- Store session/token data in `expo-secure-store`, not plain `AsyncStorage`.
- Gate features on `permissionCodes` exactly as the web frontend does (see [../agents/FRONTEND.md](FRONTEND.md)) — e.g. the Inbox tab requires `ADMIN_POOL_VIEW`.
- Never call `caseflow-ai-service` directly.
- Push notifications and reply-with-attachment are explicitly deferred pending backend support — do not build ahead of a documented backend contract for them (check [../docs/product/roadmap.md](../docs/product/roadmap.md) first).
- **Verified 2026-09-12**: `EXPO_PUBLIC_ENABLE_PUSH` and `EXPO_PUBLIC_ENABLE_AI` env flags exist but are not read anywhere in application code, and the biometrics module only checks device capability without ever calling `authenticateAsync`. Do not treat the presence of these flags/modules as evidence the features are implemented — check [../docs/architecture/mobile.md](../docs/architecture/mobile.md) before building on top of any of them, and get an explicit product decision before wiring one up (build it for real, or remove the misleading scaffolding) rather than assuming it just needs a small addition.
- Ticket workflow in mobile is read-only today (no status change/assign/reply UI) even though `getCaseTransitions()` is already implemented in the API layer — do not assume the absence of UI means the API support is also missing.
