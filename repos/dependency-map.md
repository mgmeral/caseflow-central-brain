# Dependency Map

## Purpose
Verified runtime/network dependency graph between CaseFlow repositories and their backing infrastructure, as of the **2026-09-12** source-inspection pass. Every edge below was confirmed from actual code/config in the four application repositories — nothing here is inferred from "what a system like this would probably do."

## What belongs here
Direction, protocol, purpose, sync/async nature, and mandatory/optional status of each dependency edge. Endpoint-level detail belongs in [integration-map.md](integration-map.md).

---

## Diagram

```
caseflow-fe ──────────┐
                       │  REST/JSON, JWT Bearer (sync, mandatory)
caseflow-mobil ────────┼──▶ caseflow-be
                       │        │
                       │        ├──▶ PostgreSQL (sync, mandatory) — relational system of record
                       │        ├──▶ MongoDB (sync, mandatory)    — email body/document store
                       │        ├──▶ MinIO/S3 or local filesystem (sync, mandatory) — attachment binaries
                       │        ├──▶ SMTP relay (sync, mandatory for outbound email)
                       │        ├──▶ IMAP servers incl. Microsoft 365/Entra OAuth2 (sync poll, optional per mailbox)
                       │        ├──▶ Jira REST API (sync, optional — only if Jira integration configured)
                       │        ├──▶ Slack / Teams / generic webhook endpoints (sync, optional — only if channel configured)
                       │        │
                       │        ├──▶ caseflow-ai-service   REST/JSON (sync, mandatory for AI-assist features;
                       │        │                          circuit-breaker + retry; caseflow-be degrades gracefully if unreachable)
                       │        └──▶ Kafka topics: ticket-ai-sync-requested, policy-ai-ingest-requested,
                       │                 template-ai-ingest-requested (async, OPTIONAL — disabled by default via
                       │                 caseflow.ai.async.enabled=false; producer-only from caseflow-be)
                       │
                       └── (no direct calls to caseflow-ai-service from FE/mobile — enforced by convention + ADR-0002,
                            NOT by a network boundary either app repo can see)

caseflow-ai-service
    ├──▶ Ollama (sync, mandatory) — chat completion + embeddings, via Spring AI ChatClient/EmbeddingModel
    ├──▶ Qdrant (sync, mandatory for similar-cases/policy-guidance/ingestion) — vector store, via Spring AI VectorStore
    ├──▶ PostgreSQL (sync, mandatory) — ingestion-job tracking ONLY (no ticket/summary/draft data stored here)
    └──◀ Kafka topics (same 3 as above): consumer-only, OPTIONAL — disabled by default via
             caseflow.ai.async.enabled=false; no idempotency/dedup on consume
```

---

## Edge detail

| From | To | Protocol | Sync/Async | Mandatory? | Purpose |
|---|---|---|---|---|---|
| `caseflow-fe` | `caseflow-be` | REST/JSON, JWT Bearer | Sync | Mandatory | All UI data — tickets, customers, email, notifications, Jira, SLA read, reports, admin config |
| `caseflow-mobil` | `caseflow-be` | REST/JSON, JWT Bearer | Sync | Mandatory | Read-mostly: cases, customers, inbox/queue, notifications, dashboard, email thread |
| `caseflow-be` | PostgreSQL | JDBC (Hikari pool) | Sync | Mandatory | Relational system of record — tickets, customers, users/roles, SLA, tags, automation, Jira links, notification config, ingestion-adjacent email metadata |
| `caseflow-be` | MongoDB | Spring Data MongoDB | Sync | Mandatory | `EmailDocument` — full inbound/outbound email bodies + attachment metadata (large content, kept out of Postgres) |
| `caseflow-be` | MinIO / local filesystem | S3 API (MinIO client) or filesystem I/O | Sync | Mandatory (one or the other) | Ticket/email attachment binaries. `caseflow.storage.provider` toggles `local` (default, plain `mvn` run) vs `minio` (docker-compose/k8s) — a real dev-vs-deployed behavioral difference. |
| `caseflow-be` | SMTP relay | SMTP (Spring Mail) | Sync (per outbound dispatch) | Mandatory for outbound email | Sends queued `OutboundEmailDispatch` rows; per-mailbox sender, not a single global sender |
| `caseflow-be` | IMAP servers | IMAP (incl. Microsoft OAuth2/XOAUTH2) | Sync poll (`ImapPollingScheduler`, default 30s) | Optional (per-mailbox `pollingEnabled`) | Inbound email ingestion for polling-mode mailboxes |
| `caseflow-be` | Microsoft identity platform (`login.microsoftonline.com`) | HTTPS, OAuth2 client-credentials | Sync | Optional (Outlook/M365 mailboxes only) | Obtains XOAUTH2 bearer tokens for IMAP against Office365, cached in-memory, refreshed 60s before expiry |
| `caseflow-be` | Jira REST API | HTTPS | Sync, via durable job queue (`IntegrationJob`) | Optional (per-tenant Jira config) | Create/link Jira issues from tickets. `UNKNOWN: exact endpoints/base-URL config` — not verified in this pass. |
| `caseflow-be` | Slack / Teams / webhook endpoints | HTTPS | Sync, via durable job queue (`IntegrationJob`) | Optional (per-channel config) | Outbound notification delivery on configured events |
| `caseflow-be` | `caseflow-ai-service` | REST/JSON | Sync | Mandatory for AI-assist features (gracefully degrades to `available:false` if unreachable — circuit breaker + retry, never an unhandled error to the FE) | Ticket summary, reply draft, similar cases, policy guidance |
| `caseflow-be` | Kafka (producer) | Kafka protocol, idempotent producer (`acks=all`, `retries=3`, in-flight=1) | Async | **Optional — disabled by default** (`caseflow.ai.async.enabled=false`; Spring's Kafka autoconfiguration is excluded when off, so the app has zero Kafka runtime dependency unless enabled) | Alternative/complementary lane for AI ingestion (ticket/policy/template) instead of synchronous REST ingest calls. A DB-backed retry scheduler (`AiIngestRetryScheduler`) retries failed publishes independently of Kafka's own delivery guarantees. |
| `caseflow-ai-service` | Ollama | HTTP (Spring AI `ChatClient`/`EmbeddingModel`) | Sync | Mandatory | LLM chat completion (all 4 AI-assist endpoints call or attempt to call this) + embeddings (ingestion + retrieval) |
| `caseflow-ai-service` | Qdrant | gRPC (Spring AI `VectorStore`), HTTP (health check only) | Sync | Mandatory for similar-cases, policy-guidance, ingestion | Vector storage/retrieval. Collection lifecycle (creation, dimensions, distance metric) is **not managed by this codebase** (`initialize-schema: false` — external responsibility). |
| `caseflow-ai-service` | PostgreSQL | Spring Data JPA | Sync | Mandatory | Tracks `ai_ingestion_job` lifecycle only — no ticket/summary/draft content stored |
| `caseflow-be` → Kafka → `caseflow-ai-service` (consumer) | — | Kafka protocol | Async | **Optional — disabled by default** on the consumer side too (`caseflow.ai.async.enabled=false`); if ever enabled, no idempotency/dedup is implemented on the consumer (duplicate events create duplicate ingestion jobs) | Same 3 topics; routes into the identical `VectorIngestionService` used by the REST ingest endpoints |

## What does NOT exist (explicitly verified absent)

- **No direct FE/mobile → AI-service calls** — enforced by convention/code review discipline (both FE and mobile source explicitly comment "never call AI providers/caseflow-ai-service directly"), not by a network policy visible in either app repo. The actual network boundary must be enforced at deployment (see [ADR-0002](../decisions/0002-ai-service-no-auth-p1.md)).
- **No Redis anywhere** in `caseflow-be` or `caseflow-ai-service` (verified by repo-wide grep in both) — no shared cache, no shared session store, no shared rate-limit bucket store between replicas.
- **No message broker other than the optional Kafka lane above** — no SQS, RabbitMQ, etc. The dominant async mechanism inside `caseflow-be` itself is a Postgres-backed job queue (`IntegrationJob`, `SKIP LOCKED`/`PESSIMISTIC_WRITE`), not a pub/sub bus.
- **No service registry / service discovery / API gateway config** in any repo — direct REST calls with configured base URLs (env vars), not dynamic discovery.
