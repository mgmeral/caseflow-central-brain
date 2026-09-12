# Feature: AI Ticket Assist

## Purpose
Give agents AI-assisted ticket summaries, reply drafts, similar-case lookup, and policy guidance without leaving the ticket view, always mediated by `caseflow-be` (never a direct client call to `caseflow-ai-service`).

## Scope
**Implemented end-to-end and consumed by the web frontend:** ticket summary, reply draft.
**Implemented end-to-end on both backend repos, but NOT consumed by any client yet:** similar-case lookup, policy guidance. The frontend's own source explicitly comments these as "Phase 2 — not implemented until BE is ready," which is stale — both are backend-ready today.
**Not implemented anywhere:** any AI feature in `caseflow-mobil` (its `EXPO_PUBLIC_ENABLE_AI` flag is defined but never read).

Out of scope: this feature does not cover email ingestion/routing, SLA, or Jira — those are separate features.

## Repositories Affected
- caseflow-be — `ai` module: `AiAssistantController`, `CaseflowAiClient` (circuit breaker + retry), `TicketAiResponseCache` (Postgres)
- caseflow-fe — `ai.service.ts`, `AiSummaryCard.tsx`, `AiReplyDraftCard.tsx` (summary + reply-draft only)
- caseflow-ai-service — `TicketAiController`, `TicketSummaryService`, `ReplyDraftService`, `SimilarCasesService`, `PolicyGuidanceService`, `RetrievalService`
- caseflow-mobil — not affected (no AI UI exists)

## Contract Impact
See [contracts/api/README.md](../contracts/api/README.md) and [repos/integration-map.md](../repos/integration-map.md). No pending contract changes; the gap is purely a frontend-consumption one for 2 of the 4 capabilities.

## Decision Dependencies
[ADR-0002](../decisions/0002-ai-service-no-auth-p1.md) (no-auth constraint on `caseflow-ai-service`). No ADR governs the RAG-vs-plain-completion split per endpoint — it's an implementation detail, not an architectural decision, but is important enough to be documented in [docs/architecture/ai-service.md](../docs/architecture/ai-service.md).

## Implementation Tasks
None active — see [tasks/active/](../tasks/active/).

## Rollout / Compatibility Notes
- `caseflow-be` degrades gracefully (`available:false`) if `caseflow-ai-service` is unreachable — this must be preserved by any client consuming these endpoints; never let an AI failure block or corrupt a ticket action.
- Building the frontend's similar-cases/policy-guidance UI requires no backend work — it is purely frontend implementation against an existing, stable contract.
- Ticket-summary and reply-draft prompts are built entirely from data the caller already has (ticket messages/notes/tags) — there is no server-side retrieval to keep in sync for these two, unlike similar-cases/policy-guidance which depend on the ingestion pipeline being kept current.

## Done Criteria
Frontend calls all 4 AI-assist endpoints where a UI affordance makes sense (or an explicit product decision documents why similar-cases/policy-guidance remain out of scope); the RAG-vs-plain-completion distinction stays documented and accurate as the AI service evolves.
