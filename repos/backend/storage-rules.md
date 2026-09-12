Use provider-agnostic object storage.

Rules:
- Do not depend directly on AWS SDK in business layer
- Use abstraction: ObjectStorageService
- Support multiple providers:
    - AWS S3
    - MinIO
    - Azure Blob (future)
    - Local storage (dev)

Attachment rules:
- Store binary in object storage
- Store metadata in database
- Metadata includes:
    - objectKey
    - fileName
    - size
    - contentType
    - checksum

Important:
- Do not store large binary in DB
- Use presigned URL for downloads (future-ready)