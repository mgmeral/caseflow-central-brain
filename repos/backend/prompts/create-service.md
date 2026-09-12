Follow:
- .ai/backend-rules.md
- .ai/architecture.md
- .ai/ticket-rules.md
- .ai/email-flow.md
- .ai/storage-rules.md

Task:
Create service layer for the given feature.

Output:
- service interface if useful
- service implementation
- method signatures
- transactional boundaries
- validation and business rules
- dependencies required

Rules:
- No business logic in controller
- Service must contain business rules
- Use constructor injection
- Keep methods cohesive
- If history is relevant, include history recording
- If workflow-related, enforce business constraints
- Do not generate controller unless explicitly requested

Input format:
- module name
- feature name
- related entities/documents
- business rules