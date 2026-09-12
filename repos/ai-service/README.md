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
