# User Domain Specification

Fields:
- id (Long)
- username (String, unique)
- email (String)
- fullName (String)
- isActive (Boolean)
- createdAt (Instant)
- lastLoginAt (Instant, nullable)

Relationships:
- User → Group (many-to-many or many-to-one depending on design)

Rules:
- username must be unique
- inactive users cannot be assigned to tickets