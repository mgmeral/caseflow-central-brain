# Skills

## Purpose
Defines how `caseflow-central-brain` reasons about a user request before a task is created — the procedures behind [../workflows/](../workflows/) and [../tasks/](../tasks/), not the tasks themselves.

## What belongs here
Step-by-step analysis/planning procedures for the kinds of requests this orchestration system handles. Each subfolder is one procedure, in its own `SKILL.md`.

## Who should use this
Any agent (human-directed or AI) about to plan cross-repository or contract-affecting work, before creating a task file.

## What should NOT be stored here
- Actual task files — those belong in [../tasks/](../tasks/).
- Repository-internal scaffolding prompts/recipes for generating *application* code (e.g. backend code-generation prompts) — those belong in `repos/backend/prompts/` and `repos/backend/skills/`, which are a different, repository-internal concept from the planning skills here. Do not confuse the two: this top-level `skills/` folder is Central Brain's own planning procedure, not a place for application-code generation helpers.

## Contents
- [analysis/SKILL.md](analysis/SKILL.md) — the general request-to-task-graph procedure every other skill here specializes.
- [alignment/SKILL.md](alignment/SKILL.md) — for comparing BE/FE/Mobile behavior and contracts against each other.
- [cross-repository/SKILL.md](cross-repository/SKILL.md) — for a feature request whose repository scope isn't yet known.
- [contract-change/SKILL.md](contract-change/SKILL.md) — for any change to a REST API, DTO, event, auth, or shared domain model.

## Phase
All skills here are Phase 1: they produce a plan (a task file, per [../tasks/templates/CROSS-REPOSITORY-TASK.md](../tasks/templates/CROSS-REPOSITORY-TASK.md)). None of them modify an application repository. See [../workflows/PHASE-2-AUTOMATION.md](../workflows/PHASE-2-AUTOMATION.md) for the (not yet built) automated execution layer these plans could eventually feed.
