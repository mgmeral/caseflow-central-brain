# Product Roadmap

## Purpose
This document captures planned product direction shared across all repositories.

## What belongs here
Cross-repository initiatives, sequencing, priorities, and dependencies at product level.

## Who should use this
Product managers, engineering leads, and AI agents planning coordinated work.

## What should NOT be stored here
Repository-specific sprint tickets or low-level implementation steps — repo-local backlogs live in each repo's `repos/<name>/` doc set in this repository (e.g. [repos/backend/remaining-issues.md](../../repos/backend/remaining-issues.md)).

## Current planning horizon
As of the 2026-09-12 verification pass, the backend's feature surface has grown substantially beyond what this repository previously documented: SLA policy management, tagging, automation rules, in-app notifications, Jira integration, and Slack/Teams/webhook notification-channel delivery all now exist server-side (see [docs/architecture/backend.md](../architecture/backend.md) and [repos/repository-map.md](../../repos/repository-map.md)). The frontend has kept pace for most of these (Jira UI, notification-channel admin, tags, mail templates, dashboard/reports are all real), with two notable exceptions carried forward below. Mobile remains an intentionally read-mostly Phase-1-shaped app. The AI service now has an optional Kafka-based async ingestion lane (disabled by default) alongside its original synchronous REST ingest path.

## Planned initiatives
- **Frontend SLA policy UI**: `/admin/sla-policy` is currently a static explainer page; the backend already has full CRUD (`/api/admin/sla/policies`). Building the real UI is pure frontend work with no backend dependency.
- **Frontend AI Phase 2**: wire up `similar-cases` and `policy-guidance`, both already fully implemented on `caseflow-be` and `caseflow-ai-service` (the latter's policy-guidance path includes an anti-hallucination guard). No backend dependency — this is a frontend-only gap today, contrary to what the frontend's own "not implemented until BE is ready" code comment suggests.
- **Frontend refresh-token flow**: the frontend receives and stores a refresh token on login but never uses it to renew a session (any 401 force-logs-out immediately). `caseflow-mobil` already implements this correctly (auto-refresh-and-retry on 401) and can serve as a reference implementation.
- **AI service P2 auth**: an internal-API-key filter now exists in `caseflow-ai-service` (`InternalAuthConfig`), disabled by default (`caseflow.ai.auth.enabled=false`). Turning it on requires a coordinated change in `caseflow-be`'s AI client to start sending the header — see [ADR-0002](../../decisions/0002-ai-service-no-auth-p1.md) and [agents/CROSS-REPOSITORY-CHANGE.md](../../agents/CROSS-REPOSITORY-CHANGE.md).
- **Kafka async-ingestion hardening**: the lane exists (producer in `caseflow-be`, consumer in `caseflow-ai-service`) but is disabled by default on both sides, and the consumer has no idempotency/dedup. Hardening (dedup, monitoring) should land before flipping the default in any real environment.
- **Backend distributed-execution hardening**: several `@Scheduled` jobs have no distributed-lock guard beyond DB-level claiming on the job-queue-shaped ones, and rate limiting / AI response caching are single-node (no Redis). This is a latent risk given `k8s/hpa.yaml` implies multi-replica scaling is anticipated.
- **Mobile**: push notifications and biometric-gated unlock both have inert scaffolding today (an unread env flag, and a capability-check-only biometrics module respectively) rather than working features — either build them out or remove the misleading scaffolding. Read-write ticket workflow (status change, assignment, reply) does not exist in mobile at all yet, despite a `getCaseTransitions()` API hook already being implemented and unused.
- **Backend testing**: resolve the apparent inconsistency between the CI pipeline's "no real databases required" comment and the Testcontainers (PostgreSQL + MongoDB) dependencies/integration tests present in the build.

## Dependencies and risks
- Frontend SLA-UI, AI-Phase-2, and refresh-token work are all backend-unblocked — they should not wait on any `caseflow-be` change, contrary to how they may currently be tracked internally.
- Enabling AI-service P2 auth is a coordinated two-repo change (`caseflow-ai-service` config flip + `caseflow-be` client header) — do not enable one side without the other.
- Enabling the Kafka async lane is also a coordinated two-repo change, and should not happen before consumer-side idempotency is added.
- Mobile push/biometric/read-write-workflow work should get an explicit product decision (build it, or remove the misleading flags/hooks) before further mobile investment, rather than continuing to carry inert scaffolding.

## Open roadmap questions
- No agreed product success metrics yet — see [docs/product/product-overview.md](product-overview.md#success-metrics).
- Automation rule engine's exact trigger/condition/action model needs to be documented before it can be safely extended by another agent.
- Whether Jira's exact outbound REST contract should be formally documented in [contracts/api/README.md](../../contracts/api/README.md) (currently `UNKNOWN` — not verified against `integration/jira/service/*` in depth).
