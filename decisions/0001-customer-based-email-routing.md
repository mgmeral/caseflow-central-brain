# ADR-0001: Route inbound email by Customer rules, not Contact lookup

## Status
Accepted

## Date
2026-03-27 (backend V8)

## Context
The original email ingestion pipeline routed inbound mail to a ticket/customer by looking up the sender's address in `ContactRepository`. This coupled routing (an operational, rule-driven concern) to contact data (a CRM record that can be incomplete, stale, or absent for a given sender), and made routing behavior hard to predict or configure per customer.

## Decision
Inbound email routing is customer-rule-based:
1. Thread match (`In-Reply-To`/`References` → existing ticket) — highest precedence.
2. `CustomerEmailRoutingRule` exact-email match.
3. `CustomerEmailRoutingRule` domain match.
4. Unknown-sender policy (`MANUAL_REVIEW` → quarantine, `IGNORE` → drop, default → quarantine).

Contact records are **not** consulted anywhere in the routing path. `EmailRoutingService` no longer calls `ContactRepository.findByEmail()`. The legacy `EmailProcessingServiceImpl` (contact-based) was deleted; `MatchingStrategy` enum and contact-centric `CustomerEmailSettings` fields (`trustedContactsOnly`, `autoCreateContact`, old `matchingStrategy`) were removed.

If unknown, write: TODO: Define this decision — *not applicable, this decision is settled.*

## Alternatives Considered
- **Contact-first lookup with customer fallback** — rejected: routing correctness became dependent on contact data completeness, which is a CRM concern owned by a different workflow than mail ingestion.
- **Hybrid (contact match boosts confidence, rule match still required)** — rejected as unnecessary complexity for the current scale.

## Consequences
- Deterministic, auditable routing: the same rule set produces the same routing decision regardless of contact data quality.
- Customers must have at least one `CustomerEmailRoutingRule` (or rely on the unknown-sender policy) to route correctly — onboarding a customer now requires configuring routing rules, not just contacts.
- Any future contact-based enhancement (e.g. "prefer contact's assigned agent") must be introduced as an additive signal, not a routing precondition, to avoid regressing this decision.

## Affected Repositories
- caseflow-be

## Related Contracts / Features / Tasks
- [contracts/api/README.md](../contracts/api/README.md) — `CustomerEmailSettingsController` / routing rule endpoints
- [repos/backend/email-flow.md](../repos/backend/email-flow.md)
- [agents/CODE-REVIEW.md](../agents/CODE-REVIEW.md) — invariant: "routing owner is Customer, not Contact"
