# Contact Domain Specification

Fields:
- id (Long)
- customerId (Long)
- email (String)
- name (String, optional)
- isPrimary (Boolean)
- isActive (Boolean)
- createdAt (Instant)

Indexes:
- email

Rules:
- email required
- email used for customer matching
- one primary contact per customer