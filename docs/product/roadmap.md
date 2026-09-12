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
Backend has shipped through P4 (IMAP hardening) with no P1 (blocking) gaps open. AI service has shipped its P1 (no-auth, service-to-service) foundation. Mobile has shipped its Phase 1 read-mostly foundation. Frontend has a set of documented V2-deferred gaps tied to backend endpoints not yet deployed everywhere.

## Planned initiatives
- **AI service P2**: token-based service authentication (shared secret / mTLS / JWT service account), role-based endpoint access, audit logging — see [ADR-0002](../../decisions/0002-ai-service-no-auth-p1.md).
- **Backend P3 quality items**: Testcontainers-based integration tests (PostgreSQL + MongoDB) in CI; cloud object storage provider (S3/MinIO/Azure) beyond local filesystem; API rate limiting. See [repos/backend/remaining-issues.md](../../repos/backend/remaining-issues.md).
- **Frontend V2**: real backend-deployed auth in all environments, persisted role-management edits, server-side pagination/sort everywhere, outbound email reply sending (currently 501). See [repos/frontend/README.md](../../repos/frontend/README.md#api-contract).
- **Mobile**: push notifications, reply-with-attachment support — both blocked on backend/AI-service support landing first.

## Dependencies and risks
- Frontend V2 items are blocked on backend endpoints, not frontend work — do not schedule frontend effort ahead of the backend dependency.
- Mobile push/attachment work is blocked on backend support; do not start mobile-side implementation until the backend contract exists (add an ADR + contract entry first).
- AI service P2 auth should land before any deployment that could expose `caseflow-ai-service` beyond a trusted internal network.

## Open roadmap questions
- No agreed product success metrics yet — see [docs/product/product-overview.md](product-overview.md#success-metrics).
- No decision yet on whether/when to introduce an event bus for backend ↔ AI-service sync (see [docs/architecture/system-overview.md](../architecture/system-overview.md#open-questions)).
