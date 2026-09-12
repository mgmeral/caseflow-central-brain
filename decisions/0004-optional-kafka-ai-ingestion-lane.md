# ADR-0004: Optional Kafka lane for AI ingestion, disabled by default

## Status
Accepted

## Date
`TODO: Verify` — introduced sometime after the last full central-brain sync (2026-04); exact date not established from source, confirmed present as of the 2026-09-12 verification pass.

## Context
`caseflow-be` needs to keep `caseflow-ai-service`'s vector index reasonably current with ticket/policy/template changes. The original mechanism was purely synchronous: explicit calls to `POST /api/ai/ingest/documents` / `/tickets`. A fully event-driven ingestion pipeline (Kafka) was added as an alternative/complementary lane, without removing the synchronous path.

## Decision
- `caseflow-be` gained a Kafka producer (`ai/event/KafkaAiEventPublisher`, with a `NoOpAiEventPublisher` fallback) publishing to three topics: `ticket-ai-sync-requested`, `policy-ai-ingest-requested`, `template-ai-ingest-requested`.
- `caseflow-ai-service` gained three matching `@KafkaListener` consumers that route into the **same** `VectorIngestionService` engine used by its synchronous REST ingest endpoints — ingestion logic (chunking, job tracking, error handling) is identical regardless of trigger source.
- Both sides are gated by a `caseflow.ai.async.enabled` property, **default `false` on both sides**. When disabled, `caseflow-be` excludes Spring's Kafka autoconfiguration entirely (zero runtime Kafka dependency), and `caseflow-ai-service`'s consumer beans are never created.
- The producer is configured for idempotent, exactly-once-oriented delivery; a DB-backed retry scheduler in `caseflow-be` (`AiIngestRetryScheduler`) retries failed publishes independent of Kafka's own guarantees.
- The consumer side has **no idempotency/dedup** implemented — this is an explicitly documented, deferred hardening item, not an oversight this ADR is unaware of.

## Alternatives Considered
- **Event bus for all cross-repo communication** — rejected; the synchronous REST contract for AI-assist and ingestion remains primary, and this Kafka lane is additive for ingestion only, not a general-purpose event bus.
- **Removing the synchronous ingest endpoints in favor of Kafka-only** — rejected (or at least not done) — both paths coexist and share the same underlying ingestion engine.

## Consequences
- This document (and [contracts/events/README.md](../contracts/events/README.md)) previously stated "no message broker or event bus exists in CaseFlow" — that was accurate at the time it was written but is now stale; this ADR and the events contract have been corrected as of 2026-09-12.
- Enabling this lane in either repository without the other is a deployment misconfiguration, not a supported partial state.
- Enabling this lane in a real environment before consumer-side idempotency is added risks duplicate ingestion-job creation and duplicate vector-store chunks for the same entity.
- Any future change to topic names, payload shapes, or the default-enabled state must update [contracts/events/README.md](../contracts/events/README.md) in the same change.

## Affected Repositories
- caseflow-be
- caseflow-ai-service

## Related Contracts / Features / Tasks
- [contracts/events/README.md](../contracts/events/README.md)
- [docs/architecture/ai-service.md](../docs/architecture/ai-service.md)
- [repos/dependency-map.md](../repos/dependency-map.md)
