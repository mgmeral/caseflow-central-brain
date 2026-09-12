# Frontend Architecture

## Purpose
This document defines frontend architecture context relevant to other repositories.

## What belongs here
Frontend responsibilities, integration expectations, and constraints that affect shared behavior.

## Who should use this
Frontend contributors and AI agents implementing UX flows tied to backend, AI, or mobile systems.

## What should NOT be stored here
Component-level code details, styling decisions, or internal task notes — see [repos/frontend/](../../repos/frontend/).

## Responsibility
`caseflow-fe` is the web UI for agents/admins/supervisors/viewers. It holds no domain data of its own — everything is fetched from `caseflow-be`.

## Components
React 18 + Vite 6 + TypeScript, React Router v6, Zustand (auth/UI/filter state), TanStack Query v5 (server state), Tailwind CSS, no UI kit, hand-written `fetch` API layer (no axios, no generated client). Full detail: [repos/frontend/README.md](../../repos/frontend/README.md).

## Dependencies
`caseflow-be` REST API only. Never calls `caseflow-ai-service` directly (enforced by convention/code comment, not a network boundary the frontend can see).

## Communication
`VITE_API_URL` (default `/api`), kept relative so the same build works across the Vite dev proxy, a dev gateway, and ngrok tunnels without rebuilding. JWT Bearer token (both access and refresh tokens) stored in `localStorage` under `csm-auth` (Zustand `persist`). **The frontend never uses its refresh token to renew a session** — it is stored and only ever re-read to send with the logout call. Any 401 (outside `/auth/login`) triggers an immediate client-side logout + redirect to `/login`, via two independent code paths (an `api.client.ts` custom event listened to by the auth store, and a separate React Query `onError` handler) — functionally redundant, worth simplifying if touched. This is a real gap relative to `caseflow-mobil`, which does implement silent refresh-and-retry on 401.

## Data ownership
None — pure client of `caseflow-be`. `src/mock/users.mock.ts` fixtures exist but are used only by unit tests, not by the running app; the documented `VITE_USE_MOCKS` flag only toggles the Vite dev-proxy, not any client-side mock-serving code path.

## External integrations
None directly — all external integrations (email, AI, Jira, Slack/Teams/webhooks, storage) are mediated by `caseflow-be`.

## Feature surface (verified 2026-09-12) — what's real vs. what looks real but isn't
**Fully wired to real backend endpoints:** ticket CRUD/status/assign/transfer/close-reopen/tags/notes; customers + contacts; email (thread, reply, preview, scheduled send, mailbox admin, customer email settings, templates with live preview); notifications (polling); dashboard stats; reports (per-customer + admin aggregate, client-side PDF export); user/role/group admin; Jira integration (status/create/retry/config/test); notification-channel (Slack/Teams/webhook) admin; ingress-event admin ops; AI summary; AI reply draft.

**Looks implemented, is not:**
- **`/admin/sla-policy`** — a fully permission-gated, real-looking settings page whose source is a static explainer telling admins "SLA policy management through this interface is not available" — despite the backend having full SLA policy CRUD (`/api/admin/sla/policies`). Do not assume SLA policy management exists in the frontend from the routing table alone.
- **AI "similar cases" and "policy guidance"** — both are fully implemented end-to-end on `caseflow-be`/`caseflow-ai-service`, but the frontend's `ai.service.ts` explicitly comments them as "Phase 2 — not implemented until BE is ready" and never calls either endpoint. The gap here is a frontend-consumption gap, not a backend-readiness gap — check this doc and [docs/architecture/ai-service.md](ai-service.md) before assuming otherwise.
- **`ticketService.addPublicReply`** — always throws a 501 stub; superseded by the real email-reply flow (`ticketEmail.service.ts`).

## Security considerations
Must gate UI features on the `permissionCodes` array from `GET /auth/me`, **never** on `roleCode`/`roleName` (display-only) — see [agents/CODE-REVIEW.md](../../agents/CODE-REVIEW.md) and [repos/backend/frontend-contract.md](../../repos/backend/frontend-contract.md). Route-level gating in `router/index.tsx` plus a `ProtectedRoute` permission check (belt-and-suspenders); some pages also show an in-page "Access Denied" state.

## Observability
Relies on backend-issued `X-Correlation-Id` for cross-system trace correlation when reporting issues.

## Testing
Vitest 4 + Testing Library + jsdom, ~44 test files (services, hooks, components/pages). No E2E framework (no Cypress/Playwright).

## Open questions
- Whether the frontend package's continued `crm-fe` `package.json` name (vs. the CaseFlow product name) is intentional or leftover — `TODO: Verify`.
- Whether/when to build a real refresh-token renewal flow, and whether/when to build the deferred SLA-policy CRUD and AI Phase-2 (similar-cases/policy-guidance) UI — see [docs/product/roadmap.md](../product/roadmap.md).
