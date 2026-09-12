Follow:
- .ai/backend-rules.md
- .ai/architecture.md

Task:
Create request and response DTOs for the given feature.

Output:
- request DTOs
- response DTOs
- validation annotations
- optional mapper suggestions

Rules:
- Do not expose entities directly
- Use clear naming: CreateXRequest, UpdateXRequest, XResponse
- Keep DTOs use-case specific
- Avoid generic god DTOs

Input format:
- module name
- feature name
- operations (create/update/detail/list)
- fields per operation