# Task Graph Format

## Purpose
Defines the machine-readable YAML block every cross-repository task file must include, so a task graph can eventually be parsed by tooling (Phase 2, see [PHASE-2-AUTOMATION.md](PHASE-2-AUTOMATION.md)) without needing to change format later. In Phase 1 this block is read and updated by hand, but its shape should not need to change when execution is automated.

## Where it lives
Inside the task file itself (see [../tasks/templates/CROSS-REPOSITORY-TASK.md](../tasks/templates/CROSS-REPOSITORY-TASK.md)'s `## Task Graph` section), as a fenced ` ```yaml ` block.

## Schema

```yaml
task_id: <TYPE>-<NUMBER>          # matches the task file's own ID, see ../tasks/README.md#task-naming
title: <short title>
status: <PLANNED|READY|IN_PROGRESS|BLOCKED|REVIEW|INTEGRATION|DONE|CANCELLED>  # derived, see TASK-LIFECYCLE.md

tasks:
  - id: <TASK_ID>-<REPO_SUFFIX>    # e.g. CF-001-BE, CF-001-FE, CF-001-MOBILE, CF-001-AI, CF-001-INTEGRATION
    repository: <caseflow-be|caseflow-fe|caseflow-mobil|caseflow-ai-service|caseflow-central-brain>
    agent:
      provider: <claude|copilot|codex>   # see ../agents/AGENT-OWNERSHIP.md for defaults
    status: <same enum as above>
    depends_on: [<list of other node ids, or empty>]
```

## Field notes
- `repository` must be one of the five exact repository names used throughout this repo — do not abbreviate or use a display name.
- `agent.provider` is deliberately a property of the node, not hardcoded per repository in the schema itself — see [../agents/AGENT-OWNERSHIP.md](../agents/AGENT-OWNERSHIP.md#rules) and the Phase 2 multi-provider requirement ([PHASE-2-AUTOMATION.md](PHASE-2-AUTOMATION.md#must-support-multiple-agent-providers)). Populate it from the ownership defaults unless the task has a specific reason to override.
- `status` per node follows [TASK-LIFECYCLE.md](TASK-LIFECYCLE.md) exactly — do not invent additional states.
- `depends_on` lists **node ids from this same graph** (or, rarely, a node id from a different task file if one task is genuinely blocked on another already-existing task — prefer restructuring into one task graph over cross-file dependencies where possible, since cross-file dependencies are harder to keep in sync).
- An `-INTEGRATION` node is required whenever more than one repository node exists in the graph, and its `depends_on` must list every other node in the graph (see [TASK-DEPENDENCIES.md](TASK-DEPENDENCIES.md#integration-nodes)).

## Full example

```yaml
task_id: CF-001
title: Mobile FE BE Alignment
status: PLANNED

tasks:
  - id: CF-001-BE
    repository: caseflow-be
    agent:
      provider: claude
    status: READY
    depends_on: []

  - id: CF-001-FE
    repository: caseflow-fe
    agent:
      provider: copilot
    status: READY
    depends_on: []

  - id: CF-001-MOBILE
    repository: caseflow-mobil
    agent:
      provider: copilot
    status: BLOCKED
    depends_on:
      - CF-001-BE

  - id: CF-001-INTEGRATION
    repository: caseflow-central-brain
    agent:
      provider: codex
    status: BLOCKED
    depends_on:
      - CF-001-FE
      - CF-001-MOBILE
```

Note this example graph is illustrative of the *format*, not a prescription that every task needs exactly this shape — a real task might have no FE node, might have an AI-service node instead, or might have three BE-side nodes if the backend work itself has internal phasing worth tracking separately. Build the graph from the actual dependency analysis in [TASK-DEPENDENCIES.md](TASK-DEPENDENCIES.md), not from this template's shape.

## Rules
- Every cross-repository task file must include exactly one such YAML block, kept in sync with the prose `## Affected Repositories` table and `## Dependencies` section in the same file — if they disagree, the file is wrong and needs fixing, not a judgment call about which to trust.
- Use YAML consistently across all task files — do not switch formats between tasks; that's what makes eventual Phase 2 parsing possible.
- The `TYPE` in `task_id` follows the naming convention in [../tasks/README.md](../tasks/README.md#task-naming).
