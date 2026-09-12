Follow:
- .ai/backend-rules.md
- .ai/architecture.md
- .ai/email-flow.md
- .ai/prompts/create-document.md
- .ai/prompts/create-repository.md
- .ai/prompts/create-service.md
- .ai/prompts/create-tests.md

Skill:
Create a document-oriented MongoDB module.

Use this for:
- email documents
- processing logs
- raw payload stores
- flexible metadata documents

Steps:
1. Create Mongo document
2. Define indexes
3. Create Mongo repository
4. Create service layer
5. Add helper enums/models if needed
6. Create tests

Rules:
- Do not model raw documents like relational entities
- Keep document schema flexible but intentional
- Include only fields needed for retrieval, processing, and traceability
- Do not create external controller unless explicitly requested
- Separate raw document model from business ticket model

Expected output:
- document class
- indexes
- repository
- service
- tests

Input format:
- module name
- document name
- fields
- indexing requirements
- service use cases