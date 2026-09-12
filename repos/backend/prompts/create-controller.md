Follow:
- .ai/backend-rules.md
- .ai/architecture.md

Task:
Create REST controller for the given feature.

Output:
- controller class
- endpoints
- request/response DTO usage
- validation annotations
- response status mapping

Rules:
- Controller must only orchestrate
- No business logic in controller
- Delegate to service layer
- Use clear endpoint naming
- Use proper HTTP methods
- Do not access repository directly

Input format:
- module name
- feature name
- endpoint list
- dto names
- service methods to call