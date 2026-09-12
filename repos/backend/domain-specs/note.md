# Note Domain Specification

Fields:
- id (Long)
- ticketId (Long)
- content (String)
- createdBy (Long)
- createdAt (Instant)
- type (enum)

Enum NoteType:
- INFO
- INVESTIGATION
- ESCALATION
- INTERNAL

Relationships:
- Note → Ticket (many-to-one)
- Note → User (createdBy)

Rules:
- notes are internal only
- notes are not visible to customers