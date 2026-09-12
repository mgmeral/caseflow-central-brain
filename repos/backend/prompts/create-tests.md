Follow:
- .ai/backend-rules.md
- .ai/ticket-rules.md
- .ai/email-flow.md

Task:
Create tests for the given feature.

Output:
- unit tests
- integration test suggestions if useful
- happy path and failure cases

Rules:
- Cover business rules, not only getters/setters
- Include negative cases
- Mock external dependencies
- Verify history/audit behavior when relevant
- Verify invalid state transitions when relevant

Input format:
- module name
- feature name
- methods to test
- business rules to verify