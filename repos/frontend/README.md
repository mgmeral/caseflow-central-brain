# CSM CRM — Frontend (`caseflow-fe`)

React 18 + TypeScript + Vite frontend for the CSM CRM platform.

## Tech stack

| Tool | Purpose |
|------|---------|
| React 18 + Vite | UI + build |
| TypeScript 5 | Type safety |
| React Router v6 | Client-side routing |
| Zustand | Global state (auth, UI, filters) |
| TanStack Query v5 | Server state / data fetching |
| Tailwind CSS | Styling |
| Vitest | Unit tests |

## Quick start

```bash
npm install
cp .env.example .env.local   # set VITE_API_URL and VITE_USE_MOCKS
npm run dev
```

## Environment variables

| Variable | Default | Description |
|----------|---------|--------------|
| `VITE_API_URL` | `/api` | Runtime API base path used by the browser. Keep relative in local dev and gateway/ngrok mode. |
| `VITE_USE_MOCKS` | `false` | Set to `true` to run without a backend using in-memory mock data |

## Running modes

**Real API mode** (default) — all services call relative `/api/*`; the dev server proxies to `http://localhost:8080`.
Auth note: login calls `POST /auth/login → { token, user }`. If not yet deployed on the backend, switch to mock mode.

**Mock mode** (`VITE_USE_MOCKS=true`) — all services use in-memory fixtures from `src/mock/`. Any active mock user works with any password (e.g. `ali.yilmaz@csm.com`, admin).
Mock-only features (always empty/501 in real mode): template management (`/admin/templates`), outbound email sending.

## Project structure

```text
src/
  services/   # API + mock service layer (one file per domain)
  hooks/      # React Query hooks wrapping services
  store/      # Zustand stores (auth, ui, filters)
  pages/      # Route-level page components
  components/ # Reusable UI components
  lib/        # Shared utilities (env, errors)
  mock/       # Mock data fixtures
  router/     # Router config + ProtectedRoute
  types/      # TypeScript type definitions
  test/       # Vitest unit tests
```

## Scripts

| Command | Description |
|---------|-------------|
| `npm run dev` | Start dev server |
| `npm run build` | Type-check + production build |
| `npm run typecheck` | Type-check only |
| `npm test` | Run all tests once |
| `npm run test:watch` | Watch mode |
| `npm run test:coverage` | Coverage report |

## Docker

```bash
docker build --build-arg VITE_API_URL=/api --build-arg VITE_USE_MOCKS=false -t csm-crm-fe .
docker run --rm -p 8081:80 csm-crm-fe
```

`VITE_API_URL`/`VITE_USE_MOCKS` are embedded at image build time (Vite). Served by Nginx on port 80 with SPA fallback to `index.html`.

## API contract

Full endpoint list, auth model, role-based access are the backend's responsibility to define — see [../backend/frontend-contract.md](../backend/frontend-contract.md) and [../integration-map.md](../integration-map.md) for the authoritative, backend-verified contract (the latter also covers newer endpoint groups — Jira, SLA, tags, automation, dashboard/reports — not yet in `frontend-contract.md`'s per-field detail). Auth token (both access and refresh) is stored under `localStorage` key `csm-auth` via Zustand `persist`; a 401 from any query/mutation clears auth state and redirects to `/login`.

**Verified 2026-09-12 gaps:**
- **No refresh-token flow is implemented** — the refresh token is stored but only ever re-read to send with the logout call. Any 401 triggers immediate logout rather than a silent refresh-and-retry (contrast with `caseflow-mobil`, which does implement this).
- **`VITE_USE_MOCKS` no longer does what its own `.env.example` comment describes** — it only toggles the Vite dev-server's `/api` proxy; there is no client-side mock-data-serving code path in `src/` (the `src/mock/` fixtures are used only by unit tests).
- **`/admin/sla-policy` is a static explainer page**, not a working SLA policy CRUD UI, despite being a real permission-gated route — the backend has full SLA policy CRUD that this page does not call.
- **AI "similar cases" and "policy guidance"** are commented in `ai.service.ts` as "Phase 2 — not implemented until BE is ready," but both are already fully implemented on `caseflow-be`/`caseflow-ai-service` — this is a frontend-side gap, not a backend-readiness gap.
- Real feature areas confirmed wired to actual backend endpoints (not mocked): tickets, customers, email (thread/reply/templates/scheduled-send), notifications, dashboard/reports (incl. client-side PDF export), user/role/group admin, Jira integration, notification-channel (Slack/Teams/webhook) admin, tags, ingress-event ops.

Route protection: `admin` → all routes; `supervisor` → all except `/admin/*`; `trade_agent`/`operation_agent` → tickets, customers, reports; `viewer` → read-only. Enforced in `src/router/index.tsx` via `ProtectedRoute`. **Gate features on backend `permissionCodes`, never on role name — see [../backend/frontend-contract.md](../backend/frontend-contract.md).**

## Outlook / IMAP mailbox setup (Entra app-only)

See [outlook-oauth2-imap-app-only.md](outlook-oauth2-imap-app-only.md) for the full Microsoft 365 / Exchange Online app registration + mailbox authorization runbook used when configuring an Outlook mailbox for backend IMAP polling.
