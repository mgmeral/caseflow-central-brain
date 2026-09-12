# Feature: SLA Management

## Purpose
Track first-response and resolution due dates against a configurable policy, detect breaches, and surface SLA state (on-track / at-risk / breached) to agents.

## Scope
SLA policy configuration (admin), automatic due-date stamping on ticket creation, scheduled breach detection, and SLA state display on tickets/dashboard/reports. **Does not currently include a working admin UI for policy CRUD** (see gap below) — policy configuration is backend-API-only today.

## Repositories Affected
- caseflow-be — `sla` module: `SlaPolicyController` (`/api/admin/sla/policies`, full CRUD + `/backfill`), `SlaPolicyConfig`/`SlaEventLog` entities, `SlaBreachCheckerJob` (`@Scheduled`, default 5 min), SLA fields directly on `Ticket` (`firstResponseDueAt`, `resolutionDueAt`, `firstResponseRespondedAt`, `resolvedAt`)
- caseflow-fe — `SLAIndicator.tsx` (presentational only — renders backend-computed state), dashboard/report SLA counts, ticket-list `slaBreachedOnly`/`slaAtRiskOnly` filters (server-side). `/admin/sla-policy` route/page exists but is a **static explainer with no CRUD** — see gap below.
- caseflow-ai-service — not affected
- caseflow-mobil — displays SLA state on case detail (read-only), same as web

## Gap (verified 2026-09-12)
The backend has full SLA policy CRUD; the frontend's `/admin/sla-policy` page is permission-gated and looks like a real settings screen, but its source is a static message telling admins to "contact your system administrator or update the backend configuration directly." **There is no way to configure SLA policy through the UI today.** This is pure frontend work to close — no backend dependency.

## Contract Impact
`/api/admin/sla/policies` full CRUD exists and is stable; per-field request/response shapes are `TODO: Verify` — not yet documented in [repos/backend/frontend-contract.md](../repos/backend/frontend-contract.md).

## Decision Dependencies
None recorded.

## Implementation Tasks
Build the real `/admin/sla-policy` CRUD UI in `caseflow-fe` — see [tasks/active/](../tasks/active/) to track once started.

## Rollout / Compatibility Notes
Breach detection is fully server-side and scheduled (not client-triggered) — no client behavior change needed as policies are added/edited once the UI exists.

## Done Criteria
`/admin/sla-policy` performs real CRUD against `/api/admin/sla/policies` instead of displaying a static explainer.
