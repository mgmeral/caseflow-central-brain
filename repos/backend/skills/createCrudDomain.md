Follow:
- .ai/backend-rules.md
- .ai/architecture.md
- .ai/prompts/create-entity.md
- .ai/prompts/create-repository.md
- .ai/prompts/create-dto.md
- .ai/prompts/create-service.md
- .ai/prompts/create-controller.md
- .ai/prompts/create-tests.md

Skill:
Create a CRUD-oriented domain module.

Use this for:
- customer
- group
- user
- contact
- simple master data domains

Steps:
1. Create entity model
2. Create repository
3. Create DTOs
4. Create service layer
5. Create controller
6. Create unit tests

Rules:
- Keep controller thin
- Put business logic in service
- Use DTOs for API contracts
- Add validation annotations
- Add only required CRUD operations
- Do not add workflow/state machine unless explicitly requested

Expected output:
- package/file list
- generated code for entity/repository/dto/service/controller
- test skeletons

Input format:
- module name
- entity name
- fields
- required CRUD operations
- validation rules