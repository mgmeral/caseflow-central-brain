# ALIGN-001 — Mobile ↔ Frontend ↔ Backend Alignment

## Objective
Bring `caseflow-mobil` into alignment with `caseflow-fe` and `caseflow-be`'s current contracts and behavior, and fix the one point where `caseflow-fe` itself is behind `caseflow-mobil`'s more correct implementation (session refresh). This task plans the work only — no code is implemented here.

## Status
IN_PROGRESS (per `workflows/TASK-LIFECYCLE.md`: `ALIGN-001-BE`, `ALIGN-001-MOBILE-CORE`, and `ALIGN-001-MOBILE-EXT` are all `DONE` as of 2026-09-19; `ALIGN-001-FE` — the `caseflow-fe` session-refresh fix — is still `READY`/not started, and `ALIGN-001-INTEGRATION` is `BLOCKED` on it. The parent reflects the lowest not-yet-satisfied child state.)

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
| caseflow-mobil | Claude (implemented directly this session, not handed to Copilot) | Two independent tracks — see `ALIGN-001-MOBILE-CORE` and `ALIGN-001-MOBILE-EXT` below. | **DONE** (2026-09-19) |
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
- **Bug found outside this checklist's scope, tracked and fixed separately:** while confirming the `PERM_` prefix convention (needed to verify `TICKET_TAG`/`ADMIN_CONFIG`), found that `SlaPolicyController` and `AutomationRuleController` both gated on `PERM_SETTINGS_MANAGE`, which did not exist in `identity/domain/Permission.java` — meaning those 11 endpoints were structurally unreachable by any role, not merely lacking FE UI. Tracked and fixed in `tasks/completed/BUG-001-UNREACHABLE-SETTINGS-PERMISSIONS.md` (repointed to the existing `ADMIN_CONFIG` permission) rather than here, since it was outside `ALIGN-001-BE`'s tags/Jira/attachments scope.

### FE
- [ ] Implement `refreshSession()`-equivalent logic in `caseflow-fe` (reference: `caseflow-mobil`'s `src/core/auth/session.ts` + `apiClient.ts` — in-flight-deduped, auto-retry-once-on-401).
- [ ] Wire it into `api.client.ts`'s 401 handling (currently dispatches `auth:unauthorized` → immediate logout) so a 401 attempts one silent refresh-and-retry before falling back to logout.
- [ ] Keep the existing logout-on-repeated-401 behavior as the fallback, not a replacement.

### Mobile — Core — DONE (2026-09-19, implemented directly by Claude in this session rather than handed to Copilot — see Notes)
- [x] Wire `getCaseTransitions()` into `CaseDetailScreen` to drive a status-change action. Done via a `SelectSheet` listing `allowedTransitions`, calling the now-added `POST /tickets/{id}/status`.
- [x] Add `assign`/`unassign` API functions + UI action, gated on `TICKET_ASSIGN`. **Scoped down from the original "assign/reassign" wording**: implemented as a self-service "Assign to me" / "Unassign" action (no arbitrary-user picker) — a full reassign-to-any-agent picker needs `PERM_USER_READ` (per `caseflow-fe`'s `AssignmentModal`, which lists all users) which most agent roles don't hold, and was judged out of proportion to the actual complaint (agents needing to claim/release their own tickets). Flagged as a known gap, not silently dropped — see Notes.
- [x] Add a `transfer` API function + UI action, gated on `TICKET_TRANSFER`. Full parity with FE here — group picker via `GET /groups` (no special permission required) + optional reason, matching `caseflow-fe`'s `TransferModal`.
- [x] Add a notes-add API function + UI, gated on `INTERNAL_NOTE_ADD`. Also added notes *viewing* (`GET /notes/by-ticket/{id}`), which wasn't in the original checklist wording but is the same gap (mobile had zero notes UI, read or write).
- [x] Add a claim/assign action to `InboxScreen`, gated on `ADMIN_POOL_VIEW` + `TICKET_ASSIGN`. Implemented as a "Claim" button per queue row (self-assign) rather than `caseflow-fe`'s full `AssignmentModal` + bulk-assign — matches the "claim from the pool" use case without the arbitrary-user picker (same scope note as assign above).
- Also fixed while implementing this: `CaseDetail`'s mobile type/mapping was stale against the actual backend `TicketDetailResponse` (missing `customerId`/`assignedUserId`/`assignedGroupId`/`attachments`/`history`, and `history`'s shape was a guessed inline type rather than the real `HistorySummaryResponse`) — corrected in `src/types/api.ts`, which is also what made attachments/history rendering below possible without further backend changes.
- Also added, beyond this checklist: an "Assigned to Me" filter tab on `CasesScreen` (`GET /tickets?userId=`) and a "History" section on `CaseDetailScreen` rendering the ticket's `history` array (was already fetched, never rendered) — both were direct, explicit complaints from the user this pass, not part of the original ALIGN-001 audit.

### Mobile — Extended — DONE (2026-09-19, same session as Core)
- [x] Add tag display + add/remove on `CaseDetailScreen`, gated on `TICKET_TAG`. Matches the documented `TicketTagResponse` shape exactly (flat, no nested object).
- [x] Add Jira status/create/retry UI on `CaseDetailScreen`, gated on `CUSTOMER_REPLY_SEND` **or** `INTEGRATION_CONFIG_MANAGE`. Renders all `JiraStatusResponse` states from the one combined shape (not-requested / in-flight / failed-with-retry / linked-with-external-link).
- [x] Add attachment viewing to the ticket (list + download), using `downloadPath`/`previewSupported` verbatim, never reconstructed. **Implementation note:** the download endpoint requires a Bearer auth header (no query-token alternative), which `Linking.openURL` can't send — so this needed two new native dependencies (`expo-file-system`, `expo-sharing`, both installed via `expo install` for SDK-57-compatible versions) to download with an auth header to local cache, then hand off to the OS share/open sheet. Scoped to the ticket-level attachment list (`TicketDetailResponse.attachments`), not per-email inline attachments inside the conversation thread — the latter needs the separate unified email-detail endpoint and was judged lower-value for this pass.

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
      provider: claude
    status: DONE
    depends_on: []

  - id: ALIGN-001-MOBILE-EXT
    repository: caseflow-mobil
    agent:
      provider: claude
    status: DONE
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
- [ ] `caseflow-fe` silently refreshes its session on a 401 (one retry) before falling back to logout; verified against a real/expired-token scenario. **Not done** — `ALIGN-001-FE` untouched.
- [x] `caseflow-mobil` can change ticket status (restricted to the backend's allowed-transitions set), assign, transfer, and add notes from `CaseDetailScreen`. **Partially exceeds spec, partially narrower:** "assign" is self-assign/unassign only, not arbitrary-user reassign — see `ALIGN-001-MOBILE-CORE` notes for why.
- [x] `caseflow-mobil`'s `InboxScreen` supports claiming a ticket directly from the pool. **Narrower than `AdminPoolPage`:** single-ticket claim only, no bulk-assign or reassign-to-other-agent — same scope note as above.
- [x] `caseflow-mobil` displays and can add/remove tags on a ticket.
- [x] `caseflow-mobil` displays Jira status and can create/retry a Jira link on a ticket.
- [x] `caseflow-mobil`'s ticket detail supports viewing/downloading attachments. Scoped to the ticket-level attachment list, not per-email inline attachments inside the conversation thread — see `ALIGN-001-MOBILE-EXT` notes.
- [x] All new mobile actions are gated on `permissionCodes`, never role name — `TICKET_STATUS_CHANGE`, `TICKET_ASSIGN`, `TICKET_TRANSFER`, `INTERNAL_NOTE_ADD`, `TICKET_TAG`, `CUSTOMER_REPLY_SEND`/`INTEGRATION_CONFIG_MANAGE` (Jira), all read via `hasPermission(user.permissionCodes, ...)`. FE side still `[ ]` — untouched.
- [ ] `repos/backend/frontend-contract.md` and `repos/integration-map.md` no longer carry `TODO: Verify` for the tags/Jira/attachment endpoint groups. **Already satisfied by `ALIGN-001-BE`** (done 2026-09-13) — unrelated to this session's mobile work.

## Validation
- [ ] Backend tests — N/A (no backend code change; `ALIGN-001-BE` is documentation-only)
- [ ] Frontend tests — new/updated coverage for the refresh-on-401 flow. **Not done** — `ALIGN-001-FE` untouched.
- [x] Mobile tests — `npm run typecheck` and `npm test` pass clean after every increment of this work (status/assign/transfer/notes/tags/Jira/attachments/Inbox-claim), and the web (`expo start --web`) bundle was confirmed to build after each. **No new automated test *cases*** were added for the new screens/mutations themselves — verification leaned on typecheck + manual review of the diff, not new Jest coverage. Flagged as a gap, not silently skipped.
- [~] Integration validation — the mobile side was validated live against the real `caseflow-be` Docker stack running locally (not just typechecked): Metro was run in tunnel mode and connected to a physical device over the session, backend health-checked as `UP`. **Not done**: (a) FE's refresh flow (FE untouched); (c) an explicit cross-check that `ALIGN-001-MOBILE-CORE` and `-EXT` don't conflict — moot here since both were implemented together, in the same files, by the same session, rather than merged from independent branches.

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

### Implementation pass (2026-09-19) — Mobile-Core and Mobile-Ext both DONE
Implemented directly by Claude in this session, working in `caseflow-mobil` on direct user request (prompted by "mobil hiç işlevsel değil" — the mobile app feels completely non-functional), rather than handed off to GitHub Copilot per the default in `agents/AGENT-OWNERSHIP.md` — an explicit session-level override, not a change to default ownership. This followed directly on from `MOBILE-001` (visual modernization, same session) — the new screens below were built using `MOBILE-001`'s new `Badge`/`Button`/`SectionCard`/etc. components rather than the old plain-text styling, avoiding the double-work risk that task's Notes had flagged.

What landed, beyond the per-checklist-item detail already in the Tasks section above:
- New domain folders: `src/workflow/` (assign/transfer), `src/notes/`, `src/tags/`, `src/jira/`, `src/groups/` — each with an `api/` + `hooks/` pair, following the existing per-domain folder convention (`cases/`, `customers/`, etc.).
- New shared UI primitives needed for this work (not present before): `SelectSheet` (generic bottom-sheet single-select, used for status-change and tag-add) and a dedicated `TransferSheet` (group picker + reason).
- `types/api.ts` correctness fix: several response types (`TicketDetailResponse`/`CaseDetail`, `AllowedTransitionsResponse`) were stale against the actual backend DTOs (missing fields, or guessed shapes) — corrected against `caseflow-be` source directly, not against this repo's docs alone, per `skills/analysis/SKILL.md`'s "confirm load-bearing claims against actual source" rule.
- Verified live: the local `caseflow-be` Docker stack was health-checked as `UP`, and Metro was run in tunnel mode connected to a physical device for the session, so this wasn't a typecheck-only pass — though see the Validation section for what wasn't independently re-verified (no new Jest coverage, no formal merge-conflict check since Core/Ext were done together).

### Deliberate scope reduction: self-assign only, not arbitrary reassign
`caseflow-fe`'s `AssignmentModal` lets an admin/supervisor pick *any* agent from a searchable, group-filterable list (backed by `GET /users`, gated `PERM_USER_READ`/`PERM_USER_MANAGE` — permissions most regular agent roles don't hold per the Starter Role Defaults table in `repos/backend/frontend-contract.md`). Building that full picker in mobile would have meant either gating the whole assign feature behind a permission most users don't have, or building a second, narrower user-listing endpoint that doesn't exist today. Instead, mobile implements "Assign to me" / "Unassign" (self-service claim/release, using the ticket's own `assignedUserId`) — this covers the actual complaint (agents wanting to see and claim their own work) without a new backend surface. Reassign-to-another-agent is a real, acknowledged gap versus full FE parity — worth a follow-up task if the product need shows up, not silently dropped here.

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
- The permission-catalog work needed to confirm `TICKET_TAG`/`ADMIN_CONFIG` surfaced a genuine backend bug unrelated to this task's own scope: `SlaPolicyController` and `AutomationRuleController` both checked for a `PERM_SETTINGS_MANAGE` authority that the `Permission` enum could never actually grant (it wasn't one of the enum's 30 constants) — those 11 endpoints were unreachable by any role. Flagged in Central Brain docs first (out of scope for this read-only documentation sub-task), then fixed as its own task — see `tasks/completed/BUG-001-UNREACHABLE-SETTINGS-PERMISSIONS.md`.
- This task's original attachment-field checklist item guessed `previewUrl`/`downloadUrl` as the field names (carried over from the informal audit in `tasks/active/MOBILE-FE-ALIGNMENT.md`) — the actual backend field is a single `downloadPath` plus a `previewSupported` boolean. `ALIGN-001-MOBILE-EXT`'s checklist above has been corrected accordingly.

### Out of scope, explicitly
- Push notifications and biometric unlock in mobile — unrelated to FE/BE alignment; both remain tracked as their own known gaps in `docs/architecture/mobile.md`.
- SLA policy administration in mobile — `caseflow-fe` itself has no working implementation (non-functional stub), so mobile lacking it is the aligned outcome, not a gap. See `ALIGN-002`'s Context for the full rationale.

**Superseded by an explicit product decision:** the product decision this task's original note flagged as needed ("not included here without an explicit product decision to expand mobile's scope") has now been made — mobile's scope is expanding to full feature parity with `caseflow-fe`, including AI-assist, notification-channel admin, mail templates, scheduled email, dashboard/reports, and customer detail/contacts/reports. That work is tracked in [ALIGN-002-MOBILE-FULL-FEATURE-PARITY.md](ALIGN-002-MOBILE-FULL-FEATURE-PARITY.md) rather than folded into this task, to keep this task's already-`PLANNED` scope stable — see `ALIGN-002`'s Notes for why they're kept separate.
