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
`caseflow-fe` is the web UI for agents/admins/viewers. It holds no domain data of its own — everything is fetched from `caseflow-be`.

## Components
React 18 + Vite + TypeScript, React Router v6, Zustand (auth/UI/filter state), TanStack Query v5 (server state), Tailwind CSS. Two runtime modes: real API (`VITE_USE_MOCKS=false`) and in-memory mock mode (`VITE_USE_MOCKS=true`, for UI development without a backend). Full detail: [repos/frontend/README.md](../../repos/frontend/README.md).

## Dependencies
`caseflow-be` REST API only. Never calls `caseflow-ai-service` directly.

## Communication
`VITE_API_URL` (default `/api`) is proxied to the backend in dev, or forwarded by a gateway in ngrok/production-like modes. JWT Bearer token stored in `localStorage` under `csm-auth` (Zustand `persist`). A 401 from any request clears auth state and redirects to `/login`.

## Data ownership
None — pure client of `caseflow-be`. Mock fixtures under `src/mock/` are dev-only and never a source of truth.

## External integrations
None directly — all external integrations (email, AI, storage) are mediated by `caseflow-be`.

## Security considerations
Must gate UI features on the `permissionCodes` array from `GET /auth/me`, **never** on `roleCode`/`roleName` (display-only) — see [ADR](../../decisions/README.md) discipline in [agents/CODE-REVIEW.md](../../agents/CODE-REVIEW.md) and [repos/backend/frontend-contract.md](../../repos/backend/frontend-contract.md).

## Observability
Relies on backend-issued `X-Correlation-Id` for cross-system trace correlation when reporting issues.

## Open questions
- Several backend endpoints are still V2-deferred from the frontend's perspective (auth deploy status per environment, template management, persisted role edits, server-side pagination/sort, outbound email replies) — see [repos/frontend/README.md](../../repos/frontend/README.md#api-contract) for the current list. Treat these as backend roadmap items, not frontend bugs.
