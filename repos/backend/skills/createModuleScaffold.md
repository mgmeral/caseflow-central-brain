Follow:
- .ai/backend-rules.md
- .ai/architecture.md

Skill:
Create module scaffold only.

Use this when:
- starting a new module
- defining package structure before implementation

Steps:
1. Create package structure
2. Suggest file/class list
3. Mark which parts are mandatory vs optional
4. Explain boundaries briefly

Rules:
- Do not generate full implementation unless explicitly requested
- Respect modular monolith boundaries
- Keep module cohesive
- Separate domain, api, repository, service concerns appropriately

Expected output:
- package tree
- class list
- implementation order

Input format:
- module name
- module purpose
- persistence type
- api exposure