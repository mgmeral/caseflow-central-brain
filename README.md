# CaseFlow Central Brain

CaseFlow is a multi-application product that coordinates workflow across backend, frontend, AI, and mobile systems.

`caseflow-central-brain` is the shared source of truth for cross-repository architecture, product context, contracts, decisions, feature definitions, task tracking, and AI agent guidance.

## Application Repositories

- `caseflow-be` — Backend
- `caseflow-fe` — Frontend
- `caseflow-ai-service` — AI Service
- `caseflow-mobil` — Mobile Application

Each app repository keeps only a one-line pointer `README.md` back into this repository's `repos/<name>/` — see [Single Source of Truth](#single-source-of-truth-for-documentation) below. No other `.md` files should exist in an app repository.

**Start here for a fast cross-repo orientation:** [repos/repository-context.md](repos/repository-context.md) (AI-agent-optimized summary), [repos/repository-map.md](repos/repository-map.md) (per-repo facts), [repos/dependency-map.md](repos/dependency-map.md) (who depends on whom), [repos/integration-map.md](repos/integration-map.md) (verified cross-repo endpoints/events), and [agents/CROSS-REPOSITORY-CHANGE.md](agents/CROSS-REPOSITORY-CHANGE.md) (the workflow for any change spanning more than one repository).

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
6. Current implementation work belongs in `tasks` (see `tasks/templates/` for the standard task shape).
7. AI behavior and repository-specific instructions belong in `agents` (including `agents/AGENT-OWNERSHIP.md` — who's responsible for which repository).
8. Repository-internal documentation (module maps, coding rules, domain field specs, scaffolding prompts/skills, deployment/CI guides, that repo's own README content) belongs in `repos/<name>/` — see below.
9. How a request gets turned into a task (analysis/alignment/cross-repository/contract-change procedures) belongs in `skills`.
10. Task lifecycle, dependency rules, the machine-readable task-graph format, and agent handoff format belong in `workflows`.

## Single Source of Truth for Documentation

This repository holds **every** `.md` file for the CaseFlow product, including content that is internal to one repository (not just cross-repository agreements). This is deliberate: with multiple agents working across repositories at once, one shared, always-current documentation tree avoids drift and conflicting copies.

- `repos/backend/` — everything formerly under `caseflow-be/.ai/` and `caseflow-be/docs/` (architecture, module map, coding/backend rules, ticket/storage/email rules, domain specs, scaffolding prompts and skills, current state, remaining issues, API docs, CI/CD and deployment guides).
- `repos/frontend/` — everything formerly under `caseflow-fe/README.md` and `caseflow-fe/docs/`.
- `repos/ai-service/` — everything formerly `caseflow-ai-service/README.md`.
- `repos/mobile/` — everything formerly `caseflow-mobil/README.md`.

The top-level `docs/`, `contracts/`, `decisions/`, `features/`, `agents/`, and `tasks/` folders stay reserved for genuinely cross-repository content, per the Source of Truth Rules above; they link into `repos/<name>/` for implementation-level detail rather than duplicating it.

When an app repository's behavior changes in a way that affects its own internal docs, update the matching file under `repos/<name>/` here — do not recreate a local `.md` file in the app repository (its `README.md` pointer excepted).

## AI Agent Orchestration (Phase 1 — manual)

This repository also runs a Phase 1 (manual) task-planning system for coordinating work across the four application repositories: a request is analyzed into a task graph ([skills/](skills/)), tracked through a defined lifecycle ([workflows/](workflows/), [tasks/](tasks/)), and handed off to a human-selected agent per default ownership ([agents/AGENT-OWNERSHIP.md](agents/AGENT-OWNERSHIP.md)). **Phase 1 is intentionally manual** — nothing in this repository automatically executes Claude, Copilot, or Codex; see [workflows/PHASE-2-AUTOMATION.md](workflows/PHASE-2-AUTOMATION.md) for the (unimplemented) design of what could eventually change that.

## Contribution Rules

- Do not store application source code in this repository.
- Do not silently change shared contracts; document impact and affected repositories.
- Use placeholders and `TODO: Define this decision.` when a decision is unknown.
- Keep `docs/`, `contracts/`, `decisions/`, `features/`, `agents/`, and `tasks/` focused on cross-repository context; put repository-internal detail under `repos/<name>/` instead of inventing a new top-level folder.
- Do not automatically trigger an external agent/provider from within this repository during Phase 1 — every handoff is human-mediated (see [workflows/AGENT-HANDOFF.md](workflows/AGENT-HANDOFF.md)).
