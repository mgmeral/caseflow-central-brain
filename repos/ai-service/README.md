# CaseFlow AI Service (`caseflow-ai-service`)

AI orchestration layer for CaseFlow customer support ticket workflows, built with Spring Boot 3.x and Spring AI.

> **Access model:** This service is **strictly service-to-service**. It is designed to be called only by the CaseFlow backend, never directly by browser frontends or end users. No authentication is implemented in P1 (Phase 1) — token-based service auth is planned for **P2**. **Do not expose this service on a public network or allow direct frontend access.**

## Architecture overview

```
┌─────────────────────────────────────────────────────────────┐
│                    CaseFlow AI Service                      │
│                                                             │
│  ┌──────────┐   ┌──────────────┐   ┌───────────────────┐  │
│  │   REST   │──▶│   Services   │──▶│   Spring AI       │  │
│  │   API    │   │  (AI/Ingest) │   │  (ChatClient /    │  │
│  │Controllers│  │              │   │  VectorStore)     │  │
│  └──────────┘   └──────────────┘   └───────────────────┘  │
│                        │                     │              │
│                        ▼                     ▼              │
│               ┌──────────────┐     ┌──────────────────┐   │
│               │  Prompt      │     │  Ollama (LLM +   │   │
│               │  Builders    │     │  Embeddings)     │   │
│               └──────────────┘     └──────────────────┘   │
│                                              │              │
│                                    ┌──────────────────┐   │
│                                    │  Qdrant (Vector  │   │
│                                    │  Store)          │   │
│                                    └──────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

**Key components:**
- `TicketAiController` — 4 AI endpoints per ticket: summary, reply draft, similar cases, policy guidance
- `IngestController` — ingest documents and tickets into the vector store
- `HealthController` — service readiness and model status
- Spring AI — abstracts Ollama chat + embeddings and Qdrant vector search
- `ChunkingService` — splits documents into overlapping text chunks for indexing
- Prompt builders — construct structured prompts instructing the LLM to output JSON

`/similar-cases` and `/policy-guidance` perform vector search against **Qdrant** and return empty results until policies/resolved tickets are ingested via `/api/ai/ingest/documents` or `/api/ai/ingest/tickets`.

> **Verified 2026-09-12 — RAG is not uniform across the 4 endpoints.** `/summary` and `/reply-draft` are plain LLM completions (Ollama via Spring AI `ChatClient`) built entirely from the caller-supplied request body — **no Qdrant lookup happens for these two**, despite living on the same controller. Only `/similar-cases` (pure retrieval, no LLM call) and `/policy-guidance` (retrieval + generation, with an anti-hallucination guard that skips the LLM entirely when no policy docs are found) are genuinely retrieval-augmented. Full detail: [../../docs/architecture/ai-service.md](../../docs/architecture/ai-service.md).
>
> Spring AI version in use is **1.0.0-M6** — a milestone/pre-GA release (requires Spring Milestones/Snapshots Maven repos, not on Maven Central).
>
> An optional Kafka consumer lane exists for async ingestion (3 topics, mirrored by a producer in `caseflow-be`), **disabled by default** via `caseflow.ai.async.enabled=false` on both sides — see [ADR-0004](../../decisions/0004-optional-kafka-ai-ingestion-lane.md) and [../../contracts/events/README.md](../../contracts/events/README.md). No idempotency/dedup exists on the consumer side yet.
>
> An internal-API-key auth filter (`InternalAuthConfig`, header `X-Internal-Api-Key`) exists in code but is disabled by default (`caseflow.ai.auth.enabled=false`), and `caseflow-be`'s client does not currently send this header — see [ADR-0002](../../decisions/0002-ai-service-no-auth-p1.md).
>
> No caching layer exists (every call re-invokes the LLM/vector store), and no generated summary/draft/answer is ever persisted. `RagSearchService` in source is dead code, superseded by `RetrievalService`.

## Local startup (without Docker)

Prerequisites: Java 21+, Maven 3.9+, [Ollama](https://ollama.ai) on port 11434, [Qdrant](https://qdrant.tech) on HTTP 6333 / gRPC 6334.

```bash
ollama pull llama3.1
ollama pull nomic-embed-text
docker run -p 6333:6333 -p 6334:6334 qdrant/qdrant

cd caseflow-ai-service
mvn clean package -DskipTests
java -jar target/caseflow-ai-service-*.jar
```

Service available at `http://localhost:8081`. Swagger UI: `/swagger-ui.html`, OpenAPI JSON: `/api-docs`.

## Docker Compose

```bash
docker-compose up --build
```

Starts `caseflow-ai-service` (8081), `ollama` (11434), `ollama-init` (one-shot model pull), `qdrant` (6333 HTTP / 6334 gRPC).

Optional dev-only chat UI for Ollama (**never expose to untrusted networks — no auth**):

```bash
docker-compose --profile dev up --build   # adds open-webui on :3000
```

## API endpoints

Ticket AI: `POST /api/ai/tickets/{ticketId}/summary`, `/reply-draft`, `/similar-cases`, `/policy-guidance`
Ingest: `POST /api/ai/ingest/documents`, `POST /api/ai/ingest/tickets`
Health: `GET /api/ai/health/ready`, `GET /api/ai/health/models`

See Swagger UI for full request/response schemas.

## Additional environment variables (verified 2026-09-12, not previously documented here)

| Variable | Default | Description |
|---|---|---|
| `ASYNC_ENABLED` / `caseflow.ai.async.enabled` | `false` | Master toggle for the Kafka consumer lane (3 topics). Disabled → zero Kafka runtime dependency. |
| `caseflow.ai.auth.enabled` | `false` | Master toggle for the internal-API-key auth filter. Disabled → every endpoint is unauthenticated. |
| `caseflow.ai.retrieval.similarity-threshold` | `0.6` | Minimum similarity score for retrieval matches (similar-cases, policy-guidance). |
| `caseflow.ai.default-top-k` | `5` | Default retrieval result count when a caller doesn't specify `topK`. |

## Environment variables

| Variable | Default | Description |
|---|---|---|
| `SERVER_PORT` | `8081` | HTTP port |
| `OLLAMA_BASE_URL` | `http://localhost:11434` | Ollama base URL |
| `OLLAMA_CHAT_MODEL` | `llama3.1` | Chat/generation model |
| `OLLAMA_EMBED_MODEL` | `nomic-embed-text` | Embedding model |
| `QDRANT_HOST` | `localhost` | Qdrant hostname |
| `QDRANT_PORT` | `6334` | Qdrant gRPC port (Spring AI VectorStore) |
| `QDRANT_HTTP_PORT` | `6333` | Qdrant HTTP port (health checker) |
| `QDRANT_COLLECTION_DOCUMENTS` | `caseflow-documents` | Qdrant collection name |
| `SPRING_PROFILES_ACTIVE` | `default` | Active Spring profile |

Alternative models: chat — `mistral`, `llama3.2`, `qwen2.5`; embeddings — `mxbai-embed-large`, `all-minilm`.

## Security and access model

> ⚠️ **Service-to-service only — no direct frontend access.** Authentication is intentionally omitted in P1; all endpoints are open for local development/integration testing. Token-based service auth (shared secret header, mTLS, or JWT service account) is planned for P2, along with role-based endpoint access and audit logging.
>
> Do not expose port 8081 on a public interface without a network boundary, allow browser/mobile clients to call it directly, or deploy without an API gateway / network policy in production.

## Actuator endpoints

`GET /actuator/health`, `GET /actuator/info`
