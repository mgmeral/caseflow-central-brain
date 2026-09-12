# Transfer Workflow Specification

Fields:
- id (Long)
- ticketId (Long)
- fromGroupId (Long)
- toGroupId (Long)
- transferredBy (Long)
- transferredAt (Instant)
- reason (String)

Rules:
- transfer must update assigned group
- transfer does not automatically change ticket status
- transfer must be logged in history