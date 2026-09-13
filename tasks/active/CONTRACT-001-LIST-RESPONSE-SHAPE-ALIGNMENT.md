# CONTRACT-001 — Backend/Frontend List-Response Shape Alignment

## Objective
Fix every confirmed live case where `caseflow-fe` misparses a `caseflow-be` list-endpoint response (silent empty list or wrong URL), convert the one backend endpoint that deviates from the documented pagination envelope, and harden the remaining list-parsing call sites with a shared unwrap utility — bringing every list endpoint into conformance with the cross-cutting pagination convention already documented in `contracts/api/README.md`.

## Status
PLANNED

## Priority
P1 (two confirmed user-facing breakages: Customer Email Settings customer-picker renders empty; Ingress Events admin page is non-functional. Remaining scope is P2 hardening.)

## User Request
"Analyze whether there's a BE/FE contract mismatch [beyond the users/customers bug already fixed this session] and open a new task for it" (paraphrased from Turkish, asked in plan mode).

## Context
During local dev bring-up (2026-09-13), `GET /api/users` and `GET /api/customers` were found to return `PagedResponse<T>` (`{items, page, size, totalElements, totalPages}`), while `caseflow-fe`'s `userService.getAll()` and `customerService.getAll()` assumed a bare array — both admin pages silently rendered empty, no console error (React Query swallowed the failure). Both were hotfixed this session (`caseflow-fe` commit `a3b1402`). This task is the follow-up systematic audit, since `contracts/api/README.md` already states `{items,...}` as the pagination convention for *all* endpoints — any non-conforming endpoint/client pair is a bug, not an open design question. See `contracts/api/README.md` (Pagination section) and `repos/backend/frontend-contract.md` (Pagination section — currently only documents `/tickets` and `/admin/ingress-events` as paginated, which is itself incomplete and part of what this task corrects).

## Affected Repositories

| Repository | Agent | Responsibility | Status |
|---|---|---|---|
| caseflow-fe | GitHub Copilot | Fix 2 confirmed-broken services + 1 broken-and-misrouted service; extract a shared list-unwrap utility from `role.service.ts`'s private `toArrayPayload`; adopt it across the ~12 other list-parsing call sites identified in Notes | READY |
| caseflow-be | Claude | Convert `IngressEventAdminController.list` to return `PagedResponse` instead of a raw Spring Data `Page`, matching every other paginated controller | READY |
| caseflow-central-brain | Codex | Update `repos/backend/frontend-contract.md`'s Pagination section to enumerate every paginated vs. bare-array endpoint explicitly | BLOCKED (needs BE+FE final shapes) |
| caseflow-mobil | — | Not affected — mobile doesn't consume `/users`, `/customers`, `/contacts`, or `/admin/ingress-events`; these are admin-only surfaces mobile intentionally excludes (per `repos/repository-map.md`) | N/A |
| caseflow-ai-service | — | Not affected — no FE/mobile call touches this service | N/A |

## Dependencies

### Depends On
None — root task.

### Blocks
None known yet.

Internal ordering: `CONTRACT-001-INTEGRATION` depends on both `CONTRACT-001-BE` and `CONTRACT-001-FE` (the doc update needs the final shapes). `CONTRACT-001-BE` and `CONTRACT-001-FE` are independent of each other — `ingress.service.ts`'s FE fix can target the documented target shape (`PagedResponse`) directly without waiting on the BE change landing first, though landing them in the same PR is reasonable.

## Contract Impact
- API: Yes — `GET /api/admin/ingress-events` response shape changes from raw Spring Data `Page<T>` (`{content, pageable, totalElements, totalPages, size, number, sort, first, last, numberOfElements, empty}`) to `PagedResponse<T>` (`{items, page, size, totalElements, totalPages}`). Classified **breaking** per `skills/contract-change/SKILL.md` (changed response structure), even though the only current FE consumer (`ingress.service.ts`) is already non-functional against this endpoint today (calls the wrong path: `/admin/ingress/events` instead of the real `/admin/ingress-events`) — so no working client depends on the old shape in practice.
- Event: None.
- Database: None.
- Authentication: None.
- Must update `repos/backend/frontend-contract.md`'s Pagination section in the same change per `skills/contract-change/SKILL.md`'s Output requirement.

## Tasks

### BE
- [ ] Convert `IngressEventAdminController.list` (`email/api/IngressEventAdminController.java`) to build a `PagedResponse<IngressEventAdminResponse>` via the existing `PagedResponse.from(Page<T>)` factory (`common/api/PagedResponse.java`) — same pattern already used by `CustomerController`/`ContactController`/`UserController` — instead of returning the raw Spring Data `Page`.
- [ ] Grep for any other backend caller/test asserting the old `{content, pageable, ...}` shape for this endpoint and update it.

### FE
- [ ] Fix `contact.service.ts`'s `getAll()` (and `getByCustomer()` if it hits the same endpoint) to unwrap `PagedResponse` — currently does `res.map(toContact)` directly on the raw response.
- [ ] Fix `customerEmailSettings.service.ts`'s `listCustomers()` — an independent second consumer of `/api/customers` that was missed when `customer.service.ts` was patched this session; same `{items,...}` unwrap needed.
- [ ] Fix `ingress.service.ts`: correct the request path from `/admin/ingress/events` to `/admin/ingress-events`, and update its response parsing to the `PagedResponse` `{items,...}` shape (coordinate with the BE task above).
- [ ] Extract `role.service.ts`'s private `toArrayPayload` helper (handles `items`/`content`/`data`/`results`) into a shared module (`src/services/normalizers.ts` or a new `src/lib/apiList.ts`), and adopt it in every `getAll`/`list*` method currently doing an unguarded `res.map(...)` or a narrow `items`-only check — full file list in Notes below — so future backend shape drift fails loud in one place instead of silently per-file.
- [ ] Add a `?? []` fallback wherever a narrow `items`-only check currently lacks one (`ticket.service.ts` `getAll`, `queue.service.ts`, `notification.service.ts`, `ingress.service.ts`) so a malformed response degrades to an empty list instead of throwing.

### Mobile
Not applicable — see Affected Repositories.

### AI
Not applicable — see Affected Repositories.

## Task Graph
```yaml
task_id: CONTRACT-001
title: Backend/Frontend List-Response Shape Alignment
status: PLANNED

tasks:
  - id: CONTRACT-001-BE
    repository: caseflow-be
    agent:
      provider: claude
    status: READY
    depends_on: []

  - id: CONTRACT-001-FE
    repository: caseflow-fe
    agent:
      provider: copilot
    status: READY
    depends_on: []

  - id: CONTRACT-001-INTEGRATION
    repository: caseflow-central-brain
    agent:
      provider: codex
    status: BLOCKED
    depends_on:
      - CONTRACT-001-BE
      - CONTRACT-001-FE
```

## Acceptance Criteria
- [ ] Customer Email Settings admin page's customer picker lists all customers (currently empty).
- [ ] Wherever `contactService.getAll()` is consumed, it renders actual contacts, not an empty list.
- [ ] Ingress Events admin page loads real data against the correct URL and shape (currently non-functional).
- [ ] `GET /api/admin/ingress-events` returns `{items, page, size, totalElements, totalPages}`.
- [ ] A shared list-unwrap utility exists and is used by every FE service in the audit list (Notes) instead of ad-hoc/duplicated logic.
- [ ] `repos/backend/frontend-contract.md`'s Pagination section explicitly enumerates every paginated endpoint (users, customers, contacts, tickets, tickets/admin-pool, queue, notifications, admin report aggregate, admin/ingress-events) and notes which endpoints are deliberately bare arrays.

## Validation
- [ ] Backend tests — add/update a test asserting `/api/admin/ingress-events`'s response has an `items` field, not `content`/`pageable`.
- [ ] Frontend tests — cover the 3 fixed services' list methods against a `{items,...}` fixture; unit-test the new shared unwrap utility against all known shapes (`items`/`content`/`data`/`results`, bare array, malformed input).
- [ ] Mobile tests — N/A.
- [ ] Integration validation — against a real local `caseflow-be`: exercise Customers, Users, Contacts, Customer Email Settings picker, and Ingress Events admin page; confirm no console errors and row counts match direct DB/`curl` counts (same method that caught the original bug).

## Agent Instructions

### Claude (CONTRACT-001-BE)
Read `repos/backend/module-map.md` and `email/api/IngressEventAdminController.java` first. Change only this controller's list method to return `PagedResponse`, reusing the existing `PagedResponse.from(Page<T>)` factory — do not invent a new envelope type, do not touch any other controller. Do not modify `caseflow-fe`.

### Copilot (CONTRACT-001-FE)
Read `contracts/api/README.md`'s Pagination section and this task's Notes (full per-file audit) first. Fix the 3 confirmed-broken services, then extract and adopt the shared unwrap utility across the remaining files listed in Notes. Do not modify `caseflow-be`.

### Codex (CONTRACT-001-INTEGRATION)
Do not modify application code. Once BE and FE nodes are `DONE`, update `repos/backend/frontend-contract.md`'s Pagination section per Acceptance Criteria — verify against actual current source, not this task file, which may have drifted — then mark this task `DONE` and move it to `tasks/completed/`.

## Completion Requirements
Per `workflows/TASK-LIFECYCLE.md` — not COMPLETE until BE, FE, and the integration/doc-update node are all `DONE`.

## Notes

### Full per-file FE audit (this task's research pass)
**Confirmed broken now** (backend returns `PagedResponse`, FE assumes bare array or wrong URL):
- `contact.service.ts` — `getAll`, `getByCustomer`
- `customerEmailSettings.service.ts` — `listCustomers`
- `ingress.service.ts` — wrong URL (`/admin/ingress/events` vs. real `/admin/ingress-events`) *and* wrong shape (expects `items`/bare array; backend returns raw Spring `Page` `{content,...}`)

**Safe today, but unguarded (Category A — no defensive check at all)** — will silently break the moment backend adds pagination there without a matching FE update:
`tag.service.ts` (listActiveTags/listAllTags/listTicketTags), `group.service.ts` (getAll), `groupType.service.ts` (getAll), `channelIntegration.service.ts` (listChannelConfigs), `customerEmailSettings.service.ts` (listRoutingRules), `note.service.ts` (getByTicket), `transfer.service.ts` (getByTicket), `ticket.service.ts` (getMessages ×2, getTransferHistory), `ticketEmail.service.ts` (listThread), `scheduledEmail.service.ts` (listScheduledEmails — no parsing at all).

**Safe today, narrow `items`-only unwrap (Category B)**, mostly missing a `?? []` fallback:
`ticket.service.ts` (getAll), `queue.service.ts` (getQueue), `notification.service.ts` (getAll), `template.service.ts` (getAll — BE is bare array, degrades correctly), `mailbox.service.ts`/`normalizeMailboxList` (BE is bare array, degrades correctly; separately sends dead `page`/`size` params BE ignores — cosmetic, out of scope).

**Reference implementation (Category C)**: `role.service.ts`'s `toArrayPayload` (checks `items`/`content`/`data`/`results`) — the only robust helper in the codebase; promote it per the FE Tasks above.

### Backend response-shape inventory (this task's research pass)
Three shapes coexist, from the shared `PagedResponse<T>` record (`common/api/PagedResponse.java`):
1. `PagedResponse<T>` `{items, page, size, totalElements, totalPages}` — users, customers, contacts, tickets (+admin-pool), queue, notifications, admin report aggregate.
2. Bare array `[...]` — groups, roles, group-types, tags (+all, +ticket-tags), history, notes, attachments (all 3 variants), email threads/dispatches, mail-templates, mailboxes, scheduled-emails, automation rules, sla policies, channel configs (+event-catalog), transfers, report trend/health, permission catalog.
3. Raw Spring Data `Page<T>` `{content, pageable, ...}` — only `GET /api/admin/ingress-events`. The sole outlier from the two conventions above, and the only backend shape this task changes.

### Explicitly out of scope
- Standardizing the bare-array endpoints (groups, roles, etc.) onto `PagedResponse` — these are small reference-data sets, not currently buggy; doing so would be its own larger breaking-change task with its own deployment coordination.
- `mailbox.service.ts`'s dead `page`/`size` params — cosmetic, not a rendering bug.
- Adding pagination to bare-array endpoints for future scale — a capacity/product decision, not a contract-conformance bug.
