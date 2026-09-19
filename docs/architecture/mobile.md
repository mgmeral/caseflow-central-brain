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
`caseflow-mobil` is a read-mostly mobile client for agents — Home/dashboard, Cases (list + read-only detail), Inbox (admin-pool queue), Customers, Notifications, Profile. It holds no domain data of its own and follows the **exact same backend auth contract** as `caseflow-fe` (no separate mobile-only auth flow).

## Components
Expo SDK ~57, React Native 0.86.3, React 19.2.3, TypeScript (strict), managed Expo workflow (no ejected native project, no `eas.json`). React Navigation (native-stack + bottom-tabs), TanStack Query (server state, incl. infinite-query pagination) + Zustand session store backed by `expo-secure-store`. Full detail: [repos/mobile/README.md](../../repos/mobile/README.md).

## Dependencies
`caseflow-be` REST API only (`EXPO_PUBLIC_API_BASE_URL`). Never calls `caseflow-ai-service` directly, mirroring the frontend boundary.

## Communication
`POST /api/auth/login`, `POST /api/auth/refresh`, `POST /api/auth/logout`, `GET /api/auth/me` — identical contract to `caseflow-fe`. Unlike the web frontend, mobile **does** implement automatic refresh-and-retry: a 401 triggers `refreshSession()` once and retries the original request before giving up, deduped via an in-flight promise. Does not implement OIDC/PKCE, since the backend does not expose that flow. Conversation/email-thread rendering adapts the same `GET /tickets/{id}/email/thread` contract used by the web FE, explicitly labeled in-app as "email-backed."

## Data ownership
None — pure client of `caseflow-be`, same as `caseflow-fe`.

## External integrations
None directly. `EXPO_PUBLIC_ENABLE_AI` / `EXPO_PUBLIC_ENABLE_PUSH` flags exist in `.env.example` but as of this pass **neither is read anywhere in application code** — both are currently inert, not feature switches for a working capability.

## Feature surface (verified 2026-09-12) — what's real vs. what looks real but isn't
**Fully wired to real backend endpoints:** login/refresh/logout with real 401-retry, dashboard stats, case list + case detail + SLA display, email-thread conversation view (read-only), customer list, admin-pool queue + stats (gated `ADMIN_POOL_VIEW`), in-app notifications (15s polling, mark-read), permission-based tab visibility.

**Looks implemented, is not:**
- **Push notifications** — `EXPO_PUBLIC_ENABLE_PUSH` exists but is never read by any code; no push library, permission request, or device-token registration exists. In-app notifications are polling-only.
- **Biometric unlock** — `expo-local-authentication` is used only to check device capability and drive a stored preference toggle; `authenticateAsync` (the actual biometric prompt) is never called anywhere. It does not gate login or app unlock. The Profile screen's own copy says "preference only."
- **AI features** — `EXPO_PUBLIC_ENABLE_AI` flag exists (default `true`) but is never consumed; there is no AI UI anywhere in the app.
- **Ticket workflow actions** — entirely read-only. No status change, assignment, or reply/compose UI, even though a `getCaseTransitions()` API call is already implemented in the API layer and simply never invoked from any screen.

## Security considerations
Session/token storage uses `expo-secure-store` (not plain AsyncStorage) — the entire persisted session slice (tokens + user + biometric preference) lives there. Same permission-code gating discipline as the web frontend applies (`ADMIN_POOL_VIEW` gates the Inbox tab, for example) — never gate on role name.

## Observability
Relies on backend-issued correlation IDs when reporting cross-system issues; no mobile-specific telemetry defined yet.

## Testing
Minimal — 3 test files (~4 test cases) total: permission-check logic, a query-string builder, and one login-screen render test. No coverage of navigation, session/refresh logic, or any list/detail screen.

## Open questions
- Whether push notifications, biometric-gated unlock, and read-write ticket workflow are still on the near-term roadmap, or the env flags/API hooks are leftover scaffolding from an earlier plan — see [docs/product/roadmap.md](../product/roadmap.md).
- No mobile-specific rate limiting, offline-queue, or background-sync strategy defined yet.

## Visual design gap (identified 2026-09-19)
`src/shared/theme/` is a single 9-value flat color palette with no typography/spacing/elevation scale, no icon library, and only four shared components (`Screen`, `SectionCard`, `MetricCard`, `CenteredState`) — no `Badge`/`Chip`, variant `Button`, `Avatar`, or illustrated empty state. Status/priority/SLA-risk render as plain text rather than color-coded indicators. This is a visual-layer gap distinct from the feature-completeness gaps tracked in `ALIGN-001`/`ALIGN-002` — tracked in [tasks/active/MOBILE-001-UI-UX-VISUAL-MODERNIZATION.md](../../tasks/active/MOBILE-001-UI-UX-VISUAL-MODERNIZATION.md).
