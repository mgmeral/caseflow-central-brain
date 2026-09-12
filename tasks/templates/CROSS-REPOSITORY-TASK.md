# <TASK-ID> — <TITLE>

## Objective
<One or two sentences: the concrete outcome, not a theme. See skills/analysis/SKILL.md step 1.>

## Status
PLANNED

## Priority
P0 / P1 / P2

## User Request
<The original request, close to verbatim, so a future reader can see what was actually asked versus how it was interpreted.>

## Context
<What exists today, what gap this closes, links to the relevant Central Brain docs (architecture, features, ADRs) that informed this task. Use TODO: Verify / UNKNOWN: Not established in source repository for anything not confirmed.>

## Affected Repositories

| Repository | Agent | Responsibility | Status |
|---|---|---|---|
| caseflow-be | Claude | ... | ... |
| caseflow-fe | Copilot | ... | ... |
| caseflow-mobil | Copilot | ... | ... |
| caseflow-ai-service | Claude/Codex | ... | ... |

Delete any row for a repository this task doesn't touch — do not leave a placeholder row implying uncertainty about scope. See skills/cross-repository/SKILL.md.

## Dependencies

### Depends On
<Other task IDs this cannot start without, or "None".>

### Blocks
<Other task IDs waiting on this one, or "None known yet".>

See workflows/TASK-DEPENDENCIES.md — do not default to a rigid BE→FE→Mobile chain; derive real ordering per sub-task.

## Contract Impact
- API: <yes/no, and which endpoints/DTOs>
- Event: <yes/no, and which topic/payload>
- Database: <yes/no, and which repo's migration>
- Authentication: <yes/no>
- If any of the above is yes, run skills/contract-change/SKILL.md before finalizing this section.
- None (if truly nothing above applies)

## Tasks

### BE
- [ ] ...

### FE
- [ ] ...

### Mobile
- [ ] ...

### AI
- [ ] ...

Delete sections for repositories not in scope.

## Task Graph
```yaml
task_id: <TASK-ID>
title: <TITLE>
status: PLANNED

tasks:
  - id: <TASK-ID>-BE
    repository: caseflow-be
    agent:
      provider: claude
    status: PLANNED
    depends_on: []

  # Add / remove nodes to match the Affected Repositories table above exactly.
  # Include a <TASK-ID>-INTEGRATION node whenever more than one repository node exists
  # — see workflows/TASK-GRAPH-FORMAT.md and workflows/TASK-DEPENDENCIES.md#integration-nodes.
```

## Acceptance Criteria
- [ ] ...

## Validation
- [ ] Backend tests
- [ ] Frontend tests
- [ ] Mobile tests
- [ ] Integration validation

Strike out any line for a repository not in scope rather than deleting it, so it's clear the omission was deliberate.

## Agent Instructions

### Claude
...

### Copilot
...

### Codex
...

Use workflows/AGENT-HANDOFF.md's format for the actual handoff text when a node becomes READY — this section can stay high-level (constraints/pitfalls specific to this task); the handoff document assembles the rest from this file plus linked docs.

## Completion Requirements
A task is not COMPLETE until all required repositories have completed their work and integration validation has passed. See workflows/TASK-LIFECYCLE.md.

## Notes
<Anything that doesn't fit elsewhere: open questions, deferred scope, why a dependency was or wasn't added, discoveries that should feed back into Central Brain docs.>
