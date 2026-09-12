Follow:
- .ai/backend-rules.md
- .ai/architecture.md
- .ai/ticket-rules.md
- .ai/email-flow.md

Task:
Create a workflow-oriented service/module.

Examples:
- assignment
- reassignment
- transfer
- duplicate handling
- state transition
- thread resolution

Output:
- service classes
- command/request models if needed
- business flow steps
- validations
- history integration
- exceptions

Rules:
- Workflow modules are process-driven, not simple CRUD
- Enforce business rules strictly
- Record critical actions in history
- Keep workflow logic outside controllers
- If state changes happen, validate transitions

Input format:
- workflow name
- related modules
- business rules
- expected actions