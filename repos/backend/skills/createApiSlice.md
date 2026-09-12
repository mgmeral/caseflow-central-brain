Follow:
- .ai/backend-rules.md
- .ai/architecture.md
- .ai/prompts/create-dto.md
- .ai/prompts/create-controller.md

Skill:
Create only the API slice for an already existing service/domain.

Use this for:
- exposing existing business logic over REST
- adding endpoint layer after service is ready

Steps:
1. Define request/response DTOs
2. Create controller
3. Map endpoints to service methods
4. Add validation
5. Add response status mapping

Rules:
- Do not implement business logic here
- Assume service already exists
- Do not access repository directly
- Keep endpoints task-oriented and clean

Expected output:
- DTOs
- controller
- endpoint list

Input format:
- module name
- service methods
- endpoint requirements