# Event Contracts

## Purpose
Defines shared event-driven agreements across repositories.

## What belongs here
For each event: producer, consumer(s), topic/event name, payload, delivery expectations, and compatibility rules.

## Who should use this
Contributors and AI agents implementing asynchronous or pub/sub interactions.

## What should NOT be stored here
Message broker infrastructure internals or undocumented event changes.

## Status (corrected 2026-09-12)
**A Kafka-based event lane now exists**, contrary to what this document previously stated ("no message broker or event bus exists"). It is real, implemented on both sides, and **disabled by default on both sides** — treat it as present-but-dormant, not absent.

## Kafka — AI async ingestion lane

- **Producer:** `caseflow-be` (`ai/event/KafkaAiEventPublisher`), gated by `caseflow.ai.async.enabled` (default `false`). When disabled, Spring's Kafka autoconfiguration is excluded entirely — the app has zero Kafka runtime dependency.
- **Consumer:** `caseflow-ai-service` (3 `@KafkaListener` beans in `messaging/consumer/`), independently gated by its own `caseflow.ai.async.enabled` (default `false`).
- **Enabling one side without the other is a deployment misconfiguration**, not a supported partial state — if the producer publishes and no consumer is listening, messages simply accumulate/expire per Kafka's own retention; if the consumer is enabled without the producer, it just never receives anything via this lane (ingestion can still happen via the synchronous REST endpoints).

| Topic | Producer | Consumer | Payload | Purpose |
|---|---|---|---|---|
| `ticket-ai-sync-requested` | `caseflow-be` | `caseflow-ai-service` (`TicketAiSyncConsumer`) | `AiSyncEvent`/`TicketAiSyncRequestedEvent`: ticketId, subject, body, resolutionSummary, customerName, status, tags, sourceVersion, metadata, correlationId | Async alternative to `POST /api/ai/ingest/tickets` — feed ticket text into the vector index |
| `policy-ai-ingest-requested` | `caseflow-be` | `caseflow-ai-service` (`PolicyAiIngestConsumer`) | policyId, title, content, sourceVersion, locale, metadata, correlationId | Async alternative to `POST /api/ai/ingest/documents` for policy documents |
| `template-ai-ingest-requested` | `caseflow-be` | `caseflow-ai-service` (`TemplateAiIngestConsumer`) | templateCode, name, body, sourceVersion, locale, metadata, correlationId | Async alternative to `POST /api/ai/ingest/documents` for mail templates |

**Delivery semantics:**
- Producer is configured for idempotent, exactly-once-oriented delivery (`enable.idempotence=true`, `acks=all`, `retries=3`, `max.in.flight.requests.per.connection=1`).
- Producer-side publish failures are **not silently lost**: a DB-backed retry scheduler (`AiIngestRetryScheduler`, default every 5 minutes) retries FAILED ingestion jobs recorded in `caseflow-be`'s `AiIngestionJob` Postgres entity, independent of Kafka's own guarantees.
- Consumer deserialization uses `ErrorHandlingDeserializer` + a `GenericKafkaEvent` fallback type to avoid poison-pill startup failures; `spring.kafka.admin.fail-fast: false` means `caseflow-ai-service` boots fine even if Kafka is completely unreachable.
- **Consumer-side idempotency is NOT implemented.** Duplicate events for the same ticketId+sourceVersion each create a separate ingestion job — explicitly documented in source as a "planned hardening item," not yet done. Do not enable this lane in a production-like environment without adding dedup first.
- All three topics route into the same shared `VectorIngestionService.ingest(...)` engine used by the synchronous REST ingest endpoints — chunking, job tracking, and error handling are identical regardless of trigger source.
- **No events flow in the other direction** — `caseflow-ai-service` has no Kafka producer code at all; it never publishes an "ingestion completed" or similar event back to `caseflow-be`.

## What is still NOT event-driven

The following remain synchronous, single-repo, DB-backed mechanisms — not events consumed by another repository, and not part of the Kafka lane above:
- Inbound email is a durable two-stage pipeline (`EmailIngressEvent`: `RECEIVED → PROCESSING → PROCESSED | FAILED | QUARANTINED`), processed by an in-process `EmailIngressRetryScheduler` (SKIP LOCKED batch worker) — internal to `caseflow-be` only.
- Outbound email dispatch is a similarly durable DB-backed queue (`OutboundEmailDispatch`), drained by `OutboundDispatchScheduler` — internal to `caseflow-be` only.
- Jira issue creation/linking and Slack/Teams/webhook notification delivery are processed via a shared Postgres-backed job queue (`IntegrationJob`, `SKIP LOCKED`/`PESSIMISTIC_WRITE` claiming) — internal to `caseflow-be` only, and calls an external third-party API directly, not another CaseFlow repository.
- `caseflow-be` → `caseflow-ai-service` remains primarily a **synchronous REST call** for the 4 AI-assist endpoints (summary, reply-draft, similar-cases, policy-guidance) — there is no async/event path for these, only for ingestion.

## Compatibility policy
Any change to a topic name, payload shape, or default-enabled state is a contract change: update this document in the same change that touches `caseflow.ai.async.*` configuration or the event DTOs (`AiSyncEvent`, `TicketAiSyncRequestedEvent`, `PolicyAiIngestRequestedEvent`, `TemplateAiIngestRequestedEvent`), and flag it as breaking if consumers on an already-enabled deployment would need to change.
