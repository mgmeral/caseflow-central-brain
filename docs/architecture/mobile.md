# Mobile Architecture

## Purpose
This document defines mobile architecture context relevant to other repositories.

## What belongs here
Mobile responsibilities, platform boundaries, and shared integration assumptions.

## Who should use this
Mobile contributors and AI agents that coordinate mobile-facing features with backend/frontend/AI services.

## What should NOT be stored here
Platform-specific code snippets, build scripts, or implementation-only details — see [repos/mobile/](../../repos/mobile/).

## Responsibility
`caseflow-mobile` is the Phase 1 mobile client foundation for agents — Home, Cases, Inbox, Customers, Notifications, Profile. It holds no domain data of its own and follows the **exact same backend auth contract** as `caseflow-fe` (no separate mobile-only auth flow).

## Components
Expo-based React Native + TypeScript, React Navigation, TanStack Query, Zustand session store backed by `expo-secure-store`. Full detail: [repos/mobile/README.md](../../repos/mobile/README.md).

## Dependencies
`caseflow-be` REST API only (`EXPO_PUBLIC_API_BASE_URL`). Never calls `caseflow-ai-service` directly, mirroring the frontend boundary.

## Communication
`POST /api/auth/login`, `POST /api/auth/refresh`, `POST /api/auth/logout`, `GET /api/auth/me` — identical contract to `caseflow-fe`. Does not implement OIDC/PKCE, since the backend does not expose that flow. Conversation/email-thread rendering adapts the same `GET /tickets/{id}/email/thread` contract used by the web FE.

## Data ownership
None — pure client of `caseflow-be`, same as `caseflow-fe`.

## External integrations
None directly. `EXPO_PUBLIC_ENABLE_AI` / `EXPO_PUBLIC_ENABLE_PUSH` flags gate features that ultimately depend on backend/AI-service support, not on any mobile-side integration.

## Security considerations
Session/token storage uses `expo-secure-store` (not plain AsyncStorage). Same permission-code gating discipline as the web frontend applies (`ADMIN_POOL_VIEW` gates the Inbox tab, for example) — never gate on role name.

## Observability
Relies on backend-issued correlation IDs when reporting cross-system issues; no mobile-specific telemetry defined yet.

## Open questions
- Push notifications and reply-with-attachment flows are explicitly deferred, pending backend support.
- No mobile-specific rate limiting, offline-queue, or background-sync strategy defined yet.
