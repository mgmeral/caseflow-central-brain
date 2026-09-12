Follow:
- .ai/backend-rules.md
- .ai/architecture.md
- .ai/storage-rules.md

Task:
Create provider-agnostic object storage abstraction.

Output:
- ObjectStorageService interface
- command/response models
- metadata models if needed
- implementation skeletons if requested

Rules:
- Business layer must not depend on vendor SDK directly
- Support future implementations such as AWS S3, MinIO, Azure Blob, Local
- Keep interface cohesive
- Do not hardcode vendor logic into core services

Input format:
- required operations
- metadata requirements
- provider list