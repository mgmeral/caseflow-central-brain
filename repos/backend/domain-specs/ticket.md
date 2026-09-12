# Ticket Domain Specification

Fields:
- id (Long)
- ticketNo (String, unique)
- subject (String)
- description (String)
- status (enum)
- priority (enum)
- assignedUserId (Long, nullable)
- assignedGroupId (Long, nullable)
- customerId (Long, nullable)
- createdAt (Instant)
- updatedAt (Instant)
- closedAt (Instant, nullable)

Enums:

TicketStatus:
- NEW
- TRIAGED
- ASSIGNED
- IN_PROGRESS
- WAITING_CUSTOMER
- RESOLVED
- CLOSED
- REOPENED

TicketPriority:
- LOW
- MEDIUM
- HIGH
- CRITICAL

Relationships:
- Ticket → Customer (many-to-one)
- Ticket → User (assignee)
- Ticket → Group

Rules:
- ticketNo must be unique
- only one active assignee
- status transitions must be validated
- closedAt must be set when status = CLOSED