Follow:
- .ai/backend-rules.md
- .ai/architecture.md

Task:
Create repository layer for the given entity/document.

Output:
- repository interface only
- custom query methods if needed

Rules:
- Do not create service
- Do not create controller
- Keep repository focused on persistence
- Use Spring Data JPA for relational entities
- Use Spring Data Mongo for document entities

Input format:
- module name
- entity/document name
- key query requirements