# Analysis Skill

## Purpose
Defines how `caseflow-central-brain` turns a user request into a concrete, cross-repository task graph. This is the entry point for almost everything else in `skills/`, `workflows/`, and `tasks/` — the alignment, cross-repository, and contract-change skills are specializations of this general procedure, not alternatives to it.

## Who should use this
Any agent (human-directed or AI) asked to plan work that might touch more than one CaseFlow repository, or whose scope is not yet fully clear from the request alone.

## Phase
Phase 1 (manual). This skill produces a task file. It does not execute agents, does not open pull requests, and does not modify any application repository. See [../../agents/AGENT-OWNERSHIP.md](../../agents/AGENT-OWNERSHIP.md) and [../../workflows/AGENT-HANDOFF.md](../../workflows/AGENT-HANDOFF.md) for what happens after this skill produces its output.

## Workflow

```
User Request
    ↓
Understand Objective
    ↓
Inspect Central Brain Context
    ↓
Inspect Relevant Application Repositories
    ↓
Identify Affected Repositories
    ↓
Identify Existing Capabilities
    ↓
Identify Gaps / Inconsistencies
    ↓
Determine Dependencies
    ↓
Determine Agent Ownership
    ↓
Create Task Graph
    ↓
Create Task File
```

### 1. Understand Objective
Restate the request as a concrete outcome, not a vague theme. "Add customer notification preferences" is a theme; "customers can opt out of a given notification type per channel, enforced server-side and reflected in FE/mobile settings" is an objective. If the request is ambiguous, resolve the ambiguity before proceeding rather than guessing — ask, or state the assumption explicitly in the task file's `## Context` section.

### 2. Inspect Central Brain Context
Read, in this order, before touching an application repository:
- [../../repos/repository-context.md](../../repos/repository-context.md) — fast orientation.
- [../../repos/repository-map.md](../../repos/repository-map.md), [../../repos/dependency-map.md](../../repos/dependency-map.md), [../../repos/integration-map.md](../../repos/integration-map.md) — verified facts about what exists and how repos talk to each other.
- [../../docs/architecture/](../../docs/architecture/) for every repository the request plausibly touches.
- [../../contracts/](../../contracts/) and [../../decisions/](../../decisions/) for the relevant surface.
- [../../features/](../../features/) — an existing feature doc may already cover part of this request.
- [../../tasks/active/](../../tasks/active/) and [../../tasks/blocked/](../../tasks/blocked/) — avoid planning work that duplicates or conflicts with something already in flight.

### 3. Inspect Relevant Application Repositories
Central Brain documentation can go stale (see every "verified as of" date in this repo). Before finalizing a task graph:
- Confirm load-bearing claims against actual source in the affected repositories, not just against this repo's docs — especially for anything marked `TODO: Verify` or `UNKNOWN` in the relevant architecture doc.
- Do this read-only. This skill never edits an application repository.
- If a claim in Central Brain turns out to be stale, note it in the task file and flag it for a Central Brain correction — do not silently work around stale documentation.

### 4. Identify Affected Repositories
For each of `caseflow-be`, `caseflow-fe`, `caseflow-mobil`, `caseflow-ai-service`: does this request require a change here, or not? Be explicit about "not affected" too — a task file that only lists affected repos leaves the next reader unsure whether an unlisted repo was considered and excluded, or simply forgotten.

### 5. Identify Existing Capabilities
What already exists, in which repository, that this request can build on? Reuse an existing contract/endpoint/component over inventing a new one. This is where [../alignment/SKILL.md](../alignment/SKILL.md) is useful if the request is fundamentally about comparing what two or more clients already do.

### 6. Identify Gaps / Inconsistencies
What's missing, or inconsistent between repositories, relative to what the objective requires? **Never invent implementation details to fill a gap in your understanding** — if something can't be verified, mark it `TODO: Verify` or `UNKNOWN: Not established in source repository` in the task file, the same discipline used throughout this repo's architecture docs.

### 7. Determine Dependencies
Which affected repositories can proceed in parallel, and which must wait on another? Default assumption: a backend contract change blocks any client that depends on the new/changed shape; two clients that only depend on an *already-stable* contract can proceed in parallel. Do not default to a rigid BE → FE → Mobile → Integration chain for every task — see [../../workflows/TASK-DEPENDENCIES.md](../../workflows/TASK-DEPENDENCIES.md) for how to reason about this per-task rather than by template.

### 8. Determine Agent Ownership
Map each repository-scoped piece of work to the agent that owns that repository per [../../agents/AGENT-OWNERSHIP.md](../../agents/AGENT-OWNERSHIP.md), unless the task explicitly overrides it (and says why).

### 9. Create Task Graph
Express the dependency structure from step 7 as the machine-readable format defined in [../../workflows/TASK-GRAPH-FORMAT.md](../../workflows/TASK-GRAPH-FORMAT.md) — one graph node per repository-scoped sub-task, plus an integration node if more than one repository is involved.

### 10. Create Task File
Instantiate [../../tasks/templates/CROSS-REPOSITORY-TASK.md](../../tasks/templates/CROSS-REPOSITORY-TASK.md) (or a single-repository equivalent if only one repo is affected) and save it under `tasks/active/` using the naming convention in [../../tasks/README.md](../../tasks/README.md). Do not mark anything `IN_PROGRESS` — a freshly created task starts at `PLANNED` or `READY` per [../../workflows/TASK-LIFECYCLE.md](../../workflows/TASK-LIFECYCLE.md).

## Non-negotiable rules
- Inspect actual repositories when a claim matters and isn't already verified in Central Brain.
- Use Central Brain documentation as context, not as an unquestionable source once it might be stale — cross-check load-bearing claims.
- Never invent implementation details. Mark unknowns explicitly.
- Always distinguish implemented vs. planned functionality — never let a task file imply something is built when it isn't (or vice versa).
- Always identify cross-repository dependencies and contract changes explicitly, even when the answer is "none."
- Always identify task ordering and agent ownership — a task graph with no dependency information or no owner is incomplete.
- This skill produces a plan. It does not execute one. See [../../workflows/AGENT-HANDOFF.md](../../workflows/AGENT-HANDOFF.md) for how a human moves a `READY` task to an actual agent.
