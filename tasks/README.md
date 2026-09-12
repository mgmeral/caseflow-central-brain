# Tasks

## Purpose
Tracks cross-repository implementation work derived from features, contracts, and decisions.

## What belongs here
Task definitions, ownership, status, links to related repositories, and references to ADRs/contracts/features.

## Who should use this
Contributors and AI agents coordinating active and completed work across repositories.

## What should NOT be stored here
Source code, duplicate issue tracker exports, or architecture decisions without implementation work.

## Folder usage
- `active/` contains in-progress work (any status per [../workflows/TASK-LIFECYCLE.md](../workflows/TASK-LIFECYCLE.md) except `DONE`/`CANCELLED`).
- `blocked/` contains tasks currently in the `BLOCKED` state whose dependency chain is worth tracking separately from the main `active/` list — optional; a `BLOCKED` task may also simply stay in `active/` with its status marked accordingly. Use this folder when the number of blocked tasks makes `active/` hard to scan.
- `completed/` contains finished (`DONE`) or `CANCELLED` work history — move a task file here when it reaches either terminal state, don't delete it.
- `templates/` contains reusable task file structures: [templates/CROSS-REPOSITORY-TASK.md](templates/CROSS-REPOSITORY-TASK.md) (the standard shape for any non-trivial task) and [templates/EXAMPLE-MOBILE-FE-ALIGNMENT.md](templates/EXAMPLE-MOBILE-FE-ALIGNMENT.md) (a worked, illustrative example — marked EXAMPLE ONLY, not an active task).

## How a task gets created
See [../skills/analysis/SKILL.md](../skills/analysis/SKILL.md) (and its specializations, [../skills/alignment/SKILL.md](../skills/alignment/SKILL.md), [../skills/cross-repository/SKILL.md](../skills/cross-repository/SKILL.md), [../skills/contract-change/SKILL.md](../skills/contract-change/SKILL.md)) for the procedure that turns a user request into a task file under `active/`. Task lifecycle, dependency reasoning, the machine-readable task-graph format, and how a `READY` task is handed to an agent are all defined in [../workflows/](../workflows/).

## Task naming
Use `<TYPE>-<NUMBER>`, e.g. `ALIGN-001`, `AUTH-001`, `CONTRACT-001`, `AI-001`, `MOBILE-001`, `FE-001`, `BE-001`, `CF-001` (generic/cross-cutting). A cross-repository task's per-repository sub-tasks share its ID with a repository suffix: `ALIGN-001-BE`, `ALIGN-001-FE`, `ALIGN-001-MOBILE`, `ALIGN-001-AI`, `ALIGN-001-INTEGRATION` — these suffixes must match the node ids used in that task's machine-readable task graph (see [../workflows/TASK-GRAPH-FORMAT.md](../workflows/TASK-GRAPH-FORMAT.md)). Pick a `TYPE` that describes the work, not the repository alone, when the task is genuinely cross-repository — `MOBILE-001` implies mobile-only work; a task that happens to have a mobile sub-task as part of a broader feature should use a feature-shaped `TYPE` instead.

## Note on pre-framework task files
[active/MOBILE-FE-ALIGNMENT.md](active/MOBILE-FE-ALIGNMENT.md) predates this template/lifecycle framework and does not follow the `<TYPE>-<NUMBER>` naming or the machine-readable task-graph format — it remains a valid, real analysis document and has not been rewritten to fit retroactively. New task files should use the framework in `templates/` and `../workflows/` going forward.
