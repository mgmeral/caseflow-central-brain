# CaseFlow Central Brain

CaseFlow is a multi-application product that coordinates workflow across backend, frontend, AI, and mobile systems.

`caseflow-central-brain` is the shared source of truth for cross-repository architecture, product context, contracts, decisions, feature definitions, task tracking, and AI agent guidance.

## Application Repositories

- `caseflow-be` — Backend
- `caseflow-fe` — Frontend
- `caseflow-ai-service` — AI Service
- `caseflow-mobil` — Mobile Application

Each app repository keeps only a one-line pointer `README.md` back into this repository's `repos/<name>/` — see [Single Source of Truth](#single-source-of-truth-for-documentation) below. No other `.md` files should exist in an app repository.

## Why this repository exists

Use this repository to keep cross-repository knowledge in one place and avoid conflicting assumptions between teams and AI agents — including multiple agents working in parallel across `caseflow-be`, `caseflow-fe`, `caseflow-ai-service`, and `caseflow-mobil` at the same time.

## How developers and AI agents should use this repository

- Read relevant docs before implementing cross-repository changes.
- Update the matching source-of-truth files when behavior or agreements change.
- Link decisions, contracts, and features to concrete implementation tasks in the app repositories.

## Source of Truth Rules

1. Architecture decisions belong in `docs/architecture`.
2. Product/domain definitions belong in `docs/product`.
3. API and event contracts belong in `contracts`.
4. Architecture Decision Records (ADRs) belong in `decisions`.
5. Cross-repository features belong in `features`.
6. Current implementation work belongs in `tasks`.
7. AI behavior and repository-specific instructions belong in `agents`.
8. Repository-internal documentation (module maps, coding rules, domain field specs, scaffolding prompts/skills, deployment/CI guides, that repo's own README content) belongs in `repos/<name>/` — see below.

## Single Source of Truth for Documentation

This repository holds **every** `.md` file for the CaseFlow product, including content that is internal to one repository (not just cross-repository agreements). This is deliberate: with multiple agents working across repositories at once, one shared, always-current documentation tree avoids drift and conflicting copies.

- `repos/backend/` — everything formerly under `caseflow-be/.ai/` and `caseflow-be/docs/` (architecture, module map, coding/backend rules, ticket/storage/email rules, domain specs, scaffolding prompts and skills, current state, remaining issues, API docs, CI/CD and deployment guides).
- `repos/frontend/` — everything formerly under `caseflow-fe/README.md` and `caseflow-fe/docs/`.
- `repos/ai-service/` — everything formerly `caseflow-ai-service/README.md`.
- `repos/mobile/` — everything formerly `caseflow-mobil/README.md`.

The top-level `docs/`, `contracts/`, `decisions/`, `features/`, `agents/`, and `tasks/` folders stay reserved for genuinely cross-repository content, per the Source of Truth Rules above; they link into `repos/<name>/` for implementation-level detail rather than duplicating it.

When an app repository's behavior changes in a way that affects its own internal docs, update the matching file under `repos/<name>/` here — do not recreate a local `.md` file in the app repository (its `README.md` pointer excepted).

## Contribution Rules

- Do not store application source code in this repository.
- Do not silently change shared contracts; document impact and affected repositories.
- Use placeholders and `TODO: Define this decision.` when a decision is unknown.
- Keep `docs/`, `contracts/`, `decisions/`, `features/`, `agents/`, and `tasks/` focused on cross-repository context; put repository-internal detail under `repos/<name>/` instead of inventing a new top-level folder.
