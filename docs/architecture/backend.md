# Backend Architecture

## Purpose
This document defines backend architecture context relevant to other repositories.

## What belongs here
Backend responsibilities, interfaces, and constraints that affect shared contracts and cross-repo behavior.

## Who should use this
Backend contributors and any AI agent integrating with backend capabilities.

## What should NOT be stored here
Application source code, endpoint implementation details, or internal-only refactoring notes — see [repos/backend/](../../repos/backend/) for the full internal documentation set (module map, rules, domain specs, prompts, skills).

## Responsibility
`caseflow-be` is the single source of truth for the ticket/customer/user domain and for email ingestion/dispatch. Every other repository (`caseflow-fe`, `caseflow-mobile`, `caseflow-ai-service`) is a client of its REST API.

## Components
Modular monolith (`com.caseflow.*`), feature-based packages: `ticket` (central aggregate), `email` (ingestion/parsing/routing/dispatch), `identity` (user/group), `customer` (customer/contact + email routing rules), `workflow` (assignment/transfer/state), `note`, `storage` (object storage abstraction), `common` (cross-cutting exceptions/security/config). Full detail: [repos/backend/module-map.md](../../repos/backend/module-map.md).

## Dependencies
PostgreSQL (JPA/Flyway), MongoDB (email documents), pluggable object storage (local filesystem / MinIO / S3-compatible), optional SMTP relay + IMAP for email.

## Communication
Exposes REST/JSON over HTTP, JWT Bearer auth. Full contract: [repos/backend/frontend-contract.md](../../repos/backend/frontend-contract.md) and [contracts/api/README.md](../../contracts/api/README.md).

## Data ownership
Owns all relational (PostgreSQL) and email-document (MongoDB) CaseFlow data — see [system-overview.md](system-overview.md#data-ownership). No other repository owns or duplicates this data.

## External integrations
SMTP send, IMAP poll (including Microsoft 365 app-only OAuth2 mailboxes), object storage provider, and outbound calls to `caseflow-ai-service` for AI features (never the reverse).

## Security considerations
JWT Bearer auth (access + rotating refresh tokens), permission-code-based authorization (`permissionCodes`, not role name), IMAP/SMTP credentials write-only (never returned in API responses or logs). Full detail: [docs/security/security-overview.md](../security/security-overview.md).

## Observability
`X-Correlation-Id` on every response (MDC-based), Micrometer email metrics, `/actuator/health` (+ `/health/readiness`, `/health/liveness` in containerized deployments).

## Open questions
- Integration tests (Testcontainers for Postgres/Mongo) are not yet in CI — schema/query regressions only surface at deploy time. See [repos/backend/remaining-issues.md](../../repos/backend/remaining-issues.md).
- API rate limiting is not implemented.
