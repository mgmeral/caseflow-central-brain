# Task Lifecycle

## Purpose
Defines the states a task (or a task-graph node) can be in, and the rules for moving between them. Applies to both a whole cross-repository task and each of its per-repository sub-tasks (see [TASK-GRAPH-FORMAT.md](TASK-GRAPH-FORMAT.md)) — a parent task's own status is derived from its children's, not set independently.

## States

```
PLANNED ──▶ READY ──▶ IN_PROGRESS ──▶ REVIEW ──▶ INTEGRATION ──▶ DONE
   │            │           │                        ▲
   │            │           ▼                        │
   │            └──────▶ BLOCKED ────────────────────┘
   │
   └──────────────────────────────────────────────▶ CANCELLED
                    (from any state)
```

### PLANNED
Task exists but is not ready to execute. Dependencies may be unresolved, the request may still need clarification, or the task graph hasn't been fully built out yet. A task should not sit in `PLANNED` indefinitely without a reason recorded in its `## Notes` section.

### READY
All dependencies listed in the task's `depends_on` are `DONE` (or the task has none), and the responsible agent (per [../agents/AGENT-OWNERSHIP.md](../agents/AGENT-OWNERSHIP.md)) can start immediately. A `READY` task has everything a human needs to hand it off per [AGENT-HANDOFF.md](AGENT-HANDOFF.md) — if it doesn't, it isn't actually `READY`, it's still `PLANNED`.

### IN_PROGRESS
The assigned agent is actively implementing. Only one sub-task per repository should typically be `IN_PROGRESS` against the same code area at once — if two tasks would touch the same files, sequence them instead of running both `IN_PROGRESS` in parallel.

### BLOCKED
The task cannot proceed because a dependency is incomplete, or because something outside the task's control changed (e.g., a contract it depended on turned out to need rework). Record *why* in `## Notes`. A `BLOCKED` task automatically becomes `READY` once every task in its `depends_on` list reaches `DONE` — this transition should be re-checked whenever any task moves to `DONE` (see [TASK-DEPENDENCIES.md](TASK-DEPENDENCIES.md#dependency-unlocking)).

### REVIEW
Implementation exists for this sub-task and requires review before it can be considered finished. This is a per-repository state — a `caseflow-fe` sub-task can be in `REVIEW` while a `caseflow-be` sibling is still `IN_PROGRESS`.

### INTEGRATION
Used at the parent-task level (or on an explicit integration node in the task graph, see [TASK-GRAPH-FORMAT.md](TASK-GRAPH-FORMAT.md)) when multiple repositories' changes need to be validated together — e.g., does the frontend's new UI actually work against the backend's new endpoint end-to-end. A task should not be marked `DONE` straight from each sub-task's `REVIEW` if more than one repository was involved — it must pass through `INTEGRATION` first.

### DONE
Implementation and (where applicable) integration validation are both complete. Move the task file from `tasks/active/` to `tasks/completed/` when it reaches `DONE`.

### CANCELLED
Task is intentionally abandoned — the objective changed, was superseded by another task, or turned out to be unnecessary. Record why in `## Notes` before moving the file to `tasks/completed/` (cancelled tasks are still historical record, not deleted).

## Rules
- A task file's top-level `## Status` reflects the overall task. If it has per-repository sub-tasks with different statuses, the top-level status is the "lowest" one that isn't yet satisfied — e.g., if BE is `DONE` but FE is still `IN_PROGRESS`, the parent task is `IN_PROGRESS`, not `DONE`.
- Never move a task straight from `PLANNED` to `IN_PROGRESS` — it must pass through `READY` so the handoff checklist in [AGENT-HANDOFF.md](AGENT-HANDOFF.md) is actually satisfied.
- Never silently drop a task from `BLOCKED` back to `IN_PROGRESS` without confirming the blocking dependency actually reached `DONE` — check, don't assume.
- `BLOCKED` and `REVIEW` are not failure states to be avoided — a task that surfaces a real blocker by moving to `BLOCKED` (rather than an agent working around it silently) is behaving correctly. See [PHASE-2-AUTOMATION.md](PHASE-2-AUTOMATION.md#failure-handling) for why this matters even more once execution is automated.
