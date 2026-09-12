Follow:
- .ai/backend-rules.md
- .ai/architecture.md
- .ai/ticket-rules.md
- .ai/prompts/create-entity.md
- .ai/prompts/create-repository.md
- .ai/prompts/create-dto.md
- .ai/prompts/create-service.md
- .ai/prompts/create-controller.md
- .ai/prompts/create-tests.md

Skill:
Create an aggregate-style domain module with business rules.

Use this for:
- ticket
- customer case aggregate
- domains with lifecycle and ownership

Steps:
1. Create aggregate root entity
2. Create related enums
3. Create repository
4. Create DTOs
5. Create service layer with business rules
6. Create controller
7. Add history/state validation hooks if relevant
8. Create tests

Rules:
- Aggregate is not simple CRUD
- Enforce lifecycle/state rules
- Do not expose entity directly
- Use service methods that reflect business actions
- If assignment/history is relevant, integrate through service design
- Keep invariants in service/domain boundaries

Expected output:
- aggregate model
- enums
- repository
- dto set
- service
- controller
- tests

Input format:
- module name
- aggregate name
- fields
- lifecycle rules
- business actions
- API needs