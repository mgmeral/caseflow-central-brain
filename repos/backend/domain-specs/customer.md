# Customer Domain Specification

Fields:
- id (Long)
- name (String)
- code (String, optional)
- isActive (Boolean)
- createdAt (Instant)
- updatedAt (Instant)

Relationships:
- Customer → Contact (one-to-many)

Rules:
- customer name required
- customer must be active for new ticket creation