# CaseFlow Central Brain

CaseFlow is a multi-application product that coordinates workflow across backend, frontend, AI, and mobile systems.

`caseflow-central-brain` is the shared source of truth for cross-repository architecture, product context, contracts, decisions, feature definitions, task tracking, and AI agent guidance.

## Application Repositories

- `caseflow-be` — Backend
- `caseflow-fe` — Frontend
- `caseflow-ai-service` — AI Service
- `caseflow-mobile` — Mobile Application

## Why this repository exists

Use this repository to keep cross-repository knowledge in one place and avoid conflicting assumptions between teams and AI agents.

## How developers and AI agents should use this repository

- Read relevant docs before implementing cross-repository changes.
- Update the matching source-of-truth files when behavior or agreements change.
- Link decisions, contracts, and features to concrete implementation tasks in the app repositories.

## Source of Truth Rules

1. Architecture decisions belong in `docs/architecture`.
2. Product/domain definitions belong in `docs/product`.
3. API and event contracts belong in `contracts`.
4. Architectural decisions belong in `decisions`.
5. Cross-repository features belong in `features`.
6. Current implementation work belongs in `tasks`.
7. AI behavior and repository-specific instructions belong in `agents`.

## Contribution Rules

- Do not store application source code in this repository.
- Do not silently change shared contracts; document impact and affected repositories.
- Use placeholders and `TODO: Define this decision.` when a decision is unknown.
- Keep files focused on cross-repository context, not implementation internals.
