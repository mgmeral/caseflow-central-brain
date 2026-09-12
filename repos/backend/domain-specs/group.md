# Group Domain Specification

Fields:
- id (Long)
- name (String)
- type (enum)
- isActive (Boolean)
- createdAt (Instant)

Enum GroupType:
- TRADE
- OPERATIONS
- SUPPORT

Relationships:
- Group → User (one-to-many or many-to-many)

Rules:
- group name must be unique
- group used for assignment and transfer