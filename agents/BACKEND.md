# Backend Agent Instructions

## Purpose
Defines backend-specific agent context and boundaries.

## What belongs here
Backend repository usage rules, integration expectations, and references to shared contracts.

## Who should use this
Agents operating on `caseflow-be`.

## What should NOT be stored here
Frontend/mobile/AI-service-only implementation guidance.

## Current guidance

Read, in order, before generating or reviewing backend code:
1. [../repos/backend/architecture.md](../repos/backend/architecture.md) and [../repos/backend/module-map.md](../repos/backend/module-map.md) — module boundaries, package placement, dependency direction.
2. [../repos/backend/backend-rules.md](../repos/backend/backend-rules.md) — layering, DTO, transaction, and error-handling conventions.
3. The relevant domain spec under [../repos/backend/domain-specs/](../repos/backend/domain-specs/), plus [../repos/backend/ticket-rules.md](../repos/backend/ticket-rules.md), [../repos/backend/email-flow.md](../repos/backend/email-flow.md), or [../repos/backend/storage-rules.md](../repos/backend/storage-rules.md) as applicable.
4. [../repos/backend/current-state.md](../repos/backend/current-state.md) and [../repos/backend/remaining-issues.md](../repos/backend/remaining-issues.md) — what already exists and what is a known, accepted gap (don't "fix" a documented, accepted gap without checking it isn't intentional).

Scaffolding prompts/recipes: [../repos/backend/prompts/](../repos/backend/prompts/) and [../repos/backend/skills/](../repos/backend/skills/).

Non-negotiable invariants (see [CODE-REVIEW.md](CODE-REVIEW.md) for the full review checklist):
- Routing owner is Customer, not Contact — [ADR-0001](../decisions/0001-customer-based-email-routing.md).
- IMAP/SMTP credentials are write-only — never returned in responses or logged.
- Ticket status transitions must be business-valid, not purely linear — see [../repos/backend/ticket-rules.md](../repos/backend/ticket-rules.md).
- Any change to a request/response DTO shape or enum is a contract change — update [../contracts/api/README.md](../contracts/api/README.md), [../contracts/schemas/README.md](../contracts/schemas/README.md), and [../repos/backend/frontend-contract.md](../repos/backend/frontend-contract.md) in the same change, and flag it as breaking if it is.

`caseflow-be` has no upstream backend dependency of its own — it is the root of the dependency graph (see [../docs/architecture/system-overview.md](../docs/architecture/system-overview.md)). Changes here are the ones most likely to require coordinated changes in `caseflow-fe`, `caseflow-mobil`, and `caseflow-ai-service`.
