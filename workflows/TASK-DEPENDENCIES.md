# Task Dependencies

## Purpose
Defines how to reason about ordering between a task's per-repository sub-tasks. The goal is to **maximize parallelism while preventing contract conflicts** — not to apply one rigid sequence to every task regardless of what it actually requires.

## The naive default (do not apply blindly)
```
BE contract
    ↓
FE implementation
    ↓
Mobile implementation
    ↓
Integration
```
This is a reasonable *starting assumption* when a task introduces a brand-new contract that FE/mobile don't yet have anything to build against. It is the wrong assumption whenever:
- The contract already exists and is stable — FE and mobile can both start immediately, in parallel, with no dependency on a BE sub-task at all.
- Only one client needs the change — don't create a mobile sub-task (let alone a dependency on it) for a web-only admin feature.
- BE, FE, and mobile changes are independent of each other by construction (e.g., three unrelated bug fixes bundled under one task for convenience) — each should have its own empty `depends_on`.

## How to determine real ordering
For each sub-task, ask: **what specifically does this sub-task need to exist before it can start?** Not "what repository does it feel like it should follow" — an actual artifact (an endpoint, a DTO shape, a migration, a config flag) it cannot proceed without.

Example — a genuinely new contract:
```yaml
task: AUTH-001
dependencies:
  - BE-001   # FE/mobile need the new endpoint's actual shape before they can implement against it
```

Example — FE and mobile against an already-established contract:
```yaml
task: CF-002
tasks:
  - id: CF-002-FE
    depends_on: []      # contract already exists and is stable
  - id: CF-002-MOBILE
    depends_on: []      # same contract, no reason to wait on FE
```
Both may proceed in parallel; there is no reason to serialize them just because they're "client work."

Example — mobile explicitly waiting on a new BE contract, while FE does not need this feature at all:
```yaml
task: CF-003
tasks:
  - id: CF-003-BE
    depends_on: []
  - id: CF-003-MOBILE
    depends_on: [CF-003-BE]
  # no CF-003-FE node — this feature is mobile-only
```

## Dependency unlocking
When a task in `depends_on` reaches `DONE` (per [TASK-LIFECYCLE.md](TASK-LIFECYCLE.md)), every task that listed it as a dependency should be re-evaluated: if *all* of its dependencies are now `DONE`, move it from `PLANNED`/`BLOCKED` to `READY`. In Phase 1 this re-evaluation is manual (a human checks the task graph); [PHASE-2-AUTOMATION.md](PHASE-2-AUTOMATION.md#dependency-unlocking) describes how this could become automatic.

## Contract-change dependencies are special
A dependency on a contract-change sub-task (see [../skills/contract-change/SKILL.md](../skills/contract-change/SKILL.md)) is not satisfied merely by code being written — it requires the contract document itself ([../contracts/api/README.md](../contracts/api/README.md), [../contracts/events/README.md](../contracts/events/README.md), or the relevant `repos/<name>/` doc) to be updated in the same change. A BE sub-task that changes a DTO but hasn't updated the contract doc should not be treated as `DONE` for the purpose of unlocking a dependent FE/mobile sub-task — the dependent agent needs the documented shape, not just the merged code, to work from without guessing.

## Integration nodes
Any task graph spanning more than one repository should include an integration node whose `depends_on` lists every repository-scoped sub-task (see [TASK-GRAPH-FORMAT.md](TASK-GRAPH-FORMAT.md) for the exact shape). The integration node is what actually validates the combined change works — it is not satisfied merely by every individual sub-task reaching `DONE` independently.

## Rules
- Never assume a rigid BE → FE → Mobile → Integration chain by default — derive it from what each sub-task actually needs.
- An empty `depends_on` is a valid, common, and often correct answer — don't add a dependency just because it "feels safer."
- A contract-change dependency is only satisfied once the contract document is updated, not merely once code is written.
- Every multi-repository task graph needs an explicit integration node — don't let "all sub-tasks are DONE" silently stand in for "the combined change was validated together."
