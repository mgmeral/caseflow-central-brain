# AI Service Architecture

## Purpose
This document defines AI service architecture context relevant to all CaseFlow repositories.

## What belongs here
AI service responsibilities, request/response expectations, and cross-repository integration boundaries.

## Who should use this
AI service contributors and any AI/engineering agent integrating with AI capabilities.

## What should NOT be stored here
Model prompt internals that are not shared, experiment logs, or implementation-only scripts — see [repos/ai-service/](../../repos/ai-service/).

## Responsibility
`caseflow-ai-service` provides ticket summaries, reply drafts, similar-case retrieval, and policy guidance to `caseflow-be`. It is **not** a general-purpose service and has **no independent domain data of record** — it indexes a copy of ticket/policy/template text for retrieval, and separately tracks its own ingestion-job history in Postgres.

## Current state — AI capability is not uniform across the 4 endpoints (verified 2026-09-12)
This is the single most important fact to get right when reasoning about this service:

| Endpoint | Real RAG? | What it actually does |
|---|---|---|
| `POST /tickets/{id}/summary` | **No** | Plain LLM completion (Ollama via Spring AI `ChatClient`), built entirely from caller-supplied request fields (messages/notes/tags/customer/priority). No vector-store lookup at all. |
| `POST /tickets/{id}/reply-draft` | **No** | Same as above. `policySnippets` in the request is caller-supplied text, not retrieved server-side, despite the field name. |
| `POST /tickets/{id}/similar-cases` | **Yes (pure retrieval)** | Qdrant `VectorStore.similaritySearch`, filtered `sourceType=TICKET`. No LLM call, no synthesis — raw ranked matches with a snippet. |
| `POST /tickets/{id}/policy-guidance` | **Yes (retrieval + generation)** | Retrieves `sourceType=POLICY` chunks, injects into an LLM prompt. Has a genuine anti-hallucination guard: if zero policy docs are retrieved, the LLM is never called and a static "no policy found" response is returned instead. |

Do not describe this service as uniformly "RAG-powered" — 2 of its 4 headline capabilities are plain prompt-completion services that happen to share the same controller/API surface as the 2 that are genuinely retrieval-augmented.

## Components
Spring Boot 3.4.1 + Spring AI **1.0.0-M6** (a milestone/pre-GA release, requires Spring Milestones/Snapshots Maven repos). `TicketAiController` (the 4 endpoints above), `IngestController` (`/api/ai/ingest/documents`, `/api/ai/ingest/tickets`), `HealthController` (`/api/ai/health/ready`, `/api/ai/health/models`). `ChunkingService` — a **naive fixed-size character splitter** (500 chars / 50 overlap), not sentence/token-aware. `RetrievalService` is the retrieval path actually used; `RagSearchService` is a second, unused retrieval helper (dead code). Backed by Ollama (chat + embeddings, via Spring AI's Ollama starter) and Qdrant (vector store, via Spring AI's Qdrant starter — not a raw Qdrant client). Full detail: [repos/ai-service/README.md](../../repos/ai-service/README.md).

## Dependencies
Ollama (LLM inference + embeddings), Qdrant (vector search), a dedicated PostgreSQL database used **only** for ingestion-job tracking (never ticket/summary/draft content). Called by `caseflow-be` via REST (mandatory) and optionally via Kafka (3 consumer topics, disabled by default). No Spring Security dependency in the project at all.

## Communication
Synchronous REST/JSON, called only from `caseflow-be`. **No authentication as shipped** — an internal-API-key filter (`X-Internal-Api-Key` header) exists in code (`InternalAuthConfig`) but is gated off by `caseflow.ai.auth.enabled=false` by default, and `caseflow-be`'s client does not currently send that header even if it were enabled. See [ADR-0002](../../decisions/0002-ai-service-no-auth-p1.md). Must never be reachable from `caseflow-fe` or `caseflow-mobil` directly, and never exposed on a public interface without a network boundary.

## Data ownership
Owns its Qdrant vector index (embeddings of ingested documents/tickets/templates) and its own `ai_ingestion_job` Postgres table only. `caseflow-be` remains the system of record for ticket/customer data; the AI service's copy is a derived, eventually-consistent index populated by explicit ingest calls (REST, or Kafka if enabled). **No caching layer exists** — every AI-assist call re-invokes the LLM/vector store from scratch, and **no generated summary/draft/answer is ever persisted** for audit or reuse.

## External integrations
Ollama (local LLM runtime), Qdrant (vector DB). Vector collection lifecycle (creation, embedding dimensions, distance metric) is **not managed by this codebase** (`initialize-schema: false` — explicitly external responsibility).

## Security considerations
No auth in P1 by design (accepted, time-boxed risk — [ADR-0002](../../decisions/0002-ai-service-no-auth-p1.md)); P2 scaffolding (internal API key filter) now exists in code but is disabled by default and not yet wired to a caller. Deployment must enforce the service-to-service-only boundary via network policy / API gateway, not application-level auth, until P2 is actually turned on end-to-end. Multi-tenant retrieval isolation (customerId/groupId) is structurally present in metadata but **not enforced** — no Qdrant payload indexing/filtering exists yet.

## Observability
`GET /actuator/health`, `GET /actuator/info`, `GET /actuator/prometheus`, `GET /api/ai/health/ready`, `GET /api/ai/health/models`. `AiMetrics` (Micrometer counters).

## Failure handling
Each AI-assist service (summary/reply-draft/policy-guidance) catches LLM-call failures locally and returns a degraded response object (`confidence: 0.0`, warning message) rather than throwing — callers always get HTTP 200 even when Ollama is unreachable. Malformed LLM JSON output is best-effort sanitized (`LlmJsonSanitizer`); if it still won't parse, the raw text is returned with a `MODEL_OUTPUT_NOT_JSON` warning rather than erroring.

## Testing
JUnit 5, ~10 test classes (controller, per-AI-service unit tests, chunking, retrieval, ingestion, JSON sanitizer). No test exists for `RagSearchService` (consistent with it being dead code) or for the 3 Kafka consumers' actual message-handling logic (only the "async disabled" boot path is tested).

## Known limitations
- Ticket-summary and reply-draft are not RAG despite living on the "AI assist" surface — see table above.
- Chunking is naive (fixed character count), not sentence/paragraph/token aware.
- No delete-before-reindex/dedup on ingestion — re-syncing an entity adds duplicate chunks.
- Kafka consumer path (if enabled) has no idempotency — duplicate events create duplicate ingestion jobs.
- `RagSearchService` is dead code, superseded by `RetrievalService`.

## Open questions
- P2 service-auth mechanism and timeline (shared secret / mTLS / JWT service account) is still undecided in final form, though a shared-secret-header scaffold now exists disabled.
- No defined policy for keeping the Qdrant index in sync with ticket/policy updates in `caseflow-be` beyond the optional, disabled-by-default Kafka lane (ingestion otherwise remains explicit/manual via REST).
- Whether/when the frontend will consume `similar-cases`/`policy-guidance`, which are fully implemented server-side but explicitly deferred client-side (see [docs/architecture/frontend.md](frontend.md)).
