Follow:
- .ai/backend-rules.md
- .ai/architecture.md
- .ai/ticket-rules.md

Task:
Create a domain entity for the given module.

Output:
- entity class
- related enums if needed
- JPA annotations if relational
- constraints
- indexes suggestion in comments if useful

Rules:
- Do not create controller
- Do not create service
- Do not create repository
- Keep entity focused
- Use proper naming
- Include audit fields only if relevant

Input format:
- module name
- entity name
- persistence type (jpa or mongo)
- required fields
- relationships