Ticket lifecycle (verified against `workflow/state/TicketStateMachineService.java`, 2026-09-12):

Statuses: `NEW, TRIAGED, ASSIGNED, IN_PROGRESS, WAITING_CUSTOMER, RESOLVED, CLOSED, REOPENED`

## Manual transition matrix (enforced server-side; invalid transitions throw `InvalidTicketStateException`)

```
NEW              → TRIAGED, ASSIGNED, IN_PROGRESS, CLOSED
TRIAGED          → ASSIGNED, IN_PROGRESS, WAITING_CUSTOMER, RESOLVED, CLOSED
ASSIGNED         → IN_PROGRESS, WAITING_CUSTOMER, RESOLVED, CLOSED
IN_PROGRESS      → WAITING_CUSTOMER, RESOLVED, ASSIGNED, CLOSED
WAITING_CUSTOMER → IN_PROGRESS, RESOLVED, CLOSED
RESOLVED         → IN_PROGRESS, REOPENED, CLOSED
CLOSED           → REOPENED
REOPENED         → ASSIGNED, IN_PROGRESS, WAITING_CUSTOMER, RESOLVED, CLOSED
```

This is **not** a purely linear lifecycle — do not assume every status only moves "forward" one step. `GET /api/tickets/{id}/transitions` returns the allowed next states for the current status; clients (web FE) fetch this before rendering status-change actions rather than hardcoding the matrix client-side.

## System-triggered transitions (applied automatically by `TicketSystemTransitionService`, not a user action)
- Inbound customer email while `WAITING_CUSTOMER` → `IN_PROGRESS`
- Inbound customer email while `RESOLVED` → `REOPENED`
- Inbound customer email while `CLOSED` → `REOPENED`
- Outbound agent reply from any active work state (`NEW, TRIAGED, ASSIGNED, IN_PROGRESS, REOPENED`) → `WAITING_CUSTOMER`

## Rules
- Only one active assignee per ticket (enforced at the workflow layer, not here).
- Assignment/transfer/status-change actions must be recorded in history (`History` entity).
- Invalid state transitions must be rejected (see matrix above) — this is enforced by `TicketStateMachineService`, not by ad-hoc checks in controllers.
- Transfer does not automatically change status.
- Ticket must always belong to a group or user once assigned (assignment is a separate concept from creation — a ticket can exist unassigned, e.g. in the admin/unassigned pool).
- Ticket merge is a supported concept: `parentTicketId`, `mergedAt`, `mergedBy` fields exist on `Ticket` — `UNKNOWN: exact merge workflow/API surface`, not verified in per-endpoint detail.
- Optimistic locking (`@Version`) protects concurrent updates to the same ticket.

## Fields (non-exhaustive; see `ticket/domain/Ticket.java` for the authoritative source)
- `id` (internal numeric PK), `publicId` (UUID — the preferred external/cross-repo identifier per [ADR-0003](../../decisions/0003-sequential-ticket-numbers-public-uuid.md)), `ticketNo` (unique, sequential, human-facing — `TKT-0000001`)
- `subject`, `description`
- `status`, `priority` (`LOW | MEDIUM | HIGH | CRITICAL`), `channel` (defaults `EMAIL`)
- `assignedUserId`, `assignedGroupId`, `customerId`
- `firstResponseDueAt`, `resolutionDueAt`, `firstResponseRespondedAt`, `resolvedAt` (SLA fields — stamped from the applicable `SlaPolicyConfig` at creation)
- `parentTicketId`, `mergedAt`, `mergedBy` (merge support)
- `createdAt`, `updatedAt`, `statusChangedAt`, `closedAt`, `resolvedBy`, `closedBy`, `createdBy`, `updatedBy`
- Tags via `TicketTag` (many-to-many), Jira link via `TicketJiraLink` (optional one-to-one) — see [module-map.md](module-map.md#modules-added-since-the-original-map-verified-2026-09-12)

## Important
- Ticket is not equal to email — a ticket may have multiple linked email records ([email-flow.md](email-flow.md)).
- Ticket is the central aggregate that tags, SLA state, Jira links, notes, attachments, and workflow (assignment/transfer) all attach to.
