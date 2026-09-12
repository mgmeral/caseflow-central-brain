# History Specification

Fields:
- id (Long)
- ticketId (Long)
- actionType (String)
- performedBy (Long)
- performedAt (Instant)
- details (String or JSON)

Examples of actionType:
- CREATED
- ASSIGNED
- REASSIGNED
- TRANSFERRED
- STATUS_CHANGED
- NOTE_ADDED

Rules:
- every critical action must create history record
- history must be immutable