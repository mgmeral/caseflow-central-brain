# Email Document Specification

Storage: MongoDB

Fields:
- id (String)
- messageId (String, unique)
- threadKey (String)
- inReplyTo (String)
- references (List<String>)
- subject (String)
- normalizedSubject (String)
- from (String)
- to (List<String>)
- cc (List<String>)
- bcc (List<String>)
- htmlBody (String)
- textBody (String)
- attachments (List<AttachmentMetadata>)
- receivedAt (Instant)
- parsedAt (Instant)
- ticketId (Long, nullable)

AttachmentMetadata:
- fileName (String)
- objectKey (String)
- contentType (String)
- size (Long)

Indexes:
- messageId (unique)
- threadKey
- ticketId
- receivedAt

Rules:
- messageId must be unique
- thread resolution must use messageId, inReplyTo, references
- do not rely only on subject
- email may or may not be linked to a ticket