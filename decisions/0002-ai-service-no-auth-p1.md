# ADR-0002: AI service ships without authentication in Phase 1

## Status
Accepted

## Date
2026-04 (AI integration foundation)

## Context
`caseflow-ai-service` needs to be callable by the backend for ticket summaries, reply drafts, similar-case search, and policy guidance. Building full service-to-service authentication (shared secret, mTLS, or JWT service accounts) before the AI feature set was proven would have delayed initial delivery.

## Decision
Phase 1 (P1) ships the AI service with **no authentication on any endpoint**, under the explicit constraint that it is deployed as a **service-to-service-only** component:
- Never exposed on a public interface without a network boundary.
- Never called directly by browser frontends, mobile apps, or any untrusted client — only by `caseflow-be`.
- Never deployed without a surrounding API gateway or network policy in production.

Phase 2 (P2) will add token-based service authentication (shared secret header, mTLS, or JWT service account), role-based endpoint access, and audit logging.

## Alternatives Considered
- **Ship with a shared-secret header from day one** — rejected for P1 to avoid blocking feature delivery on infrastructure that every environment (including local dev) would then require.
- **Delay AI features until full service auth exists** — rejected as unnecessary given the service is not internet-facing in any current deployment target.

## Consequences
- This is an **accepted, time-boxed risk**, not a permanent architecture choice. Any deployment that exposes `caseflow-ai-service` on a shared or public network before P2 auth lands is a policy violation, not a supported configuration.
- Ops/deployment docs and the `caseflow-central-brain` security overview must keep this restriction visible until P2 ships.
- When P2 lands, this ADR should be superseded and the AI service agent instructions updated.

## Affected Repositories
- caseflow-ai-service
- caseflow-be (the only permitted caller)

## Related Contracts / Features / Tasks
- [docs/security/security-overview.md](../docs/security/security-overview.md)
- [agents/AI-SERVICE.md](../agents/AI-SERVICE.md)
- [repos/ai-service/README.md](../repos/ai-service/README.md)
