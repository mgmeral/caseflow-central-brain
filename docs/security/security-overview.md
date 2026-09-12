# Security Overview

## Purpose
This document defines shared security principles and responsibilities across CaseFlow repositories.

## What belongs here
Cross-repository security baseline, threat assumptions, ownership expectations, and compliance boundaries.

## Who should use this
Security reviewers, engineers, and AI agents touching authentication, authorization, data protection, or external integrations.

## What should NOT be stored here
Secrets, credentials, environment-specific confidential data, or incident forensics.

## Security baseline
- **Auth**: JWT Bearer tokens issued by `caseflow-be` (`POST /api/auth/login`). Access token 1h, refresh token 7d, DB-backed and hashed (SHA-256), rotated on every refresh (old refresh token revoked immediately). `caseflow-fe` and `caseflow-mobil` are issued the identical token pair, but **only mobile actually uses the refresh token** — the web frontend stores it and never calls `/api/auth/refresh`, so a web session ends on the first 401 rather than silently renewing. This is a client-side gap, not a backend contract difference.
- **caseflow-ai-service has no authentication in P1** and must only ever be reached from `caseflow-be` over a private network — see [ADR-0002](../../decisions/0002-ai-service-no-auth-p1.md). This is the single biggest cross-repo security constraint to enforce at deploy time (network policy / API gateway), not in application code. **Verified 2026-09-12**: P2 scaffolding (an internal-API-key filter) now exists in `caseflow-ai-service`'s code but is disabled by default and not called by `caseflow-be`'s client — do not treat its presence as P2 being complete.
- Email credentials (SMTP password, IMAP password/OAuth2 client secret) are **write-only**: never returned in API responses, never logged.
- **Rate limiting**: IP-based, in-process (Bucket4j) on `caseflow-be` — auth endpoints 10 req/60s, `/api/contacts/by-email` 20 req/60s (PII-enumeration protection), AI endpoints 20 req/60s, general `/api/**` 300 req/60s. **This is single-node only** — no Redis backs it, so a multi-replica deployment would let a client exceed the intended global rate by spreading requests across pods. Account lockout (brute-force protection) and a dedicated security audit log table also exist.
- **No Redis anywhere** in `caseflow-be` or `caseflow-ai-service` (verified by repo-wide grep in both) — relevant to any future session/cache/rate-limit-sharing design across replicas.

## Access control expectations
- Authorization is **permission-code based** (`permissionCodes` from `GET /api/auth/me`), enforced server-side as `PERM_<code>` Spring Security authorities. `roleCode`/`roleName` are **display-only** — clients (web and mobile) must never gate features on role name. Full permission catalog: [repos/backend/frontend-contract.md](../../repos/backend/frontend-contract.md#permission-catalog).
- `ticketScope` (`ALL` | `OWN_GROUPS` | `OWN_AND_OWN_GROUPS` | `ASSIGNED_ONLY`) further restricts which tickets a user can see, independent of permission codes.

## Data protection expectations
- Attachment binaries live in object storage (local FS / MinIO / S3-compatible), never in the database; metadata (objectKey, fileName, size, contentType, checksum) is the only DB record. See [repos/backend/storage-rules.md](../../repos/backend/storage-rules.md).
- Raw/unsanitized email HTML (`GET /emails/{id}/raw`) is audit/debug-only and must never be rendered directly by any client; the sanitized variant (`GET /emails/{id}`) is safe to render.
- Outlook/Microsoft 365 mailbox OAuth2 client secrets: restrict who can read/rotate them, prefer short-lived secrets with a documented rotation runbook, and never expose them in list/detail screens — see [repos/frontend/outlook-oauth2-imap-app-only.md](../../repos/frontend/outlook-oauth2-imap-app-only.md#8-values-to-enter-in-caseflow).

## Compliance and audit requirements
- Critical actions (assignment, transfer, status changes) must be written to history/audit trail — enforced in [repos/backend/backend-rules.md](../../repos/backend/backend-rules.md).
- Every backend response carries an `X-Correlation-Id` header for cross-system audit correlation — see [docs/operations/observability.md](../operations/observability.md).
- No formal compliance regime (SOC2/GDPR/etc.) has been defined yet for CaseFlow — treat as TODO until a decision is recorded here or in `decisions/`.
