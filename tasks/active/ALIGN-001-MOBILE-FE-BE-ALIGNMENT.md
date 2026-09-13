# ALIGN-001 — Mobile ↔ Frontend ↔ Backend Alignment

## Objective
Bring `caseflow-mobil` into alignment with `caseflow-fe` and `caseflow-be`'s current contracts and behavior, and fix the one point where `caseflow-fe` itself is behind `caseflow-mobil`'s more correct implementation (session refresh). This task plans the work only — no code is implemented here.

## Status
READY (per `workflows/TASK-LIFECYCLE.md`: `ALIGN-001-BE` is `DONE`; `ALIGN-001-FE`, `ALIGN-001-MOBILE-CORE`, and `ALIGN-001-MOBILE-EXT` are all `READY` and unblocked — the parent reflects the lowest not-yet-satisfied child state)

## Priority
P1 (contains one P0-severity finding — see `caseflow-fe` session refresh in Tasks below — bundled into an overall P1 alignment effort, not a product-blocking outage)

## User Request
"Use the existing `tasks/active/MOBILE-FE-ALIGNMENT.md` as the source of truth for the already identified alignment findings. Now instantiate a new framework-compliant cross-repository task from those findings. Goal: Align `caseflow-mobil` with the current `caseflow-fe` and `caseflow-be` behavior and contracts."

## Context
This task operationalizes the findings in [tasks/active/MOBILE-FE-ALIGNMENT.md](MOBILE-FE-ALIGNMENT.md), re-verified against current source on this pass (see "Re-verification" in Notes below — nothing was found to be outdated, and two previously-unconfirmed items are now confirmed). `caseflow-be` is the reference contract; `caseflow-fe` is the reference client implementation for every capability except session refresh, where `caseflow-mobil` is the more correct reference instead. See [docs/architecture/frontend.md](../../docs/architecture/frontend.md), [docs/architecture/mobile.md](../../docs/architecture/mobile.md), [repos/integration-map.md](../../repos/integration-map.md), [repos/backend/frontend-contract.md](../../repos/backend/frontend-contract.md).

## Affected Repositories

| Repository | Agent | Responsibility | Status |
|---|---|---|---|
| caseflow-be | Claude | Document exact field-level shapes for tags, Jira (ticket-level), and attachment-metadata endpoints (`TODO: Verify` items in `repos/backend/frontend-contract.md`) so mobile can build against them without guessing. **No backend code change is required** — these endpoints already exist and are already consumed correctly by `caseflow-fe`. | **DONE** (2026-09-13) |
| caseflow-fe | GitHub Copilot | Implement a real `/auth/refresh`-based session renewal flow (currently stores the refresh token but never uses it), using `caseflow-mobil`'s `refreshSession()` pattern as the reference implementation. | READY |
| caseflow-mobil | GitHub Copilot | Two independent tracks — see `ALIGN-001-MOBILE-CORE` (ready now) and `ALIGN-001-MOBILE-EXT` (was waiting on the BE documentation sub-task; unblocked now that `ALIGN-001-BE` is `DONE`) below. | READY / READY (split — see Task Graph) |
| caseflow-ai-service | — | **Not affected.** None of the re-verified findings involve AI-assist features in mobile; mobile does not implement any AI capability today, and adding one is out of scope for this alignment task (it would be its own feature-shaped task — see `features/ai-ticket-assist.md`). | N/A |

## Dependencies

### Depends On
None — this is the root task.

### Blocks
None known yet.

Internal ordering (see Task Graph): `ALIGN-001-MOBILE-EXT` depends on `ALIGN-001-BE`. Every other node is independent and may proceed in parallel — see Notes for why a rigid BE→FE→Mobile chain was deliberately **not** used here, per `workflows/TASK-DEPENDENCIES.md`.

## Contract Impact
- API: No new endpoints or shape changes — every capability below reuses an existing, stable `caseflow-be` endpoint already consumed by at least one client.
- Event: None.
- Database: None.
- Authentication: None (the FE refresh-token fix consumes the existing `POST /api/auth/refresh` contract; it does not change it).
- Net effect: this task is documentation + client-side implementation only. If any sub-task discovers the actual contract differs from what's documented, stop and route through `skills/contract-change/SKILL.md` rather than working around it silently.

## Tasks

### BE — DONE (2026-09-13)
- [x] Read `integration/jira/api/JiraController.java` + DTOs and documented the exact `/api/tickets/{ticketPublicId}/jira` request/response shape. **Confirmed:** create/retry are gated by `canSendCustomerReply` (`CUSTOMER_REPLY_SEND`) **or** `PERM_INTEGRATION_CONFIG_MANAGE` — the same permission `ALIGN-002-BE` confirmed for notification-channel admin, now also confirmed here for Jira rather than assumed.
- [x] Read `ticket/api/TicketTagController.java` + `Tag`/`TicketTag` DTOs and documented `/api/tags` and `/api/tickets/{id}/tags` exact shapes. **Finding:** two distinct permissions gate this one resource — tag *catalog* management (`/tags/all`, CRUD, activate/deactivate) requires `ADMIN_CONFIG`, while per-ticket tag add/remove requires `TICKET_TAG`. **Finding:** `TicketTagResponse` (a ticket's tag assignment) is a differently-shaped, flatter record than `TagResponse` (a catalog entry) — no nested object, and fields like `isActive`/`description` don't carry over.
- [x] Read the attachment-metadata shape returned on ticket/email detail responses (`AttachmentController`, `TicketEmailAttachmentController`) and documented the exact fields. **Correction to this checklist's own guess:** the actual field is `downloadPath` (one ready-to-use relative URL) + `previewSupported` (boolean) — there are no separate `previewUrl`/`downloadUrl` fields as originally guessed here. Two parallel serving paths exist depending on attachment origin (direct-upload vs. email-sourced); the backend already picks the right one per-attachment in `downloadPath`.
- [x] Updated `repos/integration-map.md` and `repos/backend/frontend-contract.md`'s `TODO: Verify` markers for these three endpoint groups — all now fully documented.
- **Bug found outside this checklist's scope, flagged not fixed:** while confirming the `PERM_` prefix convention (needed to verify `TICKET_TAG`/`ADMIN_CONFIG`), found that `SlaPolicyController` and `AutomationRuleController` both gate on `PERM_SETTINGS_MANAGE`, which does not exist in `identity/domain/Permission.java` — meaning those 11 endpoints are structurally unreachable by any role, not merely lacking FE UI. See `repos/backend/frontend-contract.md`'s Permission Catalog section for detail. Recommend a separate small bug-fix task in `caseflow-be`; not fixed here since it's outside `ALIGN-001-BE`'s tags/Jira/attachments scope and this sub-task is documentation-only regardless.

### FE
- [ ] Implement `refreshSession()`-equivalent logic in `caseflow-fe` (reference: `caseflow-mobil`'s `src/core/auth/session.ts` + `apiClient.ts` — in-flight-deduped, auto-retry-once-on-401).
- [ ] Wire it into `api.client.ts`'s 401 handling (currently dispatches `auth:unauthorized` → immediate logout) so a 401 attempts one silent refresh-and-retry before falling back to logout.
- [ ] Keep the existing logout-on-repeated-401 behavior as the fallback, not a replacement.

### Mobile — Core (unblocked, contract already fully documented and proven by FE)
- [ ] Wire the already-implemented `getCaseTransitions()` call (`src/cases/api/casesApi.ts:34`, confirmed still unused) into `CaseDetailScreen` to drive a status-change action.
- [ ] Add `assign`/`reassign` API functions (none exist today — `casesApi.ts` only has `getCases`, `getCaseDetail`, `getCaseTransitions`) and a UI action, gated on `TICKET_ASSIGN`.
- [ ] Add a `transfer` API function + UI action, gated on `TICKET_TRANSFER`.
- [ ] Add a notes-add API function + UI (currently notes are not addable anywhere in mobile), gated on `INTERNAL_NOTE_ADD`.
- [ ] Add a claim/assign action to `InboxScreen` (currently explicitly read-only — confirmed no `onAssign`-equivalent exists), mirroring `caseflow-fe`'s `AdminPoolPage.tsx` (`AssignmentModal` + bulk-assign), gated on `ADMIN_POOL_VIEW` (already used for tab visibility) + `TICKET_ASSIGN`.

### Mobile — Extended (unblocked — `ALIGN-001-BE` is DONE, exact shapes now in `repos/backend/frontend-contract.md`)
- [ ] Add tag display + add/remove on `CaseDetailScreen` (confirmed zero tag references anywhere in `caseflow-mobil/src` today, including in the `CaseDetail` type itself), gated on `TICKET_TAG` for add/remove. (Tag *catalog* management — creating new tags — is a separate `ADMIN_CONFIG`-gated capability, out of scope here; this checklist item is about applying existing tags to a ticket, matching `caseflow-fe`'s `TicketTagsCard`, not `TagManagementPage`.)
- [ ] Add Jira status/create/retry UI on `CaseDetailScreen` (confirmed zero Jira references anywhere in mobile today), gated the same way `caseflow-fe`'s `JiraIntegrationCard` is (`CUSTOMER_REPLY_SEND` for create/retry — the same permission that gates sending a reply, not a Jira-specific one, plus the `INTEGRATION_CONFIG_MANAGE` admin override). Render all three `JiraStatusResponse` states (`NOT_REQUESTED`, an in-flight/failed job, or a linked issue) — it's one combined shape, not three separate response types.
- [ ] Add attachment viewing (view/download) to the conversation thread view (confirmed zero attachment references anywhere in mobile today), mirroring `caseflow-fe`'s `AttachmentViewerModal` at a mobile-appropriate fidelity. Use the `downloadPath` field verbatim (don't reconstruct attachment URLs client-side — the backend already picks the correct one of two possible path shapes per attachment) and use `previewSupported` to decide inline-render vs. forced download.

### AI
Not applicable — see Affected Repositories above.

## Task Graph
```yaml
task_id: ALIGN-001
title: Mobile FE BE Alignment
status: READY

tasks:
  - id: ALIGN-001-BE
    repository: caseflow-be
    agent:
      provider: claude
    status: DONE
    depends_on: []

  - id: ALIGN-001-FE
    repository: caseflow-fe
    agent:
      provider: copilot
    status: READY
    depends_on: []

  - id: ALIGN-001-MOBILE-CORE
    repository: caseflow-mobil
    agent:
      provider: copilot
    status: READY
    depends_on: []

  - id: ALIGN-001-MOBILE-EXT
    repository: caseflow-mobil
    agent:
      provider: copilot
    status: READY
    depends_on:
      - ALIGN-001-BE

  - id: ALIGN-001-INTEGRATION
    repository: caseflow-central-brain
    agent:
      provider: codex
    status: BLOCKED
    depends_on:
      - ALIGN-001-BE
      - ALIGN-001-FE
      - ALIGN-001-MOBILE-CORE
      - ALIGN-001-MOBILE-EXT
```

## Acceptance Criteria
- [ ] `caseflow-fe` silently refreshes its session on a 401 (one retry) before falling back to logout; verified against a real/expired-token scenario.
- [ ] `caseflow-mobil` can change ticket status (restricted to the backend's allowed-transitions set), assign/reassign, transfer, and add notes from `CaseDetailScreen`.
- [ ] `caseflow-mobil`'s `InboxScreen` supports claiming/assigning a ticket directly from the pool, matching `caseflow-fe`'s `AdminPoolPage` capability.
- [ ] `caseflow-mobil` displays and can add/remove tags on a ticket.
- [ ] `caseflow-mobil` displays Jira status and can create/retry a Jira link on a ticket.
- [ ] `caseflow-mobil`'s conversation thread view supports viewing/downloading attachments.
- [ ] All new mobile/FE actions are gated on `permissionCodes`, never role name (per `agents/CODE-REVIEW.md`).
- [ ] `repos/backend/frontend-contract.md` and `repos/integration-map.md` no longer carry `TODO: Verify` for the tags/Jira/attachment endpoint groups.

## Validation
- [ ] Backend tests — N/A (no backend code change; `ALIGN-001-BE` is documentation-only)
- [ ] Frontend tests — new/updated coverage for the refresh-on-401 flow
- [ ] Mobile tests — new coverage for status/assign/transfer/notes/tags/Jira/attachments and the Inbox claim action
- [ ] Integration validation — manual verification against a real `caseflow-be` instance that: (a) FE's new refresh flow actually renews an expiring session; (b) every new mobile action produces the same server-side effect as its `caseflow-fe` equivalent; (c) `ALIGN-001-MOBILE-CORE` and `ALIGN-001-MOBILE-EXT` changes don't conflict when merged together

## Agent Instructions

### Claude
Read `docs/architecture/backend.md`, `repos/backend/module-map.md`, and `repos/integration-map.md` first. This is a **read-and-document** sub-task, not an implementation one — do not modify `caseflow-be` source. Output is an update to `repos/backend/frontend-contract.md` and `repos/integration-map.md` with the exact confirmed shapes for tags, Jira, and attachment-metadata endpoints. Do not modify `caseflow-fe`, `caseflow-mobil`, or `caseflow-ai-service`.

### Copilot (ALIGN-001-FE)
Read `docs/architecture/frontend.md`, `repos/frontend/README.md`, and `caseflow-mobil`'s `src/core/auth/session.ts` + `src/core/auth/sessionStore.ts` + `src/core/api/apiClient.ts` (as the reference pattern — read-only, do not modify `caseflow-mobil`). Implement the refresh-and-retry flow in `caseflow-fe`'s `src/store/auth.store.ts` and `src/services/api.client.ts`. Preserve the existing logout-on-failure fallback. Do not modify `caseflow-be`, `caseflow-mobil`, or `caseflow-ai-service`.

### Copilot (ALIGN-001-MOBILE-CORE)
Read `docs/architecture/mobile.md`, `repos/backend/ticket-rules.md`, and `repos/backend/frontend-contract.md`. Match `caseflow-fe`'s permission-gating pattern exactly (`permissionCodes`, never `roleCode`). This sub-task does not need to wait for `ALIGN-001-BE` — the endpoints it uses (`/tickets/{id}/status`, `/transitions`, `/assign`, `/transfer`, `/notes`, `/queue`) are already fully documented and already proven working via `caseflow-fe`. Do not modify `caseflow-be`, `caseflow-fe`, or `caseflow-ai-service`.

### Copilot (ALIGN-001-MOBILE-EXT)
Same repository/constraints as above. `ALIGN-001-BE` is now `DONE` — read the Tags/Jira/Attachments sections of `repos/backend/frontend-contract.md` before writing any request/response typing; they call out several non-obvious shapes (e.g. `TicketTagResponse` vs. `TagResponse` are not the same shape; `JiraStatusResponse` is one combined record covering three states; attachment URLs come from the `downloadPath` field, never reconstructed client-side). Coordinate with `ALIGN-001-MOBILE-CORE` and, if in flight concurrently, with `ALIGN-002`'s mobile sub-tasks — all touch `CaseDetailScreen`.

### Codex (ALIGN-001-INTEGRATION)
Do not modify any application repository. Once `ALIGN-001-BE`, `ALIGN-001-FE`, `ALIGN-001-MOBILE-CORE`, and `ALIGN-001-MOBILE-EXT` are all `DONE`, validate the combined change per the Validation section above, then update this task's `## Status` to `DONE` and move the file to `tasks/completed/`. Fold any new discoveries back into the relevant Central Brain docs in the same change.

## Completion Requirements
A task is not COMPLETE until all four implementation/documentation sub-tasks and integration validation have passed. Note that `ALIGN-001-FE` and `ALIGN-001-MOBILE-CORE` can reach `DONE` well before `ALIGN-001-MOBILE-EXT` (which waits on `ALIGN-001-BE`) — the parent task's status stays `IN_PROGRESS` until every node is `DONE`, per `workflows/TASK-LIFECYCLE.md`.

## Notes

### Re-verification (this pass) — findings confirmed, none discarded
Every finding in `tasks/active/MOBILE-FE-ALIGNMENT.md` that this task depends on was re-checked directly against current source (not assumed from the prior analysis):
- **FE never calls `/auth/refresh`** — confirmed: `src/store/auth.store.ts` stores `refreshToken` on login and only re-reads it to send with `/auth/logout`; no call to `/auth/refresh` exists anywhere in the file.
- **Mobile's `getCaseTransitions()` is defined but never called** — confirmed: the only match for that symbol in the entire `caseflow-mobil/src` tree is its own definition in `casesApi.ts`.
- **Mobile has no assign/transfer/notes-add capability** — confirmed, and slightly stronger than originally stated: `casesApi.ts` has no `assign`/`transfer`/`addNote` functions at all (not just missing UI — the API layer itself doesn't have them).
- **Mobile has zero tags/Jira/attachment references anywhere** — confirmed by direct search of `caseflow-mobil/src`; zero matches for all three. The `CaseDetail` type mapping (`mapCaseDetail` in `casesApi.ts`) doesn't carry tag or attachment fields through either, so this isn't only a missing-UI gap — the mobile data layer would need extending too.
- **Two previously-unconfirmed ("`TODO: Verify`") items are now CONFIRMED gaps**, upgraded from "possible" to certain:
  1. `caseflow-fe`'s `AdminPoolPage.tsx` does implement a real claim/assign action (`AssignmentModal`, single + bulk assign) — so mobile's read-only `InboxScreen` is now a confirmed gap, not a maybe.
  2. `caseflow-mobil` has zero attachment-related code anywhere (previously only inferred as "likely absent").
- **Backend contract stability spot-checked**: `AiAssistantController`, `JiraController`, `TicketTagController`, `AssignmentController`, `TransferController`, and `NoteController` all still exist at their expected paths in `caseflow-be` — no evidence the backend contract has moved since the original audit.
- **No findings were discarded as outdated.** Everything held up under direct re-inspection.

### Why the dependency graph isn't a simple BE→FE→Mobile chain
Per `workflows/TASK-DEPENDENCIES.md`, ordering was derived from what each sub-task actually needs, not applied as a template:
- `ALIGN-001-FE` needs nothing from this task's BE or Mobile sub-tasks — it's fixing a client-side bug against an already-stable, already-used endpoint.
- `ALIGN-001-MOBILE-CORE` needs nothing new from BE either — every endpoint it uses is already fully documented and already proven correct by `caseflow-fe`.
- Only `ALIGN-001-MOBILE-EXT` genuinely needs `ALIGN-001-BE`'s output first, because those three endpoint groups are the ones marked `TODO: Verify` in Central Brain today.
- This means three of the four implementation/documentation nodes can start immediately and in parallel — only one is actually blocked.

### ALIGN-001-BE completion notes (2026-09-13)
Confirmed exact shapes for all three endpoint groups against `caseflow-be` source — see the `### BE` checklist above for the finding-by-finding detail, now folded into `repos/backend/frontend-contract.md` and `repos/integration-map.md`. Two things worth calling out at the task level:
- The permission-catalog work needed to confirm `TICKET_TAG`/`ADMIN_CONFIG` surfaced a genuine backend bug unrelated to this task's own scope: `SlaPolicyController` and `AutomationRuleController` both check for a `PERM_SETTINGS_MANAGE` authority that the `Permission` enum can never actually grant (it's not one of the enum's ~30 constants) — those 11 endpoints are unreachable by any role today. Flagged in Central Brain docs, not fixed (out of scope for a read-only documentation sub-task); worth its own small bug-fix task in `caseflow-be` at some point.
- This task's original attachment-field checklist item guessed `previewUrl`/`downloadUrl` as the field names (carried over from the informal audit in `tasks/active/MOBILE-FE-ALIGNMENT.md`) — the actual backend field is a single `downloadPath` plus a `previewSupported` boolean. `ALIGN-001-MOBILE-EXT`'s checklist above has been corrected accordingly.

### Out of scope, explicitly
- Push notifications and biometric unlock in mobile — unrelated to FE/BE alignment; both remain tracked as their own known gaps in `docs/architecture/mobile.md`.
- SLA policy administration in mobile — `caseflow-fe` itself has no working implementation (non-functional stub), so mobile lacking it is the aligned outcome, not a gap. See `ALIGN-002`'s Context for the full rationale.

**Superseded by an explicit product decision:** the product decision this task's original note flagged as needed ("not included here without an explicit product decision to expand mobile's scope") has now been made — mobile's scope is expanding to full feature parity with `caseflow-fe`, including AI-assist, notification-channel admin, mail templates, scheduled email, dashboard/reports, and customer detail/contacts/reports. That work is tracked in [ALIGN-002-MOBILE-FULL-FEATURE-PARITY.md](ALIGN-002-MOBILE-FULL-FEATURE-PARITY.md) rather than folded into this task, to keep this task's already-`PLANNED` scope stable — see `ALIGN-002`'s Notes for why they're kept separate.
