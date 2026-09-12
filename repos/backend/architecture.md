Project uses modular monolith with feature-based packages.

Modules:
- ticket → core ticket logic
- email → email ingestion, parsing, thread resolution
- identity → user, role, group
- customer → customer and contact mapping
- workflow → assignment, transfer, state transitions
- note → internal notes
- storage → object storage abstraction

Rules:
- Each module contains its own domain, repository, service, api, dto
- Do not mix modules
- Shared utilities go to common package

Important:
- Ticket is central aggregate
- Workflow is process-driven (not pure CRUD)
- Email module handles external integrations (SMTP/IMAP)