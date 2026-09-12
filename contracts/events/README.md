# Event Contracts

## Purpose
Defines shared event-driven agreements across repositories.

## What belongs here
For each event: producer, consumer(s), topic/event name, payload, delivery expectations, and compatibility rules.

## Who should use this
Contributors and AI agents implementing asynchronous or pub/sub interactions.

## What should NOT be stored here
Message broker infrastructure internals or undocumented event changes.

## Compatibility policy
Not yet applicable — **no message broker or event bus exists in CaseFlow today.** All cross-repository communication is synchronous REST/JSON (see [contracts/api/README.md](../api/README.md)).

The closest analogue to an "event" today is internal, single-repo, and DB-backed within `caseflow-be`:
- Inbound email is a durable two-stage pipeline (`EmailIngressEvent`: `RECEIVED → PROCESSING → PROCESSED | FAILED | QUARANTINED`), processed by an in-process `EmailIngressRetryScheduler` (SKIP LOCKED batch worker) — not a message queue, and not consumed by any other repository.
- Outbound email dispatch is similarly a durable DB-backed queue (`OutboundEmailDispatch`), drained by `OutboundDispatchScheduler`.

If a real event bus (Kafka, SQS, etc.) is introduced for cross-repository use (e.g. `caseflow-be` → `caseflow-ai-service` ticket-updated events instead of manual ingest calls), record that decision as an ADR first, then populate this document with the producer/consumer/payload/compatibility contract.
