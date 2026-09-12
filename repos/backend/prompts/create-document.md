Follow:
- .ai/backend-rules.md
- .ai/architecture.md
- .ai/email-flow.md

Task:
Create MongoDB document model and related repository/service parts for the given document feature.

Output:
- document class
- indexes
- repository
- supporting enums if needed

Rules:
- Use Mongo annotations
- Design for flexible but controlled structure
- Include metadata needed for lookup and processing
- Avoid mixing business ticket state into raw document model

Input format:
- module name
- document name
- required fields
- indexing requirements