> # EXAMPLE ONLY
> # DO NOT EXECUTE
>
> This file demonstrates how to fill out `CROSS-REPOSITORY-TASK.md` for a
> real-shaped CaseFlow request, using architecture and gaps that are
> actually documented elsewhere in this repository. It is a template
> illustration, not an active task — it lives in `tasks/templates/`, not
> `tasks/active/`, on purpose. A related *real* analysis document already
> exists at `tasks/active/MOBILE-FE-ALIGNMENT.md`; this example does not
> replace or supersede it, and does not itself authorize any
> implementation work. If this work is ever actually scheduled, a new
> task should be created under `tasks/active/` using this shape, not by
> promoting this file.

# ALIGN-EXAMPLE-001 — Bring caseflow-mobil ticket workflow to parity with caseflow-fe

## Objective
Add read-write ticket workflow actions (status change, assignment, tagging) to `caseflow-mobil`, matching capabilities `caseflow-fe` already has against the same `caseflow-be` contract.

## Status
PLANNED

## Priority
P1

## User Request
"Mobile users keep asking why they can see tickets but can't update them — can we bring mobile up to parity with the web app for basic ticket actions?"

## Context
Per `tasks/active/MOBILE-FE-ALIGNMENT.md`, `caseflow-mobil` is read-only for ticket workflow: `GET /tickets/{id}/transitions` is already implemented in its API layer (`casesApi.ts`) but never called from any screen, and there is no status-change, assignment, or tag UI at all. `caseflow-fe` already implements all of this against the same `caseflow-be` contract (see `repos/backend/frontend-contract.md` and `repos/integration-map.md`). No backend contract change is required — this is a client-side implementation gap on the mobile side only.

## Affected Repositories

| Repository | Agent | Responsibility | Status |
|---|---|---|---|
| caseflow-be | Claude | None expected — contract already supports this; Claude reviews only if mobile's implementation surfaces an actual contract gap | READY |
| caseflow-fe | Copilot | None — reference implementation only, no changes needed | N/A |
| caseflow-mobil | Copilot | Build status-change, assignment, and tag UI against the existing contract | READY |

No `caseflow-ai-service` row — this feature has no AI-service involvement.

## Dependencies

### Depends On
None — the backend contract this depends on already exists and is stable (see `repos/integration-map.md`'s `/tickets/{id}/status`, `/tickets/{id}/transitions`, `/tickets/{id}/assign`, `/api/tags` rows).

### Blocks
None known yet.

## Contract Impact
- API: No — reuses existing `caseflow-be` endpoints already consumed by `caseflow-fe`.
- Event: None.
- Database: None.
- Authentication: None.
- None of the above requires a new ADR or contract-doc update; if mobile's implementation surfaces a real shape mismatch, escalate through `skills/contract-change/SKILL.md` before proceeding rather than working around it silently.

## Tasks

### BE
- [ ] (Reference only) Confirm `/tickets/{id}/transitions`, `/tickets/{id}/status`, `/tickets/{id}/assign`, `/api/tags` shapes are current against `repos/backend/frontend-contract.md` before mobile work starts, since that document is not yet fully verified for every field on these endpoints.

### FE
- [ ] (Reference only) None — `caseflow-fe`'s existing implementation (`AssignmentModal`, `TransferModal`, status-transition UI, `TicketTagsCard`) is the model to follow for UX/permission-gating patterns, not to be modified.

### Mobile
- [ ] Wire the existing `getCaseTransitions()` call into `CaseDetailScreen` to drive a status-change action.
- [ ] Add an assignment action (assign/reassign), gated by the same permission codes FE uses (`TICKET_ASSIGN`).
- [ ] Add tag display + add/remove on `CaseDetailScreen`, gated by `TICKET_TAG`.
- [ ] Follow `caseflow-fe`'s pattern of fetching allowed transitions before rendering status actions rather than hardcoding the transition matrix client-side (see `repos/backend/ticket-rules.md`).

## Task Graph
```yaml
task_id: ALIGN-EXAMPLE-001
title: Bring caseflow-mobil ticket workflow to parity with caseflow-fe
status: PLANNED

tasks:
  - id: ALIGN-EXAMPLE-001-MOBILE
    repository: caseflow-mobil
    agent:
      provider: copilot
    status: READY
    depends_on: []

  - id: ALIGN-EXAMPLE-001-INTEGRATION
    repository: caseflow-central-brain
    agent:
      provider: codex
    status: BLOCKED
    depends_on:
      - ALIGN-EXAMPLE-001-MOBILE
```
Only one implementation node: this is a mobile-only change against an already-stable contract, so there is no BE/FE node to depend on — see `workflows/TASK-DEPENDENCIES.md` for why a rigid BE→FE→Mobile chain would be the wrong assumption here.

## Acceptance Criteria
- [ ] Agent can change a ticket's status from `CaseDetailScreen`, restricted to the allowed-transitions set returned by the backend.
- [ ] Agent can assign/reassign a ticket to a user or group, gated by `TICKET_ASSIGN`.
- [ ] Agent can view and add/remove tags on a ticket, gated by `TICKET_TAG`.
- [ ] All new actions are gated on `permissionCodes`, never role name, per `agents/CODE-REVIEW.md`.

## Validation
- [ ] Backend tests — N/A (no backend change)
- [ ] Frontend tests — N/A (no frontend change)
- [ ] Mobile tests — new coverage for the added screens/actions
- [ ] Integration validation — manual verification against a real `caseflow-be` instance that mobile's status/assign/tag actions produce the same server-side effects as the equivalent `caseflow-fe` actions

## Agent Instructions

### Copilot
Read `docs/architecture/mobile.md`, `repos/backend/ticket-rules.md`, and `repos/backend/frontend-contract.md` before starting. Match `caseflow-fe`'s permission-gating pattern exactly (`permissionCodes`, never `roleCode`). Do not modify `caseflow-be`, `caseflow-fe`, or `caseflow-ai-service`.

## Completion Requirements
A task is not COMPLETE until mobile's implementation has been validated end-to-end against a real backend and the acceptance criteria above are all checked.

## Notes
This example intentionally reuses the real gap already documented in `tasks/active/MOBILE-FE-ALIGNMENT.md` (P1 items 2, 4, 5) to stay grounded in actual CaseFlow architecture rather than an invented scenario. **It remains an illustration of task-graph structure, not an authorization to begin this work.**
