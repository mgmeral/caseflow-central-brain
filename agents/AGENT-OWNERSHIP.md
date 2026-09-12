# Agent Ownership

## Purpose
Defines which agent provider is responsible for work in each CaseFlow repository, for Phase 1 of the AI Agent Orchestration system ([workflows/PHASE-2-AUTOMATION.md](../workflows/PHASE-2-AUTOMATION.md) describes how this could eventually become automated — it is not automated today).

## What this is NOT
This is an **ownership convention**, not an automatic execution mechanism. Central Brain does not invoke Claude, Copilot, or Codex on anyone's behalf in Phase 1. A human reads a task file, follows [../workflows/AGENT-HANDOFF.md](../workflows/AGENT-HANDOFF.md), and manually starts the appropriate agent in the appropriate repository.

## Default ownership

| Repository | Agent | Notes |
|---|---|---|
| `caseflow-be` | Claude | Backend is the dependency root — most cross-repository tasks have a `caseflow-be` sub-task that other sub-tasks depend on. |
| `caseflow-fe` | GitHub Copilot / VS Code | |
| `caseflow-mobil` | GitHub Copilot / VS Code | |
| `caseflow-ai-service` | Codex or Claude | Either is acceptable; a specific task may name one explicitly if it matters (e.g., continuity with a prior session on that task). |
| `caseflow-central-brain` | Codex | Planning/documentation work in this repository itself — task graph creation, contract-doc updates, ADRs. |

## Rules
- **Ownership, not exclusivity of knowledge.** Any agent may *read* any repository or any part of Central Brain to gather context. Ownership only governs who is expected to *implement* changes in a given repository.
- **Agents are responsible only for their assigned repository** unless a task file explicitly states otherwise (e.g., a small cross-cutting fix a task author decides to hand to one agent across two repos — this must be called out in that task's `## Agent Instructions` section, not assumed).
- **A task's `Affected Repositories` table is the actual assignment for that task** — this document defines the *default*, which a task graph should follow unless it has a specific reason not to (e.g., the human running the process prefers a different provider that session, or a provider is unavailable).
- **Provider is a property of the task graph, not a hardcoded assumption.** Per the multi-provider requirement for Phase 2 ([../workflows/PHASE-2-AUTOMATION.md](../workflows/PHASE-2-AUTOMATION.md#must-support-multiple-agent-providers)), express ownership in task graphs as `agent: {provider, repository}` (see [../workflows/TASK-GRAPH-FORMAT.md](../workflows/TASK-GRAPH-FORMAT.md)) rather than assuming any one vendor is permanently bound to any one repository — the table above is today's default, not an architectural constraint.
- **`caseflow-central-brain` itself is owned by Codex by default**, but any agent working in any repository is expected to update Central Brain documentation when it learns something cross-repository-relevant (see [../agents/GLOBAL.md](GLOBAL.md)'s multi-agent coordination rule) — ownership of *planning/task-graph* work in this repo defaults to Codex; ownership of *keeping docs accurate* is everyone's responsibility, all the time.

## Changing ownership
If a repository's default agent changes (e.g., a team standardizes on a single provider for both `caseflow-fe` and `caseflow-mobil`), update the table above in the same change that updates any task templates or in-flight task files referencing the old default.
