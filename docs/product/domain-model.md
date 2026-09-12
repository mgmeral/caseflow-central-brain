# Domain Model

## Purpose
This document defines shared domain language and business entities used across repositories.

## What belongs here
Canonical terms, entities, relationships, and lifecycle definitions at product/domain level.

## Who should use this
All repository contributors and AI agents producing cross-repository behavior.

## What should NOT be stored here
Database schemas, ORM mappings, or implementation-specific model classes — see [repos/backend/domain-specs/](../../repos/backend/domain-specs/) for full per-entity field-level specs.

## Domain glossary
- **Ticket** — the central aggregate. Not equal to an email; a ticket may have multiple linked email documents.
- **Customer** — the external organization/account a ticket belongs to. Owns email routing configuration.
- **Contact** — a person at a customer. Does **not** drive email routing (see [ADR-0001](../../decisions/0001-customer-based-email-routing.md)).
- **User** — an internal actor (agent/admin/supervisor/viewer) who can be assigned tickets.
- **Group** — an internal queue/team a ticket or user can belong to.
- **Assignment** — the current owning user/group of a ticket. Only one active assignment per ticket.
- **Transfer** — a change of ticket ownership between groups/users; does not automatically change ticket status.
- **Note** — an internal comment on a ticket. Not an email reply.
- **EmailDocument / EmailIngressEvent / OutboundEmailDispatch** — the email platform's record of inbound/outbound messages linked to a ticket.
- **Attachment** — binary content (in object storage) with metadata owned by the ticket.

Full field-level specs: [repos/backend/domain-specs/](../../repos/backend/domain-specs/) (ticket, customer, contact, user, group, note, assignment, transfer, attachment, email-documents).

## Core entities
See glossary above. Central aggregate: **Ticket**. Everything else either belongs to a ticket (notes, attachments, assignments, transfers, linked emails) or provides context for one (customer, contact, user, group).

## Entity relationships
```
Customer 1──* Contact
Customer 1──* CustomerEmailRoutingRule
Ticket   *──1 Customer
Ticket   1──* Assignment (only one active at a time)
Ticket   1──* Transfer
Ticket   1──* Note
Ticket   1──* Attachment (metadata; binary in object storage)
Ticket   1──* EmailDocument (via EmailIngressEvent / OutboundEmailDispatch)
User     *──* Group
Assignment *──1 User or Group (assignee target)
```

## Lifecycle/state concepts
Ticket status: `NEW → TRIAGED → ASSIGNED → IN_PROGRESS → WAITING_CUSTOMER → RESOLVED → CLOSED → REOPENED`. Reopen is allowed only from `CLOSED` or `RESOLVED`. Invalid transitions must be rejected — full rules in [repos/backend/ticket-rules.md](../../repos/backend/ticket-rules.md).

Email ingress lifecycle: `RECEIVED → PROCESSING → PROCESSED | FAILED | QUARANTINED`. Outbound dispatch lifecycle: `PENDING → SENDING → SENT | FAILED | PERMANENTLY_FAILED`. Full flow: [repos/backend/email-flow.md](../../repos/backend/email-flow.md).
