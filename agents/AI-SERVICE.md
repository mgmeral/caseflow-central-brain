# AI Service Agent Instructions

## Purpose
Defines AI-service-specific agent context and boundaries.

## What belongs here
AI service repository usage rules, interface expectations, and decision dependencies.

## Who should use this
Agents operating on `caseflow-ai-service`.

## What should NOT be stored here
Backend/frontend/mobile-only implementation guidance.

## Current guidance

Read before changing `caseflow-ai-service`:
1. [../repos/ai-service/README.md](../repos/ai-service/README.md) — architecture, endpoints, environment variables, security/access model.
2. [ADR-0002](../decisions/0002-ai-service-no-auth-p1.md) — why P1 ships with no authentication, and the hard constraint that comes with it.

Rules:
- This service is **service-to-service only**, called exclusively by `caseflow-be`. Never add a code path that lets `caseflow-fe` or `caseflow-mobil` call it directly, and never assume it will be reachable from a public network.
- Do not add authentication yourself outside of the P2 initiative described in [ADR-0002](../decisions/0002-ai-service-no-auth-p1.md) without first updating that ADR — auth mechanism choice (shared secret / mTLS / JWT service account) is an open decision, not an implementation detail to improvise per-endpoint. Note: an `InternalAuthConfig` filter already exists in code, disabled by default — do not flip it on without also updating `caseflow-be`'s client to send the header, in the same change (see [../agents/CROSS-REPOSITORY-CHANGE.md](CROSS-REPOSITORY-CHANGE.md)).
- `/similar-cases` and `/policy-guidance` depend on prior ingestion (`/api/ai/ingest/documents`, `/api/ai/ingest/tickets`, or the optional Kafka consumer lane if enabled) — empty results are expected, not a bug, until data has been ingested for that environment.
- **Do not assume `/summary` and `/reply-draft` use retrieval** — they are plain LLM completions built from the caller-supplied request body only. Only `/similar-cases` and `/policy-guidance` touch Qdrant. If you change this, update [../docs/architecture/ai-service.md](../docs/architecture/ai-service.md) in the same change — this distinction is load-bearing for anyone reasoning about the service's behavior.
- The optional Kafka consumer lane (`caseflow.ai.async.enabled`, default `false`) has no idempotency/dedup — do not enable it in a shared/production-like environment without adding that first, and never enable it without confirming the producer side in `caseflow-be` is also enabled (see [ADR-0004](../decisions/0004-optional-kafka-ai-ingestion-lane.md)).
- The Qdrant vector index is a derived copy, not the system of record — `caseflow-be` remains authoritative for ticket/customer data (see [../docs/architecture/system-overview.md](../docs/architecture/system-overview.md#data-ownership)).
- Any new endpoint or payload shape consumed by `caseflow-be` is a contract change — update [../contracts/api/README.md](../contracts/api/README.md) in the same change.
