# Agent Handoff

## Purpose
Defines the format a human uses to manually hand one task-graph node (see [TASK-GRAPH-FORMAT.md](TASK-GRAPH-FORMAT.md)) to the agent that owns it. This is the Phase 1 substitute for automatic agent triggering (see [PHASE-2-AUTOMATION.md](PHASE-2-AUTOMATION.md#task-triggering) for how that could eventually work) — in Phase 1, a person reads this, opens the target repository in the right tool, and pastes/adapts it as the agent's instructions.

## When to use this
Whenever a task-graph node's status is `READY` (per [TASK-LIFECYCLE.md](TASK-LIFECYCLE.md)) and a human is about to start the owning agent on it.

## Format

```
Task:
<node id, e.g. CF-001-MOBILE>

Repository:
<exact repository name>

Agent:
<provider, from the node's agent.provider>

Read first:
- Central Brain repository-context (repos/repository-context.md)
- <the architecture doc(s) relevant to this repository>
- <the specific contract doc(s) this task touches>
- <the parent task file itself, e.g. tasks/active/CF-001.md>

Objective:
<restated from the parent task's ## Objective and this node's specific slice of it>

Constraints:
<anything this agent must NOT do — see the standard list below>

Acceptance Criteria:
<copied/derived from the parent task's ## Acceptance Criteria, scoped to this repository>

Do not modify:
<every other repository not listed above>
```

## Standard constraints to include on every handoff
Unless a task explicitly overrides one of these (and says why, in its own `## Notes`):
- Do not modify a repository other than the one named in this handoff.
- Do not change a shared contract (API shape, event payload, permission code, shared enum) without it being explicitly in scope for this task — if it turns out to be necessary, stop and flag it rather than making the change unilaterally (see [../skills/contract-change/SKILL.md](../skills/contract-change/SKILL.md)).
- Do not invent behavior the parent task doesn't call for — if something is ambiguous, flag it rather than guessing.
- Preserve existing behavior outside this task's scope; no unrelated refactoring.
- Run this repository's own test suite before considering the sub-task ready for `REVIEW`.
- Report back anything discovered that changes shared understanding (a contract shape, a domain rule, a gap in Central Brain's documentation) so it can be folded back into Central Brain in the same change, per [../agents/GLOBAL.md](../agents/GLOBAL.md).

## Worked example

```
Task:
CF-001-MOBILE

Repository:
caseflow-mobil

Agent:
GitHub Copilot

Read first:
- repos/repository-context.md
- docs/architecture/mobile.md
- repos/backend/frontend-contract.md
- tasks/active/CF-001.md

Objective:
Implement the mobile-side piece of CF-001 as scoped in that task's
"Tasks > Mobile" checklist, once CF-001-BE is DONE and its contract
doc updates have landed.

Constraints:
- Do not modify caseflow-be, caseflow-fe, or caseflow-ai-service.
- Do not add a new auth flow — mobile follows the exact same
  POST /api/auth/login + /refresh + /logout + GET /api/auth/me
  contract as caseflow-fe; do not invent a mobile-only variant.
- Gate any new UI on permissionCodes, never on role name.

Acceptance Criteria:
- [ ] (copied from CF-001's Mobile-scoped acceptance criteria)

Do not modify:
- caseflow-be
- caseflow-fe
- caseflow-ai-service
```

## Rules
- A handoff must contain enough context for the agent to work without guessing — if you find yourself wanting to add "and use your judgment on X," that's a sign the parent task file is underspecified and should be fixed instead.
- Never hand off a node that isn't actually `READY` (check its `depends_on` first).
- When the agent finishes, update the node's status in the task file (`IN_PROGRESS` → `REVIEW`) before moving on — don't let the task graph drift from what's actually true.
