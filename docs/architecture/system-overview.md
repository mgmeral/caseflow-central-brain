# System Overview

## Purpose
This document describes the high-level architecture shared across all CaseFlow repositories.

## What belongs here
Cross-repository architecture boundaries, interaction model, and shared responsibilities.

## Who should use this
Engineers and AI agents working across `caseflow-be`, `caseflow-fe`, `caseflow-ai-service`, and `caseflow-mobile`.

## What should NOT be stored here
Repository-specific implementation details and code-level design — see `repos/<name>/` for those.

## Responsibility
CaseFlow is a ticket and mail-based case management system. `caseflow-be` is the single source of truth for tickets, customers, users, and email; every other repository is a client of its API (directly, or via `caseflow-ai-service` for AI features).

## Components
| Repo | Role |
|---|---|
| `caseflow-be` | Core domain + API. Modular monolith (Java 21 / Spring Boot 3). Owns PostgreSQL (relational entities) and MongoDB (email documents). |
| `caseflow-fe` | Web UI for agents/admins (React 18 + Vite + TS). Calls `caseflow-be` REST API only. |
| `caseflow-mobile` | Mobile client (Expo/React Native + TS), Phase 1 foundation. Calls the same `caseflow-be` REST API/auth contract as the web FE. |
| `caseflow-ai-service` | AI orchestration (Spring Boot 3 + Spring AI + Ollama + Qdrant). Called only by `caseflow-be` — never directly by FE/mobile. See [ADR-0002](../../decisions/0002-ai-service-no-auth-p1.md). |
| `caseflow-central-brain` | This repository — no application code; cross-repository source of truth. |

## Dependencies
```
caseflow-fe ────┐
caseflow-mobile ┼──▶ caseflow-be ──▶ caseflow-ai-service
                 │        │
                 │        ├──▶ PostgreSQL
                 │        └──▶ MongoDB (email documents)
                 │
                 └── (no direct calls to caseflow-ai-service)
```
`caseflow-ai-service` additionally depends on Ollama (LLM + embeddings) and Qdrant (vector store).

## Communication
- All inter-repo communication is synchronous REST/JSON over HTTP.
- No message broker / event bus exists yet — see [contracts/events/README.md](../../contracts/events/README.md).
- Auth: JWT Bearer tokens issued by `caseflow-be` (`POST /api/auth/login`), consumed by `caseflow-fe` and `caseflow-mobile` identically. `caseflow-ai-service` is called service-to-service by `caseflow-be` with no auth in P1 (see [ADR-0002](../../decisions/0002-ai-service-no-auth-p1.md)).

## Data ownership
- **PostgreSQL** (owned by `caseflow-be`): tickets, customers, contacts, users, groups, notes, assignments, transfers, email mailboxes/ingress/dispatch metadata, refresh tokens.
- **MongoDB** (owned by `caseflow-be`): `EmailDocument` (parsed email bodies/threads).
- **Qdrant** (owned by `caseflow-ai-service`): vector embeddings of ingested tickets/policy documents. Populated only via explicit `/api/ai/ingest/*` calls — not automatically synced from `caseflow-be`.
- No other repository persists CaseFlow domain data of its own; FE/mobile hold only client-side session/UI state (localStorage / `expo-secure-store`).

## External integrations
- Outbound/inbound email via SMTP relay and IMAP polling (including Microsoft 365 / Exchange Online app-only OAuth2 — see [repos/frontend/outlook-oauth2-imap-app-only.md](../../repos/frontend/outlook-oauth2-imap-app-only.md), which documents the Entra/Exchange side of mailbox onboarding).
- Ollama (local LLM + embeddings) and Qdrant (vector DB) for `caseflow-ai-service`.
- Object storage (local filesystem in dev, MinIO/S3-compatible in Docker/K8s) for ticket attachments, owned by `caseflow-be`.

## Security considerations
See [docs/security/security-overview.md](../security/security-overview.md).

## Observability
See [docs/operations/observability.md](../operations/observability.md).

## Open questions
- No formal event/message contract exists yet between `caseflow-be` and `caseflow-ai-service` (ingestion is a synchronous REST call today) — define if async ingestion becomes necessary.
- `caseflow-ai-service` P2 authentication design (shared secret vs. mTLS vs. JWT service account) is not yet decided — see [ADR-0002](../../decisions/0002-ai-service-no-auth-p1.md).
