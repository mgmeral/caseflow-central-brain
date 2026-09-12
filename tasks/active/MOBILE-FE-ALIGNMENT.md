# Task: Mobile ↔ Frontend Alignment Audit

## Status
Active — analysis complete, no implementation started. This is a documentation/planning task only; no code was changed to produce it.

## Purpose
Compare `caseflow-mobil` against `caseflow-fe`, using the `caseflow-be` API contract ([repos/backend/frontend-contract.md](../../repos/backend/frontend-contract.md), [repos/integration-map.md](../../repos/integration-map.md)) as the reference for what's actually available to build against, and surface every point where mobile is inconsistent with — or behind — the web frontend. Verified against source as of the 2026-09-12 inspection pass (see [repos/repository-map.md](../../repos/repository-map.md), [docs/architecture/frontend.md](../../docs/architecture/frontend.md), [docs/architecture/mobile.md](../../docs/architecture/mobile.md)).

## Method
Both clients call the identical `caseflow-be` contract (same JWT auth, same `permissionCodes`, same endpoints) — there is no mobile-only API. Any capability gap below is therefore a **client-side implementation gap**, not a missing backend contract, unless explicitly noted otherwise.

---

## Comparison by capability area

| Capability | BE contract available | caseflow-fe | caseflow-mobil | Alignment |
|---|---|---|---|---|
| **Session refresh** | `POST /auth/refresh` rotates the refresh token; both clients receive the same token pair on login | **Stores `refreshToken` but never calls `/auth/refresh`** — any 401 force-logs-out immediately | **Correctly implements it** — `refreshSession()` auto-retries once on 401, deduped via in-flight promise | 🔴 **Inverted** — mobile is the *more correct* implementation here. This is a bug to fix in FE, not a gap to fill in mobile. |
| **Token storage** | N/A (client choice) | `localStorage` (readable by any script on the page) | `expo-secure-store` (OS-level secure storage) | ⚠️ Not a functional gap, but mobile's approach is the stronger security posture — worth FE reconsidering if a comparable web-safe mechanism exists, though this isn't a contract issue. |
| **Ticket status change / close / reopen** | `POST /tickets/{id}/status`, `/close`, `/reopen`; `GET /tickets/{id}/transitions` returns the allowed next states | Full UI: status change, close, reopen, all gated by fetching `/transitions` first (never hardcodes the matrix) | **None.** `getCaseTransitions()` is implemented in the mobile API layer (`casesApi.ts`) but is never called from any screen — no status-change UI exists at all | ⚠️ Mobile gap — the API call already exists, only the UI is missing |
| **Assignment / transfer** | `POST /tickets/{id}/assign`, `/{id}/transfer`, unassign | Full UI (`AssignmentModal`, `TransferModal`, transfer history panel) | **None** | ⚠️ Mobile gap |
| **Internal notes (add)** | `POST /tickets/{id}/notes` | Full UI, incl. @-mentions | **Notes are not addable** — mobile's case detail is read-only for the ticket timeline | ⚠️ Mobile gap |
| **Tags** | `/api/tags`, `/api/tickets/{id}/tags` (add/remove) | Full CRUD UI (`TicketTagsCard`, `TagManagementPage`) | **None** — no tag display or management anywhere in mobile | ⚠️ Mobile gap |
| **Email — read thread** | `GET /tickets/{id}/email/thread` | Full thread view | Full thread view (labeled in-app "email-backed") | ✅ Aligned |
| **Email — compose/reply** | `POST /tickets/{id}/email/reply` (+ `/reply/preview`) | Full compose/preview/send UI (`EmailReplyComposer`) | **None** — thread is read-only, no reply/compose UI at all | ⚠️ Mobile gap |
| **Email — attachments** | Attachment metadata returned on ticket/email detail | `AttachmentViewerModal` (view/download) | `UNKNOWN: Not established in source repository` — not confirmed present or absent in the mobile inspection pass; likely absent given the read-only conversation view, but not directly verified | ⚠️ Mobile gap (probable) — flagged for a follow-up code check before assuming either way |
| **AI assist — summary / reply-draft** | `GET /ai-summary`, `POST /ai-reply-draft` (plain LLM completion, not RAG — see [features/ai-ticket-assist.md](../../features/ai-ticket-assist.md)) | Both implemented, click-to-generate | **None.** `EXPO_PUBLIC_ENABLE_AI` flag exists (default `true`) but is never read anywhere in mobile code | ⚠️ Mobile gap |
| **AI assist — similar-cases / policy-guidance** | Both implemented server-side (genuine RAG) | **Not implemented either** — commented "Phase 2" in FE source | Not implemented | ✅ Aligned (both clients equally behind the backend here — not a Mobile-vs-FE inconsistency) |
| **SLA display** | SLA fields on ticket detail response | `SLAIndicator.tsx` — presentational, renders backend-computed state | Displayed in `CaseDetailScreen` (due dates/state) | ✅ Aligned (read-only in both) |
| **SLA policy administration** | `/api/admin/sla/policies` full CRUD | Route exists (`/admin/sla-policy`) but is a **static explainer, no real CRUD** | Not present (mobile has no admin surface at all) | ✅ Aligned in outcome (neither client has working SLA admin), though for different reasons — FE has a misleading stub page, mobile simply has no admin section. See [features/sla-management.md](../../features/sla-management.md). |
| **Jira integration** | Ticket-level status/create/retry + admin config | Full UI (`JiraIntegrationCard`, admin settings page) | **None** | ⚠️ Mobile gap |
| **Notification channels (Slack/Teams/webhook) admin** | Full CRUD | Full admin UI | **None** (no admin surface in mobile at all) | ⚠️ Mobile gap — though likely out of scope by design, since mobile has no other admin screens either; treat as a scope question, not necessarily a bug |
| **In-app notifications** | `GET /notifications`, unread-count, mark-read, mark-all-read | Polling (15s list / 10s unread-count), bell dropdown | Polling (15s), full-screen list, mark-read/mark-all-read | ✅ Aligned — both are polling-only (neither has push/WebSocket), both call the identical endpoints |
| **Push notifications** | No backend push contract exists (mobile-only concept; not applicable to a browser FE in the same way) | N/A | `EXPO_PUBLIC_ENABLE_PUSH` flag exists, never read; no push library/permission/token-registration anywhere | N/A for comparison — this is a mobile-only gap already tracked in [docs/architecture/mobile.md](../../docs/architecture/mobile.md), not a Mobile-vs-FE inconsistency |
| **Admin pool / unassigned queue** | `GET /queue`, `/queue/stats`, gated `ADMIN_POOL_VIEW` | `AdminPoolPage` — wired to the same endpoints; `UNKNOWN: Not established` whether it offers a claim/assign action from the pool view (not independently verified this pass) | `InboxScreen` — explicitly **read-only**, "no claim/assign action from this screen" (verified) | ⚠️ Possible gap — needs a direct FE code check to confirm whether FE's admin pool view allows claiming a ticket; if it does, that's a concrete mobile gap. Currently `TODO: Verify` on the FE side. |
| **Customers** | Full CRUD + contacts + per-customer email settings | Full: list, detail, create/update, activate/deactivate, contacts CRUD, email-settings admin | **List only** — no detail screen, no contacts, no create/edit, no linking from customer to their tickets | ⚠️ Mobile gap |
| **Dashboard stats** | `GET /dashboard/stats` | `DashboardPage` (stat cards, activity feed, my-tickets table) | `HomeScreen` (dashboard) — same endpoint | ✅ Aligned at the data-source level; mobile's presentation is simpler but not contract-inconsistent |
| **Reports (per-customer / aggregate) + PDF export** | `GET /customers/{id}/reports/tickets`, `/admin/reports/customers/tickets` | Full UI + client-side PDF export | **None** | ⚠️ Mobile gap |
| **Mail templates, scheduled email** | Full CRUD / schedule-send | Full UI | **None** | ⚠️ Mobile gap (consistent with no compose/reply UI at all in mobile) |
| **Permission-code gating discipline** | `permissionCodes[]` from `/auth/me`; never gate on role name | Correctly gates on `permissionCodes` throughout (`usePermissions()`, ~18 flags) | Correctly gates on `permissionCodes` (`hasPermission()`), used for the Inbox tab (`ADMIN_POOL_VIEW`) | ✅ Aligned in principle — mobile simply has far fewer gated actions because it has far fewer actions overall |
| **Ticket identifier usage (`id` vs `publicId`)** | Core ticket endpoints use numeric `id`; Jira/scheduled-email/some email-detail endpoints use `publicId` (UUID) per [ADR-0003](../../decisions/0003-sequential-ticket-numbers-public-uuid.md) | **Inconsistent even within itself**: `ticketEmail.service.ts` uses `:id` for thread and reply-send, but `:publicId` for reply-preview and email-detail | Uses numeric `id` throughout (only reads `/tickets/:id/email/thread`, `/tickets/:id/detail`, `/tickets/:id/transitions`) | ℹ️ Not a Mobile-vs-FE gap today (mobile's surface is small enough to not hit the `publicId`-only endpoints), but **flag for future mobile work**: if mobile ever builds Jira, scheduled-email, or the email-reply/preview flow, it must use `publicId` for those specific endpoints, not `id` — the two identifiers are not interchangeable and FE's own mixed usage is not a reliable pattern to copy from directly. |

---

## Priority findings

### P0 — Fix in caseflow-fe, not caseflow-mobil
1. **Refresh-token flow is missing in the web frontend, not in mobile.** `caseflow-mobil` already has the correct reference implementation (`refreshSession()` with in-flight dedup, auto-retry-on-401). Port that pattern into `caseflow-fe` rather than treating this as a mobile deficiency. See [docs/architecture/frontend.md](../../docs/architecture/frontend.md).

### P1 — Real mobile capability gaps (backend contract already supports these; mobile has no UI)
2. Ticket status change / close / reopen (transitions API already wired, unused).
3. Assignment and transfer.
4. Internal notes (add).
5. Ticket tags.
6. Email compose/reply.
7. AI assist (summary, reply-draft — both already implemented server-side and consumed by FE).
8. Jira integration (ticket-level view/create/retry).
9. Customer detail, contacts, and reports.

### P2 — Needs verification before scoping further
10. Whether FE's `AdminPoolPage` allows claiming a ticket from the pool (if yes, that's a concrete mobile gap to add; if FE is also read-only there, this is aligned, not a gap).
11. Whether mobile's email thread view supports attachment viewing/download at all.

### Not a gap — explicitly confirmed aligned or out of scope
- AI similar-cases/policy-guidance: neither client implements these yet (backend-ready, frontend-consumption gap on both sides equally — see [features/ai-ticket-assist.md](../../features/ai-ticket-assist.md)).
- SLA policy administration: neither client has a working admin UI (FE's page is a non-functional stub; mobile has no admin surface at all).
- Notification-channel admin, mail templates, scheduled email: mobile has no admin section at all, consistent with its documented read-mostly, non-admin scope — likely a deliberate product boundary rather than an oversight, pending a product decision either way.
- Push notifications: a mobile-only concept already tracked as its own gap in [docs/architecture/mobile.md](../../docs/architecture/mobile.md); not a Mobile-vs-FE inconsistency since FE has no equivalent push concept either.

## Related documents
- [docs/architecture/frontend.md](../../docs/architecture/frontend.md), [docs/architecture/mobile.md](../../docs/architecture/mobile.md)
- [repos/backend/frontend-contract.md](../../repos/backend/frontend-contract.md), [repos/integration-map.md](../../repos/integration-map.md)
- [decisions/0003-sequential-ticket-numbers-public-uuid.md](../../decisions/0003-sequential-ticket-numbers-public-uuid.md)
- [features/ai-ticket-assist.md](../../features/ai-ticket-assist.md), [features/sla-management.md](../../features/sla-management.md), [features/jira-integration.md](../../features/jira-integration.md)

## Next steps
Needs a product decision on which P1 gaps (if any) are intentional scope boundaries for a "read-mostly Phase 1" mobile app versus genuinely planned follow-up work, before turning any row above into an implementation task. The P0 refresh-token fix has no such ambiguity and can be scheduled independently.
