# Contracts

Contracts are shared agreements between CaseFlow services and applications.

## What belongs here
Definitions that must remain consistent across repositories:
- API contracts
- Event contracts
- Shared schemas

## Who should use this
Contributors and AI agents changing interfaces between `caseflow-be`, `caseflow-fe`, `caseflow-ai-service`, and `caseflow-mobile`.

## What should NOT be stored here
Implementation code, service internals, or undocumented breaking changes.

## Contract governance rules
- Do not silently change a shared contract.
- Breaking changes must be explicitly documented.
- Contract changes must identify affected repositories.
- API contracts should eventually use OpenAPI where appropriate.
- Event contracts must document producer, consumer, topic/event name, payload, and compatibility requirements.
- Shared schemas must have clear ownership.
