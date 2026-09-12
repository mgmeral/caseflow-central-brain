# Workflows

## Purpose
Defines the procedural rules of the Phase 1 AI Agent Orchestration system: task states, dependency reasoning, the machine-readable task-graph format, how a task is handed to an agent, and the design (not yet implemented) for Phase 2 automation.

## What belongs here
Process/lifecycle documentation that applies across all task types — not the tasks themselves (see [../tasks/](../tasks/)) and not the per-request-type reasoning skills (see [../skills/](../skills/)).

## Who should use this
Anyone creating, updating, or handing off a task in this repository's orchestration system.

## What should NOT be stored here
Actual task files (`tasks/`), or skill-specific analysis procedures (`skills/`) — this folder is the shared rulebook those two draw on.

## Contents
- [TASK-LIFECYCLE.md](TASK-LIFECYCLE.md) — task/node states and transition rules.
- [TASK-DEPENDENCIES.md](TASK-DEPENDENCIES.md) — how to determine real ordering between sub-tasks.
- [TASK-GRAPH-FORMAT.md](TASK-GRAPH-FORMAT.md) — the machine-readable YAML schema every cross-repository task file must include.
- [AGENT-HANDOFF.md](AGENT-HANDOFF.md) — the format for manually handing a `READY` task node to its owning agent (Phase 1).
- [PHASE-2-AUTOMATION.md](PHASE-2-AUTOMATION.md) — design-only document for how this could eventually be automated. Not implemented; do not build against it without a separate explicit decision to start Phase 2.
