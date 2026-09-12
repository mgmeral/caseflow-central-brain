# System Overview

## Purpose
This document describes the high-level architecture shared across all CaseFlow repositories.

## What belongs here
Cross-repository architecture boundaries, interaction model, and shared responsibilities.

## Who should use this
Engineers and AI agents working across `caseflow-be`, `caseflow-fe`, `caseflow-ai-service`, and `caseflow-mobil`.

## What should NOT be stored here
Repository-specific implementation details and code-level design — see `repos/<name>/` for those, and [repos/repository-map.md](../../repos/repository-map.md) / [repos/dependency-map.md](../../repos/dependency-map.md) / [repos/integration-map.md](../../repos/integration-map.md) for verified cross-repo detail.

## Responsibility
CaseFlow is a ticket and mail-based case management system, with SLA tracking, tagging, automation rules, and third-party integrations (Jira, Slack/Teams/webhooks). `caseflow-be` is the single source of truth for tickets, customers, users, and email; every other repository is a client of its API (directly, or via `caseflow-ai-service` for AI features).

## Components
| Repo | Role |
|---|---|
| `caseflow-be` | Core domain + API. Modular monolith (Java 21 / Spring Boot 3.3.0). Owns PostgreSQL (relational entities) and MongoDB (email documents). Not microservices — one deployable unit, no service discovery/gateway. |
| `caseflow-fe` | Web UI for agents/admins (React 18 + Vite + TS). Calls `caseflow-be` REST API only. |
| `caseflow-mobil` | Mobile client (Expo/React Native + TS). Read-mostly foundation. Calls the same `caseflow-be` REST API/auth contract as the web FE. |
| `caseflow-ai-service` | AI orchestration (Spring Boot 3.4.1 + Spring AI 1.0.0-M6 + Ollama + Qdrant). Called by `caseflow-be` via REST always, and optionally via Kafka (disabled by default). Never called directly by FE/mobile. See [ADR-0002](../../decisions/0002-ai-service-no-auth-p1.md). |
| `caseflow-central-brain` | This repository — no application code; cross-repository source of truth. |

## Dependencies
```
caseflow-fe ────┐
caseflow-mobil ──┼──▶ caseflow-be ──▶ caseflow-ai-service   (REST, sync, mandatory)
                 │        │                    │
                 │        │                    ├──▶ Ollama (chat + embeddings)
                 │        │                    └──▶ Qdrant (vector store)
                 │        ├──▶ PostgreSQL
                 │        ├──▶ MongoDB (email documents)
                 │        ├──▶ MinIO/S3 or local filesystem (attachments)
                 │        ├──▶ SMTP / IMAP (incl. Microsoft 365 OAuth2)
                 │        ├──▶ Jira REST API (optional, per-tenant config)
                 │        ├──▶ Slack / Teams / webhooks (optional, per-channel config)
                 │        └──▶ Kafka → caseflow-ai-service (async AI ingestion, OPTIONAL — disabled by default both sides)
                 │
                 └── (no direct calls to caseflow-ai-service — enforced by convention on FE/mobile, not a network boundary)
```
Full detail, including which edges are mandatory vs. optional and sync vs. async: [repos/dependency-map.md](../../repos/dependency-map.md).

## Communication
- Primary cross-repo communication is synchronous REST/JSON over HTTP.
- An **optional Kafka lane** exists between `caseflow-be` (producer) and `caseflow-ai-service` (consumer) for async AI ingestion (3 topics: ticket/policy/template) — **disabled by default on both sides** (`caseflow.ai.async.enabled=false`). See [contracts/events/README.md](../../contracts/events/README.md).
- The dominant async mechanism *inside* `caseflow-be` is a Postgres-backed durable job queue (`IntegrationJob`), used for Jira/Slack/Teams/webhook delivery and as a retry path for failed AI-ingest Kafka publishes — not a pub/sub bus, and not consumed by any other repository.
- Auth: JWT Bearer tokens issued by `caseflow-be` (`POST /api/auth/login`), consumed identically by `caseflow-fe` and `caseflow-mobil` — but **only mobile actually implements silent token refresh**; the web FE stores a refresh token it never uses, so a web session dies on the first 401. `caseflow-ai-service` is called service-to-service by `caseflow-be` with no auth as shipped (an internal-API-key filter exists in the AI service's code but is disabled by default, and `caseflow-be`'s client does not send it) — see [ADR-0002](../../decisions/0002-ai-service-no-auth-p1.md).

## Data ownership
- **PostgreSQL** (owned by `caseflow-be`): tickets, customers, contacts, users/roles/groups, notes, assignments, transfers, tags, SLA policies/event log, automation rules, Jira links, notification-channel config, in-app notification records, email mailboxes/ingress/dispatch metadata, mail templates, refresh tokens, security audit log.
- **MongoDB** (owned by `caseflow-be`): `EmailDocument` (parsed email bodies/threads/attachment metadata).
- **PostgreSQL** (owned by `caseflow-ai-service`, separate database/instance): `ai_ingestion_job` only — tracks ingestion job lifecycle, never stores ticket content, summaries, or drafts.
- **Qdrant** (owned by `caseflow-ai-service`): vector embeddings of ingested tickets/policy/template documents — a derived, eventually-consistent index, populated via explicit `/api/ai/ingest/*` REST calls or (if enabled) the Kafka consumer lane. Vector collection lifecycle (creation, dimensions, distance metric) is managed externally, not by any application code.
- No other repository persists CaseFlow domain data of its own; FE/mobile hold only client-side session/UI state (`localStorage` / `expo-secure-store`).
- **No multi-tenancy**: `Customer` is a business-data concept (the external company a ticket belongs to), not a SaaS tenant/isolation boundary. All customers share one Postgres database and one Mongo database — no `tenantId` discriminator exists anywhere.

## External integrations
- Outbound/inbound email via SMTP relay and IMAP polling (including Microsoft 365 / Exchange Online app-only OAuth2 — see [repos/frontend/outlook-oauth2-imap-app-only.md](../../repos/frontend/outlook-oauth2-imap-app-only.md)).
- Jira (issue creation/linking from tickets) and Slack/Teams/generic-webhook (outbound notifications) — both delivered via `caseflow-be`'s durable `IntegrationJob` queue. `UNKNOWN: exact Jira REST endpoints/base-URL config` — not verified in the 2026-09-12 pass.
- Ollama (local LLM + embeddings) and Qdrant (vector DB) for `caseflow-ai-service`.
- Object storage (local filesystem in dev/default, MinIO/S3-compatible in Docker/K8s) for ticket/email attachments, owned by `caseflow-be`.

## Security considerations
See [docs/security/security-overview.md](../security/security-overview.md).

## Observability
See [docs/operations/observability.md](../operations/observability.md).

## Open questions
- P2 authentication for `caseflow-ai-service` has partial code scaffolding (an internal-API-key filter, disabled by default) but is not enabled, and `caseflow-be`'s client does not yet send the header — see [ADR-0002](../../decisions/0002-ai-service-no-auth-p1.md).
- Whether to enable the existing Kafka async-ingestion lane in production, and if so, how to add idempotency/dedup on the consumer side first (currently absent).
- Whether Testcontainers-based backend integration tests actually execute in CI, given a CI comment claiming no real databases are needed despite Testcontainers dependencies being present.
- The automation-rules engine's exact trigger/condition/action model was not verified in depth this pass.
