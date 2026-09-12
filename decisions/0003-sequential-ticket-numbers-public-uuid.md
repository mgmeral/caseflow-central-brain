# ADR-0003: Sequential ticket numbers for display, numeric PK internally

## Status
Accepted

## Date
2026-03 (backend V7)

## Context
Ticket numbers need to be human-friendly and collision-free (`TKT-0000001` style, sequential) for support-agent use, while internal foreign keys and joins benefit from a simple numeric primary key. Exposing the raw numeric DB primary key externally (in URLs, storage object keys, or API responses used by other repositories) also creates an enumeration/identity-leak risk.

## Decision
- `ticket_no_seq` (PostgreSQL sequence) generates collision-free sequential ticket numbers in the `TKT-%07d` format, used by both `TicketService` and `EmailIngressServiceImpl.createTicketFromEvent`.
- The numeric database primary key may remain for internal use (FKs, joins), but a **public UUID is preferred for external or storage-facing identity** (e.g. object storage key components, cross-repository references) wherever a new identifier is introduced.
- Object storage keys follow `tickets/{ticketId}/{uuid}_{sanitizedName}` — one bucket per env/app, prefixed by ticket, not a bucket per ticket.

## Alternatives Considered
- **Expose numeric PK everywhere** — rejected: creates predictable, enumerable external identifiers.
- **UUID-only ticket identity (no human-readable ticket number)** — rejected: support agents need a short, sequential, speakable identifier.

## Consequences
- New cross-repository identifiers (mobile deep links, AI service references, storage keys) should default to UUID, not the raw numeric PK, unless there is a specific reason to use the sequence-based `ticketNo`.
- This is enforced as a review invariant — see [agents/CODE-REVIEW.md](../agents/CODE-REVIEW.md).

## Affected Repositories
- caseflow-be
- caseflow-fe (ticket number display)
- caseflow-mobile (ticket number display)

## Related Contracts / Features / Tasks
- [repos/backend/current-state.md](../repos/backend/current-state.md)
- [contracts/api/README.md](../contracts/api/README.md)
