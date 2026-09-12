# Attachment Metadata Specification

Fields:
- id (Long)
- ticketId (Long, nullable)
- emailId (String, nullable)
- fileName (String)
- objectKey (String)
- contentType (String)
- size (Long)
- uploadedAt (Instant)

Rules:
- binary data stored in object storage
- metadata stored in database
- objectKey must be unique