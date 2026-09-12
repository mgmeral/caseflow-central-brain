# Assignment Workflow Specification

Fields:
- id (Long)
- ticketId (Long)
- assignedUserId (Long)
- assignedGroupId (Long)
- assignedAt (Instant)
- unassignedAt (Instant, nullable)
- assignedBy (Long)

Rules:
- only one active assignment per ticket
- reassignment must close previous assignment
- assignment changes must be logged in history