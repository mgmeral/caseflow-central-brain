Ticket lifecycle:

NEW → TRIAGED → ASSIGNED → IN_PROGRESS → WAITING_CUSTOMER → RESOLVED → CLOSED → REOPENED

Rules:
- Only one active assignee per ticket
- Assignment changes must be recorded in history
- Invalid state transitions must be rejected
- Reopen allowed only from CLOSED or RESOLVED
- Transfer does not automatically change status
- Ticket must always belong to a group or user

Fields:
- ticketNo (unique)
- subject
- status
- priority
- assignedUserId
- assignedGroupId
- createdAt
- updatedAt
- closedAt

Important:
- Ticket is not equal to email
- Ticket may have multiple email records