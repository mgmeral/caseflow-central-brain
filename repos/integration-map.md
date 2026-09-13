# Integration Map

## Purpose
Endpoint- and event-level detail for the cross-repository, business-critical integrations named in [dependency-map.md](dependency-map.md). Trivial/internal-only endpoints are intentionally omitted — see each repo's own docs (`repos/backend/`, `repos/ai-service/`, `repos/frontend/`, `repos/mobile/`) for the fuller endpoint inventories.

Verified against source as of **2026-09-12** (core), with a follow-up pass on **2026-09-13** for `ALIGN-002-BE`. Full request/response field lists live in [repos/backend/frontend-contract.md](backend/frontend-contract.md) — auth/ticket/email core, notification-channel admin, mail templates, scheduled email, and reports are all kept current there. **Still `TODO: Verify`** for the remaining newer controller groups (Jira, SLA, tags, automation) pending a dedicated documentation pass against those controllers/DTOs directly.

---

## REST — caseflow-fe / caseflow-mobil → caseflow-be

Cross-cutting: JWT Bearer auth on everything except `/api/auth/login` and `/api/auth/refresh`; authorization is `permissionCodes`-based (never role name); every response carries `X-Correlation-Id`; pagination shape `{items, page, size, totalElements, totalPages}`.

| Caller | Endpoint | Method | Purpose | Auth | Notes |
|---|---|---|---|---|---|
| FE, Mobile | `/api/auth/login` | POST | Issue access+refresh token pair | Public | `{username,password} → {accessToken,refreshToken,expiresIn,tokenType}` |
| FE, Mobile | `/api/auth/refresh` | POST | Rotate refresh token | Public | Mobile actually uses this (auto-retry-on-401); **FE stores the refresh token but never calls this endpoint** — verified gap, see [repos/frontend/README.md](frontend/README.md) |
| FE, Mobile | `/api/auth/me` | GET | Current user profile + `permissionCodes[]` + `ticketScope` | Bearer | Source of truth for all client-side authorization |
| FE, Mobile | `/api/tickets` | GET | Paged/filtered ticket list | `TICKET_READ` + `ticketScope` | Sort fields allowlisted server-side |
| FE, Mobile | `/api/tickets/{id}/detail` | GET | Full ticket detail (aggregates workflow/note/email/attachment/tag data) | `@ticketAuth.canReadTicket` | Auto-marks ticket notifications read (FE-observed side effect) |
| FE | `/api/tickets/{id}/status`, `/close`, `/reopen` | POST | Manual status transitions | `@ticketAuth.canChangeTicketStatus` / `TICKET_CLOSE` | Validated against `TicketStateMachineService`'s transition matrix — see [backend/ticket-rules.md](backend/ticket-rules.md) |
| Mobile | `/api/tickets/{id}/transitions` | GET | Allowed next states | `@ticketAuth.canReadTicket` | API function exists in mobile's `casesApi.ts` but **is not called from any mobile screen** — mobile has no status-change UI at all |
| FE | `/api/tickets/{id}/email/thread` | GET | Merged inbound+outbound chronological email timeline | `TICKET_EMAIL_VIEW` | Primary email-thread contract, also used by mobile (read-only there) |
| FE | `/api/tickets/{id}/email/reply` | POST | Send customer reply | `TICKET_EMAIL_REPLY_SEND` | `202 Accepted`; actually enqueues onto the durable outbound dispatch queue, not a synchronous send |
| FE | `/api/tickets/{ticketPublicId}/jira` (GET), `/jira/create`, `/jira/retry` (POST) | GET/POST | Jira issue status / create / retry | ticket-visibility + reply-permission checks | Uses the ticket's `publicId` (UUID), consistent with [ADR-0003](../decisions/0003-sequential-ticket-numbers-public-uuid.md) |
| FE | `/api/admin/sla/policies` (CRUD) | GET/POST/PUT/DELETE | SLA policy configuration | `PERM_SETTINGS_MANAGE` (`TODO: Verify` exact code name against current `Permission` enum) | **Backend implements full CRUD; the FE's `/admin/sla-policy` page does NOT call it** — it's a static explainer page telling admins to configure the backend directly. A real, verified FE/BE contract gap. |
| FE, Mobile | `/api/notifications`, `/unread-count`, `/{id}/read`, `/read-all` | GET/POST | In-app notification list/read-state | `PERM_TICKET_READ` (class-level) | Both FE (15s/10s polling) and Mobile (15s polling) — no push/WebSocket on either client |
| FE | `/api/tickets/{id}/ai-summary`, `/ai-reply-draft`, `/ai-similar-cases`, `/ai-policy-guidance` | GET/POST | AI-assisted ticket actions, backend-mediated | `PERM_AI_ASSIST` + `@ticketAuth.canReadTicket` | **FE only calls `ai-summary` and `ai-reply-draft`.** `ai-similar-cases`/`ai-policy-guidance` are implemented end-to-end on `caseflow-be` and `caseflow-ai-service` but explicitly commented as "Phase 2 — not implemented until BE is ready" in the FE source — i.e., the gap is a frontend-consumption gap, not a backend-readiness gap. Mobile does not call any AI endpoint at all (`EXPO_PUBLIC_ENABLE_AI` flag exists but is unread). |
| FE | `/api/tags`, `/api/tickets/{id}/tags` | GET/POST/DELETE | Tag catalog + per-ticket tag assignment | `TODO: Verify` exact permission codes | New since the last full doc pass — not previously documented anywhere in this repo |
| FE | `/api/admin/mail-templates` (CRUD) + `/preview` + `/help` | GET/POST/PUT/DELETE | Reusable reply templates with live preview | `PERM_EMAIL_CONFIG_VIEW`/`PERM_EMAIL_CONFIG_MANAGE` — confirmed | FE code is fully wired (except `/help`, which FE doesn't call — it hardcodes an equivalent panel). **Correction:** the `.env.example` "empty/501 in real-mode" caveat previously noted here is **stale** — full field shapes confirmed in `repos/backend/frontend-contract.md`; no stub/501 path exists in current source. |
| FE | `/api/tickets/{publicId}/scheduled-emails` (+ `/{dispatchId}` DELETE) | GET/POST/DELETE | Delay-send outbound replies | `PERM_SCHEDULED_EMAIL_MANAGE` — confirmed | Uses `publicId` (UUID), not numeric `id` — see [ADR-0003](decisions/0003-sequential-ticket-numbers-public-uuid.md). Full shape in `repos/backend/frontend-contract.md`. |
| FE | `/api/admin/integrations/jira/config`, `/test` | GET/POST | Jira integration admin config + connection test | `INTEGRATION_CONFIG_MANAGE` | Still `TODO: Verify` per-field shape (Jira remains ALIGN-001-BE's scope) |
| FE | `/api/admin/integrations/channels` (CRUD) + `/event-catalog` | GET/POST | Slack/Teams config (exactly 2 types: `SLACK`, `TEAMS` — no separate generic "webhook" type) | `PERM_INTEGRATION_CONFIG_MANAGE` — confirmed | `subscribedEvents` in the response is a JSON-encoded **string**, not a native array — clients must `JSON.parse` it. Full shape in `repos/backend/frontend-contract.md`. |
| FE | `/api/admin/ingress-events` (+ `/retry`, `/quarantine`, `/release`, `/process`) | GET/POST | Inbound email ingress ops/debug tooling | `EMAIL_OPERATIONS_VIEW`/`MANAGE` | |
| FE | `/api/dashboard/stats`, `/api/customers/{id}/reports/tickets`, `/api/admin/reports/customers/tickets` | GET | Dashboard + reporting | `PERM_REPORT_VIEW` — confirmed | PDF export is client-side only (`html-to-image` + `jspdf`), not a backend-generated file. **Four more report endpoints exist backend-side and are unconsumed by any client** — `/api/admin/reports/summary`, `/trend`, `/aging`, `/workload`, `/health` — same "ready but nobody calls it" pattern as `ai-similar-cases`/`ai-policy-guidance`. Full shapes in `repos/backend/frontend-contract.md`. |
| Mobile only | `/api/queue`, `/api/queue/stats` | GET | Unassigned-ticket admin pool | `ADMIN_POOL_VIEW` | Read-only in mobile; FE's equivalent is `AdminPoolPage` |

## REST — caseflow-be → caseflow-ai-service

| Caller | Endpoint | Method | Purpose | Request → Response | Notes |
|---|---|---|---|---|---|
| `caseflow-be` (`CaseflowAiClient`) | `/api/ai/tickets/{ticketId}/summary` | POST | Ticket summary | Ticket context (customer/status/priority/tags/messages/notes/locale) → summary + key points + risk signals + confidence | Plain LLM completion — no retrieval on the AI-service side |
| `caseflow-be` | `/api/ai/tickets/{ticketId}/reply-draft` | POST | Draft customer reply | Adds tone/template hint/`policySnippets` (caller-supplied, not retrieved) → suggested subject/body + confidence | Plain LLM completion |
| `caseflow-be` | `/api/ai/tickets/{ticketId}/similar-cases` | POST | Similar-case retrieval | `queryText`, `topK`, filters → ranked matches with snippet/score | Pure vector retrieval, no LLM call |
| `caseflow-be` | `/api/ai/tickets/{ticketId}/policy-guidance` | POST | Policy Q&A | Query + ticket context → answer + recommended actions + policy references | Retrieval-augmented generation with anti-hallucination guard (refuses to call the LLM if no policy docs retrieved) |
| `caseflow-be` | `/api/ai/ingest/documents`, `/api/ai/ingest/tickets` | POST | Synchronous vector-store ingestion | Text + metadata → `{jobId, chunksIndexed, status}` | Alternative to the Kafka async lane below |

Every call from `caseflow-be` sets `Content-Type`/`Accept: application/json`, propagates `X-Correlation-ID` and `X-Source: caseflow-be`. All failure modes (4xx/5xx/network/parse errors/circuit-open) are mapped to a single `AiServiceUnavailableException` server-side — the FE never sees a raw AI-service error, only a graceful `available:false` response. **No auth header is currently sent**, even though `caseflow-ai-service` has an (disabled-by-default) internal-API-key filter ready to receive one — `TODO: wire up when P2 auth is enabled on both sides`.

## Kafka — caseflow-be → caseflow-ai-service (optional async lane)

See [contracts/events/README.md](../contracts/events/README.md) for the full event contract. Summary: 3 topics (`ticket-ai-sync-requested`, `policy-ai-ingest-requested`, `template-ai-ingest-requested`), producer-only in `caseflow-be`, consumer-only in `caseflow-ai-service`, **both sides disabled by default** (`caseflow.ai.async.enabled=false`). When enabled, this is a strict alternative delivery path into the same ingestion engine the synchronous `/api/ai/ingest/*` REST endpoints use — not a different feature.

## What is explicitly NOT a cross-repository integration

- `caseflow-fe`/`caseflow-mobil` never call `caseflow-ai-service` directly — always via `caseflow-be`.
- Jira/Slack/Teams/webhook calls are `caseflow-be` → third-party SaaS only; no other CaseFlow repo talks to them.
- The `IntegrationJob` Postgres job queue, `EmailIngressEvent`/`OutboundEmailDispatch` pipelines, and `@Scheduled` workers are entirely internal to `caseflow-be` — not events consumed by any other repository.
