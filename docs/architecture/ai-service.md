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
`caseflow-ai-service` provides ticket summaries, reply drafts, similar-case retrieval, and policy guidance to `caseflow-be`. It is **not** a general-purpose service and has **no independent domain data of record** — it indexes a copy of ticket/policy text for retrieval.

## Components
Spring Boot 3 + Spring AI. `TicketAiController` (4 endpoints), `IngestController`, `HealthController`, `ChunkingService`, prompt builders. Backed by Ollama (chat + embeddings) and Qdrant (vector store). Full detail: [repos/ai-service/README.md](../../repos/ai-service/README.md).

## Dependencies
Ollama (LLM inference + embeddings), Qdrant (vector search). Called exclusively by `caseflow-be`.

## Communication
Synchronous REST/JSON, called only from `caseflow-be`. **No authentication in P1** — see [ADR-0002](../../decisions/0002-ai-service-no-auth-p1.md). Must never be reachable from `caseflow-fe` or `caseflow-mobile` directly, and never exposed on a public interface without a network boundary.

## Data ownership
Owns its Qdrant vector index only (embeddings of ingested documents/tickets). `caseflow-be` remains the system of record for ticket/customer data; the AI service's copy is a derived, eventually-consistent index populated only by explicit ingest calls.

## External integrations
Ollama (local LLM runtime), Qdrant (vector DB).

## Security considerations
No auth in P1 by design (accepted, time-boxed risk — [ADR-0002](../../decisions/0002-ai-service-no-auth-p1.md)). Deployment must enforce the service-to-service-only boundary via network policy / API gateway, not application-level auth, until P2.

## Observability
`GET /actuator/health`, `GET /actuator/info`, `GET /api/ai/health/ready`, `GET /api/ai/health/models`.

## Open questions
- P2 service-auth mechanism (shared secret header vs. mTLS vs. JWT service account) is undecided.
- No defined policy yet for keeping the Qdrant index in sync with ticket/policy updates in `caseflow-be` (ingestion is currently manual/explicit, not event-driven).
