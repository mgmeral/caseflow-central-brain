# ALIGN-002 — Mobile Full Feature Parity with Frontend

## Objective
Bring `caseflow-mobil` to feature and UX parity with `caseflow-fe` on every capability `ALIGN-001` deliberately left out of scope, so that mobile stops being a "read-mostly" client and instead covers substantially the same feature surface as the web frontend. This task plans the work only — no code is implemented here.

## Status
READY (per `workflows/TASK-LIFECYCLE.md`: `ALIGN-002-BE` is `DONE`; `ALIGN-002-MOBILE-CORE` and `ALIGN-002-MOBILE-EXT` are both `READY` and unblocked — the parent reflects the lowest not-yet-satisfied child state)

## Priority
P1 (product-scope expansion, not a defect — no capability here is broken today, mobile simply doesn't have it yet)

## User Request
"ALIGN-001-MOBILE-FE-BE-ALIGNMENT'ı oluşturmuştuk ama benim şu an asıl istediğim şey mobile FE ile aynı featurelara sahip olsun aynı kullanıcı deneyimi versin (En kötü %80 benzerlik ile)" — i.e., mobile should have the same features as the web frontend and give the same user experience, with a minimum ~80% feature-parity bar. Follow-up clarification: **full parity including back-office admin screens** (notification-channel admin, mail-template admin, scheduled-email management) is explicitly in scope — not just day-to-day case-handling actions.

## Context
`tasks/active/MOBILE-FE-ALIGNMENT.md` (the original audit) and `tasks/active/ALIGN-001-MOBILE-FE-BE-ALIGNMENT.md` (the first remediation task) both explicitly deferred a set of capabilities as "out of scope, explicitly," pending a product decision on whether mobile's documented "read-mostly Phase 1" boundary ([docs/architecture/mobile.md](../../docs/architecture/mobile.md) line 16) should expand. That decision has now been made: expand it. This task operationalizes every capability `ALIGN-001` deferred, using the same audit as its source of truth, re-scoped against current source where noted below.

`caseflow-be` is the reference contract; `caseflow-fe` is the reference client implementation for everything in this task — there is no mobile-is-more-correct exception here (unlike `ALIGN-001`'s session-refresh finding).

**One deliberate exclusion:** SLA policy administration (`/api/admin/sla/policies` CRUD) is **not** included. `caseflow-fe` itself has no working implementation — its `/admin/sla-policy` route is a non-functional static explainer, not real CRUD (see [features/sla-management.md](../../features/sla-management.md)). Building real SLA admin in mobile would make mobile *exceed* the frontend's actual capability, which is not "parity with FE" — it would be new backend-facing product work outside this task's purpose. Mobile having no SLA admin, matching FE's non-functional stub, is already the aligned outcome.

See [docs/architecture/frontend.md](../../docs/architecture/frontend.md), [docs/architecture/mobile.md](../../docs/architecture/mobile.md), [repos/integration-map.md](../../repos/integration-map.md), [repos/backend/frontend-contract.md](../../repos/backend/frontend-contract.md).

## Affected Repositories

| Repository | Agent | Responsibility | Status |
|---|---|---|---|
| caseflow-be | Claude | Document exact field-level shapes for notification-channel admin, mail-template admin, scheduled-email, and customer/admin reports endpoints (all currently `TODO: Verify` per `repos/integration-map.md` line 6) so mobile can build against them without reverse-engineering `caseflow-fe`'s TypeScript types. **No backend code change is required** — every endpoint already exists and is already consumed correctly by `caseflow-fe`. | **DONE** (2026-09-13) |
| caseflow-mobil | Copilot | Two independent tracks — see `ALIGN-002-MOBILE-CORE` (ready now, contracts already stable/documented) and `ALIGN-002-MOBILE-EXT` (was waiting on the BE documentation sub-task; unblocked now that `ALIGN-002-BE` is `DONE`) below. | READY / READY (split — see Task Graph) |
| caseflow-ai-service | — | **Not affected.** Mobile's new AI-assist UI calls `caseflow-be`'s existing AI-assist endpoints only, mirroring `caseflow-fe`'s and `caseflow-mobil`'s existing architectural boundary (neither client talks to `caseflow-ai-service` directly). | N/A |

`caseflow-fe` has no row — every capability in this task is a mobile-only gap; the reference implementation already exists in `caseflow-fe` and needs no change.

## Dependencies

### Depends On
None — independent of `ALIGN-001`. The two tasks cover disjoint capability sets (this task explicitly starts where `ALIGN-001`'s "out of scope" list ends) and neither blocks the other, though both touch `caseflow-mobil`'s `CaseDetailScreen` — see Notes for merge-coordination guidance.

### Blocks
None known yet.

Internal ordering (see Task Graph): `ALIGN-002-MOBILE-EXT` depends on `ALIGN-002-BE`. `ALIGN-002-MOBILE-CORE` and `ALIGN-002-BE` may proceed in parallel — per `workflows/TASK-DEPENDENCIES.md`, don't serialize work that doesn't actually need to wait.

## Contract Impact
- API: No new endpoints or shape changes — every capability below reuses an existing, stable `caseflow-be` endpoint already consumed by `caseflow-fe`.
- Event: None.
- Database: None.
- Authentication: None.
- Net effect: this task is documentation + client-side implementation only. If any sub-task discovers the actual contract differs from what's documented, stop and route through `skills/contract-change/SKILL.md` rather than working around it silently.

## Tasks

### BE — DONE (2026-09-13)
- [x] Read `NotificationChannelConfigController` + DTOs and documented the exact `/api/admin/integrations/channels/*` and `/event-catalog` shapes, plus the confirmed `PERM_INTEGRATION_CONFIG_MANAGE` permission code string. **Finding:** `channelType` is only `SLACK`/`TEAMS` — there is no separate generic `WEBHOOK` channel type despite the common shorthand. **Finding:** `subscribedEvents` in the response is a JSON-encoded string, not a native array — must be parsed client-side.
- [x] Read `MailTemplateController`/`Service` + DTOs and documented the exact `/api/admin/mail-templates/*` (CRUD + `/preview` + `/help`) shapes. **Finding:** the `.env.example` "empty/501 in real-mode" caveat in `caseflow-fe` is stale — full real CRUD exists, no stub path. **Finding:** response has no `canEdit`/`canDelete` fields; derive from `isBuiltIn` instead.
- [x] Read `ScheduledEmailController`/`Service` + DTOs and documented the exact `/api/tickets/{ticketPublicId}/scheduled-emails` (GET/POST/DELETE) shapes, plus the confirmed `PERM_SCHEDULED_EMAIL_MANAGE` permission code string. **Confirmed:** uses `ticketPublicId` (UUID), not numeric `id` — mobile will need to resolve `publicId` for this feature specifically.
- [x] Read `CustomerReportController`/`AdminReportController`/`ReportingService` DTOs and documented all six report endpoints' exact shapes (permission `PERM_REPORT_VIEW` confirmed). **Finding, scope-relevant:** `caseflow-fe` only consumes 2 of the 6 (`/customers/{id}/reports/tickets`, `/admin/reports/customers/tickets`) — `/summary`, `/trend`, `/aging`, `/workload`, `/health` are backend-ready but unconsumed by any client. Per this task's own parity principle (match FE's *actual* UI, don't exceed it — same as the SLA-admin exclusion), **`ALIGN-002-MOBILE-EXT`'s reports scope is corrected to the 2 FE-consumed endpoints only**; see that section below.
- [x] Updated `repos/integration-map.md` and `repos/backend/frontend-contract.md`'s `TODO: Verify` markers for these four endpoint groups — all four now fully documented. Remaining `TODO: Verify` groups (Jira, SLA, tags, automation) are `ALIGN-001-BE`'s scope, untouched here.

### Mobile — Core (unblocked, contract already fully documented and proven by FE)
- [ ] Add email compose/reply UI to the conversation thread view (`POST /tickets/{id}/email/reply`, `/reply/preview`), mirroring `caseflow-fe`'s `EmailReplyComposer` at a mobile-appropriate fidelity, gated on `TICKET_EMAIL_REPLY_SEND`.
- [ ] Add AI-assist UI to `CaseDetailScreen` — ticket summary (`GET /ai-summary`) and reply-draft generation (`POST /ai-reply-draft`), gated on `AI_ASSIST`, wired to the currently-inert `EXPO_PUBLIC_ENABLE_AI` flag as the feature switch. Both endpoints always degrade gracefully server-side (`200` with `metadata.available=false`) — surface that state in the UI rather than treating it as an error.
- [ ] Add a customer detail screen (currently list-only — no detail screen exists), reachable from the customer list and from a ticket's customer reference.
- [ ] Add contacts CRUD under a customer (`/api/contacts`), mirroring `caseflow-fe`'s per-customer contacts management.
- [ ] Add customer create/update UI (`/api/customers`), gated on `CUSTOMER_MANAGE` (distinct from the `TICKET_READ`-gated read-only list mobile already has).

### Mobile — Extended (unblocked — `ALIGN-002-BE` is DONE, exact shapes now in `repos/backend/frontend-contract.md`)
- [ ] Add notification-channel admin (Slack/Teams CRUD + event-catalog view — there is no separate generic "webhook" channel type, just `SLACK`/`TEAMS`) mirroring `caseflow-fe`'s admin integrations page, gated on `PERM_INTEGRATION_CONFIG_MANAGE`. Remember to `JSON.parse` the `subscribedEvents` string field — it is not a native array in the response.
- [ ] Add mail-template admin (CRUD + live preview) mirroring `caseflow-fe`'s template management UI, gated on `PERM_EMAIL_CONFIG_VIEW`/`PERM_EMAIL_CONFIG_MANAGE`. Derive edit/delete affordance from `isBuiltIn` (built-in templates can be edited, not deleted) — the response has no `canEdit`/`canDelete` fields to read.
- [ ] Add scheduled-email management (view/schedule/cancel a delayed reply send) on the ticket conversation view, gated on `PERM_SCHEDULED_EMAIL_MANAGE`. **Uses `ticketPublicId` (UUID), not the numeric ticket `id`** — resolve `publicId` before calling this endpoint group, per ADR-0003.
- [ ] Add per-customer and admin aggregate reports + client-side PDF export, mirroring `caseflow-fe`'s **actual** reports UI, gated on `PERM_REPORT_VIEW`. Scope is exactly the two endpoints FE consumes (`/customers/{id}/reports/tickets`, `/admin/reports/customers/tickets`) — do **not** build UI for `/summary`, `/trend`, `/aging`, `/workload`, or `/health`; those are backend-ready but unconsumed by `caseflow-fe` itself, so building them in mobile would exceed FE parity rather than match it (same principle as the SLA-admin exclusion in Context).

### AI
Not applicable — see Affected Repositories above.

## Task Graph
```yaml
task_id: ALIGN-002
title: Mobile Full Feature Parity
status: READY

tasks:
  - id: ALIGN-002-BE
    repository: caseflow-be
    agent:
      provider: claude
    status: DONE
    depends_on: []

  - id: ALIGN-002-MOBILE-CORE
    repository: caseflow-mobil
    agent:
      provider: copilot
    status: READY
    depends_on: []

  - id: ALIGN-002-MOBILE-EXT
    repository: caseflow-mobil
    agent:
      provider: copilot
    status: READY
    depends_on:
      - ALIGN-002-BE

  - id: ALIGN-002-INTEGRATION
    repository: caseflow-central-brain
    agent:
      provider: codex
    status: BLOCKED
    depends_on:
      - ALIGN-002-BE
      - ALIGN-002-MOBILE-CORE
      - ALIGN-002-MOBILE-EXT
```

## Acceptance Criteria
- [ ] `caseflow-mobil` can compose, preview, and send email replies from the conversation thread view.
- [ ] `caseflow-mobil` can generate an AI ticket summary and an AI reply-draft on `CaseDetailScreen`, both gated on `AI_ASSIST` and degrading gracefully when the backend reports `metadata.available=false`.
- [ ] `caseflow-mobil` has a customer detail screen, contacts CRUD, and customer create/update — matching `caseflow-fe`'s customer management capability.
- [ ] `caseflow-mobil` has notification-channel admin, mail-template admin (with preview), and scheduled-email management — matching `caseflow-fe`'s admin capability.
- [ ] `caseflow-mobil` has per-customer and aggregate reports with PDF export — matching `caseflow-fe`'s reports capability.
- [ ] All new mobile actions are gated on `permissionCodes`, never role name (per `agents/CODE-REVIEW.md`).
- [ ] Combined with `ALIGN-001`'s acceptance criteria, `caseflow-mobil` now covers every capability in `tasks/active/MOBILE-FE-ALIGNMENT.md`'s comparison table except SLA policy administration (deliberately excluded — see Context) and mobile-only concepts with no FE equivalent (push notifications, biometric unlock) — i.e., at or above the ~80% feature-parity bar the product decision set.
- [ ] `repos/backend/frontend-contract.md` and `repos/integration-map.md` no longer carry `TODO: Verify` for the notification-channel/mail-template/scheduled-email/reports endpoint groups.
- [ ] `docs/architecture/mobile.md`'s "Responsibility" line (currently: "read-mostly mobile client") is updated to reflect the expanded scope once this task and `ALIGN-001` are both `DONE`.

## Validation
- [ ] Backend tests — N/A (no backend code change; `ALIGN-002-BE` is documentation-only)
- [ ] Frontend tests — N/A (no `caseflow-fe` code change in this task)
- [ ] Mobile tests — new coverage for email reply/preview, AI-assist, customer detail/contacts/create-edit, notification-channel admin, mail-template admin, scheduled-email, and reports+PDF export
- [ ] Integration validation — manual verification against a real `caseflow-be` instance that: (a) every new mobile action produces the same server-side effect as its `caseflow-fe` equivalent; (b) `ALIGN-002-MOBILE-CORE` and `ALIGN-002-MOBILE-EXT` changes don't conflict when merged together; (c) this task's mobile changes don't conflict with `ALIGN-001`'s mobile changes when both are merged (both touch `CaseDetailScreen` — see Notes)

## Agent Instructions

### Claude
Read `docs/architecture/backend.md`, `repos/backend/module-map.md`, and `repos/integration-map.md` first. This is a **read-and-document** sub-task, not an implementation one — do not modify `caseflow-be` source. Output is an update to `repos/backend/frontend-contract.md` and `repos/integration-map.md` with the exact confirmed shapes for notification-channels, mail-templates, scheduled-email, and reports endpoints. Do not modify `caseflow-fe`, `caseflow-mobil`, or `caseflow-ai-service`.

### Copilot (ALIGN-002-MOBILE-CORE)
Read `docs/architecture/mobile.md`, `repos/backend/frontend-contract.md`, and `caseflow-fe`'s `EmailReplyComposer`, AI-assist components, and customer management pages as the UX reference (read-only — do not modify `caseflow-fe`). Match `caseflow-fe`'s permission-gating pattern exactly (`permissionCodes`, never `roleCode`). This sub-task does not need to wait for `ALIGN-002-BE` — the endpoints it uses (email reply, AI assist, customers, contacts) are already fully documented and already proven working via `caseflow-fe`. Coordinate with (but do not block on) `ALIGN-001-MOBILE-CORE`/`ALIGN-001-MOBILE-EXT` if either is in flight concurrently — both change `CaseDetailScreen`. Do not modify `caseflow-be`, `caseflow-fe`, or `caseflow-ai-service`.

### Copilot (ALIGN-002-MOBILE-EXT)
Same repository/constraints as above, but **do not start until `ALIGN-002-BE` is DONE** — this sub-task's endpoints (notification channels, mail templates, scheduled email, reports) are the ones whose exact shapes are being confirmed there. Starting early risks building against a guessed shape.

### Codex (ALIGN-002-INTEGRATION)
Do not modify any application repository. Once `ALIGN-002-BE`, `ALIGN-002-MOBILE-CORE`, and `ALIGN-002-MOBILE-EXT` are all `DONE`, validate the combined change per the Validation section above, then update this task's `## Status` to `DONE` and move the file to `tasks/completed/`. As part of the same change, update `docs/architecture/mobile.md`'s "Responsibility" line and "Feature surface" section to reflect that mobile is no longer read-mostly, once `ALIGN-001` has also reached `DONE`. Fold any new discoveries back into the relevant Central Brain docs in the same change.

## Completion Requirements
A task is not COMPLETE until all three implementation/documentation sub-tasks and integration validation have passed. Note that `ALIGN-002-MOBILE-CORE` can reach `DONE` well before `ALIGN-002-MOBILE-EXT` (which waits on `ALIGN-002-BE`) — the parent task's status stays `IN_PROGRESS` until every node is `DONE`, per `workflows/TASK-LIFECYCLE.md`.

## Notes

### Why this is a separate task from ALIGN-001, not a rewrite of it
`ALIGN-001` is already `PLANNED` and well-scoped around a disjoint capability set (status/assign/transfer/notes/tags/Jira/attachments, plus the FE session-refresh bug). Folding this task's scope into it would conflate two different kinds of change: `ALIGN-001` is closing gaps in mobile's *existing* product surface; this task is a *product-scope expansion* (per the user's explicit decision) into capabilities mobile never had. Keeping them separate also means `ALIGN-001` isn't held up by this task's larger scope, and either can complete independently.

### Merge-coordination risk with ALIGN-001
Both tasks add UI to `CaseDetailScreen` (`ALIGN-001`: status/assign/transfer/notes/tags/Jira/attachments; this task: email reply, AI assist). If both are implemented concurrently by different agent runs, treat this the same way `ALIGN-001` treats its own Core/Ext split risk (see its Validation section) — the integration node for whichever task lands second should explicitly check for merge conflicts against the other's changes to that screen, not just its own sub-tasks.

### Scope decision record
Per the user's explicit request (see User Request above) and follow-up clarification, this task's scope is "full parity including admin" — the broader of two options presented, the narrower being case-handling-only parity (email/AI/customers) with back-office admin screens (notification channels, mail templates, scheduled email) staying web-only. The user chose full parity. If a future product decision narrows this again, split the Mobile — Extended section back out rather than deleting it.

### Not included, and why
- SLA policy administration — see Context above; `caseflow-fe` itself has no working implementation, so mobile lacking it is the aligned outcome, not a gap.
- Push notifications, biometric unlock — mobile-only concepts with no `caseflow-fe` equivalent; already tracked as their own gaps in `docs/architecture/mobile.md`'s "Open questions," not a parity question.
- Automation rules — `repos/backend/frontend-contract.md` line 456 marks `caseflow-fe`'s own consumption of this as `UNKNOWN: FE consumption status`, not verified. Since the reference client's behavior here isn't even established, this isn't ready to scope into a parity task yet — needs a `caseflow-fe` verification pass first, independent of this task.
