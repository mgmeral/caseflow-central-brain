Follow:
- .ai/backend-rules.md
- .ai/architecture.md
- .ai/ticket-rules.md
- .ai/email-flow.md
- .ai/prompts/create-workflow.md
- .ai/prompts/create-dto.md
- .ai/prompts/create-tests.md

Skill:
Create a workflow-oriented module.

Use this for:
- assignment
- reassignment
- transfer
- duplicate handling
- merge
- ticket state transition
- thread resolution
- customer matching

Steps:
1. Define workflow actions
2. Define request/command models if needed
3. Create service layer
4. Add validations and business rules
5. Add history integration
6. Add tests
7. Add controller only if explicitly requested

Rules:
- Workflow modules are process-centric
- Do not force CRUD structure
- Record critical changes into history
- Validate preconditions and invariants
- Keep orchestration in service layer
- Avoid repository leakage to controller

Expected output:
- workflow service(s)
- request/command models
- exception points
- history touchpoints
- tests
- optional controller

Input format:
- workflow name
- related modules
- business rules
- actions
- API exposure needed or not