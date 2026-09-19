# BUG-001 — SLA Policy and Automation Rule Admin Endpoints Unreachable by Any Role

## Objective
Fix `SlaPolicyController` and `AutomationRuleController` in `caseflow-be`, both of
which gate every endpoint on a `PERM_SETTINGS_MANAGE` authority that no role can
ever hold — because `SETTINGS_MANAGE` does not exist in the `Permission` enum.
These 11 endpoints are structurally unreachable via the REST API today, for any
user, regardless of role. This task plans and implements the fix — it is a real
code change, not documentation-only.

## Status
READY

## Priority
P2 (real defect, but low product impact today: `caseflow-fe` doesn't call any of
these 11 endpoints yet either — see Context — so nothing currently visibly broken
for an end user; it becomes P0-relevant the moment any client tries to build
against them, e.g. `caseflow-fe`'s dormant `/admin/sla-policy` stub page)

## User Request
"PERM_SETTINGS_MANAGE bug'ı için ayrı bir bugfix task oluştur" — i.e., create a
separate bug-fix task for the `PERM_SETTINGS_MANAGE` finding surfaced while
completing `ALIGN-001-BE` and `ALIGN-002-BE`'s documentation work.

## Context
While confirming the `PERM_` prefix convention for `ALIGN-001-BE` (needed to
verify `TICKET_TAG`/`ADMIN_CONFIG` exact code strings), `identity/domain/Permission.java`
was read in full as ground truth. It has exactly 30 constants and **no
`SETTINGS_MANAGE` value**:
```
USER_MANAGE, ROLE_MANAGE, GROUP_MANAGE, ADMIN_CONFIG, TICKET_READ, ADMIN_POOL_VIEW,
TICKET_ASSIGN, TICKET_TRANSFER, TICKET_STATUS_CHANGE, TICKET_CLOSE, TICKET_PRIORITY_CHANGE,
CUSTOMER_REPLY_SEND, INTERNAL_NOTE_ADD, REPORT_VIEW, DATA_EXPORT, EMAIL_CONFIG_VIEW,
EMAIL_CONFIG_MANAGE, EMAIL_OPERATIONS_VIEW, EMAIL_OPERATIONS_MANAGE, TICKET_EMAIL_VIEW,
TICKET_EMAIL_REPLY_SEND, TICKET_TAG, INTEGRATION_CONFIG_MANAGE, INTEGRATION_JOB_VIEW,
SCHEDULED_EMAIL_MANAGE, AI_ASSIST, CUSTOMER_MANAGE, GROUP_TYPE_MANAGE, USER_READ,
ATTACHMENT_DELETE
```
`CaseFlowUserDetails.getAuthorities()` (`auth/CaseFlowUserDetails.java:68`)
mechanically generates every Spring `GrantedAuthority` as `"PERM_" + p.name()` for
each `Permission` constant a user's role actually holds — there is no other path
to a `GrantedAuthority` string anywhere in the codebase. Since `SETTINGS_MANAGE`
isn't one of the 30 constants above, `PERM_SETTINGS_MANAGE` can never be granted
to anyone, by any role, under any seeded `role_permissions` row.

**Confirmed affected — both gate every endpoint on this authority:**
- `sla/api/SlaPolicyController.java` (`/api/admin/sla/policies`) — `GET /`,
  `GET /{id}`, `POST /`, `PUT /{id}`, `DELETE /{id}`, `POST /backfill` (6 endpoints)
- `automation/api/AutomationRuleController.java` (`/api/admin/automation/rules`) —
  `GET /`, `GET /meta/triggers`, `POST /`, `PUT /{id}`, `DELETE /{id}` (5 endpoints)

**Why the test suite didn't catch this:** `AutomationRuleControllerTest.java`
uses `@WithMockUser(authorities = "PERM_SETTINGS_MANAGE")` throughout (lines 61,
74, 88, 100, 115, 125, 137, 153, 168, 179) — this seeds Spring Security's test
context with the literal string directly, bypassing `CaseFlowUserDetails`'s real
`Permission`-enum-to-authority pipeline entirely. The tests have always passed
because they never exercise the real authority-derivation path. **No test file
exists at all for `SlaPolicyController`** — its 6 endpoints have zero test
coverage of any kind, permission-related or otherwise.

**Product impact today is limited but not zero:** per `repos/backend/frontend-contract.md`
and `repos/integration-map.md` (updated during `ALIGN-001-BE`/`ALIGN-002-BE`),
`caseflow-fe`'s `/admin/sla-policy` page is a static explainer that never calls
this API anyway, and no client calls the automation-rules API at all — so this
bug has not caused a visible incident. It matters because `ALIGN-002`'s explicit
decision to exclude SLA-policy admin from mobile's feature-parity scope rests
partly on "the reference client doesn't have a working implementation either" —
this task doesn't change that exclusion's validity, but it does mean the
underlying reason is deeper than "no FE UI was built," and it blocks anyone
(including a future `caseflow-fe` fix to that stub page) from building working
admin UI against either endpoint group until fixed.

See [docs/architecture/backend.md](../../docs/architecture/backend.md),
[repos/backend/frontend-contract.md](../../repos/backend/frontend-contract.md)
(Permission Catalog section — "Bug found, not fixed"),
[repos/integration-map.md](../../repos/integration-map.md).

## Affected Repositories

| Repository | Agent | Responsibility | Status |
|---|---|---|---|
| caseflow-be | Claude | Fix the permission wiring on both controllers, add real test coverage that exercises the actual `Permission`-enum-to-authority path (not a hardcoded mock string), and update Central Brain docs to remove the "bug found, not fixed" framing once it's actually fixed. | READY |

No other repository is touched — `caseflow-fe`/`caseflow-mobil` don't call either
endpoint group today (see Context), so there is nothing for a client to react to.
If `ALIGN-002`'s SLA-admin exclusion or a future `caseflow-fe` fix to its
`/admin/sla-policy` stub ever changes as a result of this becoming reachable,
that's separate follow-up work, not part of this bug fix.

## Dependencies

### Depends On
None.

### Blocks
None known yet. Practically unblocks: any future task that wants to build a
real SLA-policy-admin or automation-rules-admin UI in `caseflow-fe` or
`caseflow-mobil` — those would be blocked on this fix landing first, but no such
task exists yet.

## Contract Impact
Per `skills/contract-change/SKILL.md` (triggered because this touches
authorization): the recommended fix (see Tasks below) changes an authority
string checked by `@PreAuthorize`, which the skill's own classification calls
a breaking change ("changed/removed permission code") by default.
**Assessed as non-breaking in practice, not just by policy**, because:
- API: No endpoint shape, DTO, or path changes — only the `@PreAuthorize`
  authority string on 11 existing endpoints.
- Authentication/Authorization: **Yes** — this is the entire point of the fix.
  However, `PERM_SETTINGS_MANAGE` currently has zero possible holders (see
  Context), so no currently-working access is removed or narrowed for anyone —
  the fix can only ever *add* reachability, never take it away, because there
  is no baseline of working access to regress from.
- Event: None. Database: None (if the `ADMIN_CONFIG`-reuse fix option below is
  chosen; a new-enum-constant option would need a migration — see Tasks).
- No ADR needed per the skill's own criterion ("if it reflects a real
  architectural decision, not just a shape fix") — this is a shape/bug fix
  correcting a typo-shaped defect, not a new architectural decision.
- Still: update `repos/backend/frontend-contract.md`'s "Bug found, not fixed"
  note to reflect the fix in the same change, per the skill's Output
  requirements.

## Tasks

### BE
- [ ] Decide the fix approach (see "Two valid fix options" in Notes) — default
      recommendation: repoint both controllers' `@PreAuthorize("hasAuthority('PERM_SETTINGS_MANAGE')")`
      to `@PreAuthorize("hasAuthority('PERM_ADMIN_CONFIG')")`, reusing the
      existing, already-seeded `ADMIN_CONFIG` permission (already used for tag
      catalog management — see `V22__p1_closure.sql`). This requires no new
      Flyway migration. Confirm with a human before applying if there's a
      product reason SLA/automation admin should be a *separate* permission
      from tag-catalog admin — nothing in current source suggests one, but this
      task shouldn't unilaterally assume that was never intended.
- [ ] Apply the chosen fix to `sla/api/SlaPolicyController.java` (6
      `@PreAuthorize` annotations) and `automation/api/AutomationRuleController.java`
      (5 `@PreAuthorize` annotations).
- [ ] Rewrite `AutomationRuleControllerTest.java`'s permission tests to source
      the authority from the real `Permission` enum rather than a hardcoded
      `"PERM_SETTINGS_MANAGE"` string (e.g. `"PERM_" + Permission.ADMIN_CONFIG.name()`,
      or whatever constant the chosen fix lands on) — the whole reason this bug
      shipped undetected is that the existing tests bypass the real
      authority-derivation path. Also add a test asserting a user **without**
      the new permission gets `403`, which the current suite never checks either.
- [ ] Add a new `SlaPolicyControllerTest.java` — none exists today. Cover all 6
      endpoints with both an authorized and an unauthorized case, following the
      corrected pattern from the automation-rule test above.
- [ ] Update `repos/backend/frontend-contract.md`'s Permission Catalog section
      (the "Bug found, not fixed" paragraph) and `repos/integration-map.md`'s
      SLA-policy row to reflect the fix, in the same change.

## Task Graph
```yaml
task_id: BUG-001
title: Unreachable SLA/Automation Admin Permissions
status: READY

tasks:
  - id: BUG-001-BE
    repository: caseflow-be
    agent:
      provider: claude
    status: READY
    depends_on: []
```

Single-repository task — no integration node per `workflows/TASK-DEPENDENCIES.md`'s
integration-node rule, which applies to task graphs spanning more than one
repository.

## Acceptance Criteria
- [ ] A user holding the corrected permission can successfully call all 11
      previously-unreachable endpoints; a user without it gets `403`.
- [ ] `AutomationRuleControllerTest` no longer hardcodes `"PERM_SETTINGS_MANAGE"`
      as a mock authority string — it derives the expected authority from the
      real `Permission` enum, so a future rename of the underlying constant
      would break the test (correctly) instead of silently continuing to pass.
- [ ] `SlaPolicyController` has test coverage for the first time.
- [ ] `repos/backend/frontend-contract.md` and `repos/integration-map.md` reflect
      the fix — no repository still describes these endpoints as unreachable.
- [ ] No other repository (`caseflow-fe`, `caseflow-mobil`, `caseflow-ai-service`)
      needed a code change to keep working, confirming the "non-breaking in
      practice" assessment in Contract Impact was correct.

## Validation
- [ ] Backend tests — new/updated coverage per the BE checklist above; full
      `caseflow-be` suite must still pass (in particular, confirm nothing else
      in the codebase — a role seed script, an integration test, a fixture —
      also hardcodes `PERM_SETTINGS_MANAGE` and would need updating alongside
      the two controllers)
- [ ] Frontend tests — N/A, no `caseflow-fe` change
- [ ] Mobile tests — N/A, no `caseflow-mobil` change
- [ ] Integration validation — N/A, single-repository change; manually verify
      against a real backend instance that a role holding the corrected
      permission can now `GET /api/admin/sla/policies` and
      `GET /api/admin/automation/rules` successfully (both currently return
      `403` for every role in the current seed data)

## Agent Instructions

### Claude
Read `docs/architecture/backend.md`, `repos/backend/frontend-contract.md`'s
Permission Catalog section (the `PERM_` prefix rule and the bug writeup — full
context for why this is happening), and `identity/domain/Permission.java` +
`auth/CaseFlowUserDetails.java` (the authority-derivation code path) first. This
is a real code change, unlike `ALIGN-001-BE`/`ALIGN-002-BE`'s documentation-only
sub-tasks — implement the fix, don't just describe it. Do not modify
`caseflow-fe`, `caseflow-mobil`, or `caseflow-ai-service` — nothing there needs
to change (see Context/Affected Repositories for why). Before applying the
`ADMIN_CONFIG`-reuse fix, grep the whole `caseflow-be` source tree (not just
the two controllers) for any other `PERM_SETTINGS_MANAGE` reference — the two
controllers and one test file are the only ones found during this task's own
investigation, but re-check, since a stale reference elsewhere (a seed script,
a fixture, a comment pointing at the wrong string) would be easy to miss and
would undermine the fix if left behind.

## Completion Requirements
A task is not COMPLETE until the fix is applied, tested per the checklist
above, and the two Central Brain docs are updated in the same change. Move
this file to `tasks/completed/` and set `## Status` to `DONE` once all of the
above holds, per `workflows/TASK-LIFECYCLE.md`.

## Notes

### Two valid fix options — a decision, not a foregone conclusion
1. **Reuse `ADMIN_CONFIG`** (recommended default). Already exists, already
   seeded onto the Admin role since `V10__email_v6_enhancements.sql`, already
   used for exactly this shape of capability (tag catalog admin, per
   `V22__p1_closure.sql`'s own comment: "ADMIN_CONFIG continues to govern the
   global tag vocabulary"). No migration needed. Risk: conflates SLA-policy and
   automation-rule admin with tag-catalog admin under one permission — fine if
   that was always the intent, wrong if a human actually wanted these scoped
   separately (nothing in current source suggests the latter, but this task
   can't fully rule it out without asking).
2. **Add a real `SETTINGS_MANAGE` constant** to `Permission.java` and seed it
   onto the appropriate role(s) via a new Flyway migration. More faithful to
   whatever the original (evidently incomplete) intent behind the
   `PERM_SETTINGS_MANAGE` string was, but requires a migration and a decision
   about which role(s) should get it by default — more moving parts for a bug
   fix with no currently-observed product urgency.

This task's own checklist defaults to option 1 but explicitly asks the
implementing agent to confirm with a human first rather than deciding
unilaterally — this is exactly the kind of permission-boundary product
question this framework's own rules say shouldn't be assumed silently.

### Relationship to ALIGN-001/ALIGN-002
This bug was discovered as a side effect of `ALIGN-001-BE`'s work (confirming
the `PERM_` prefix convention needed for `TICKET_TAG`/`ADMIN_CONFIG`), not
something either task set out to find. It's deliberately a separate task
rather than folded into either, because it's a `caseflow-be`-only defect fix
with no mobile/FE-alignment shape to it — see `agents/GLOBAL.md`'s
multi-agent coordination rule (fold discoveries back into Central Brain, but
that doesn't mean cramming an unrelated fix into an in-flight task's scope).
