# Shared Schemas

## Purpose
Defines canonical shared schemas used by multiple CaseFlow repositories.

## What belongs here
Reusable schema definitions, ownership, change policy, and compatibility notes.

## Who should use this
Contributors and AI agents introducing or modifying cross-repository data structures.

## What should NOT be stored here
Repository-private model definitions or one-off temporary payload examples.

## Ownership and versioning
`caseflow-be` owns the canonical shape and serialization of every shared enum and DTO; other repositories must treat these as read-only contracts and never redefine them independently. Enums serialize as their string name (Spring Boot default).

### Canonical enums (owned by caseflow-be)
```
TicketStatus        : NEW | TRIAGED | ASSIGNED | IN_PROGRESS | WAITING_CUSTOMER | RESOLVED | CLOSED | REOPENED
TicketPriority      : LOW | MEDIUM | HIGH | CRITICAL
TicketScope         : ALL | OWN_GROUPS | OWN_AND_OWN_GROUPS | ASSIGNED_ONLY
NoteType            : INFO | INVESTIGATION | ESCALATION | INTERNAL
GroupType           : TRADE | OPERATIONS | SUPPORT
IngressEventStatus  : RECEIVED | PROCESSING | PROCESSED | FAILED | QUARANTINED
DispatchStatus      : PENDING | SENDING | SENT | FAILED | PERMANENTLY_FAILED
ProviderType        : SMTP_RELAY | SENDGRID | MAILGUN
InboundMode         : WEBHOOK | IMAP_POLL | MANUAL
OutboundMode        : SMTP | API
SenderMatchType     : EXACT_EMAIL | DOMAIN
UnknownSenderPolicy : MANUAL_REVIEW | IGNORE | REJECT
```
Full field-level DTO shapes: [repos/backend/frontend-contract.md](../../repos/backend/frontend-contract.md#key-response-shapes) and [repos/backend/api-endpoints.md](../../repos/backend/api-endpoints.md).

**Corrected 2026-09-12**: `caseflow-be` was previously believed to carry a generated `docs/api-models.ts` file, but source inspection confirms `caseflow-be` has **no `docs/` directory at all** — its README explicitly delegates all documentation to this repository. `TODO: Verify` where (if anywhere) a generated TypeScript-types-from-DTOs artifact currently lives, or whether the frontend hand-maintains its own types in `src/types/*.types.ts` with no generation step (the frontend inspection pass found hand-written types, not generated ones — `TODO: Verify` this is the current, intended state rather than a lapsed generation step).

The enum list above predates several backend additions confirmed in the 2026-09-12 pass (SLA, tags, Jira, automation, notification-channel, mail-template enums) — `TODO: Define` those canonical enum names/values here once the corresponding controllers get a full contract-documentation pass (see [repos/integration-map.md](../../repos/integration-map.md) for what's confirmed to exist vs. `TODO: Verify`).

### Change policy
Any change to a canonical enum's values or a shared DTO's field shape is a contract change — follow [contracts/api/README.md](../api/README.md#contract-governance-rules): update the affected doc, identify affected repositories, and add an ADR if the change reflects a real decision (e.g. removing a field because a routing strategy changed — see [ADR-0001](../../decisions/0001-customer-based-email-routing.md) for a precedent).
