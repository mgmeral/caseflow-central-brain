# Cross-Repository Skill

## Purpose
Specialization of [../analysis/SKILL.md](../analysis/SKILL.md) for a user request that describes a *feature* (not an audit — see [../alignment/SKILL.md](../alignment/SKILL.md) for comparisons) whose scope across repositories isn't yet known. Example request: "Add customer notification preferences."

## Who should use this
Any agent handed a feature request where it isn't immediately obvious which repositories are involved.

## Phase
Phase 1 (manual, planning-only). Produces a task graph; does not implement it. See [../../workflows/AGENT-HANDOFF.md](../../workflows/AGENT-HANDOFF.md) for how a human then starts each sub-task with the owning agent.

## Workflow
For the request, determine each of the following explicitly — write "no" or "none" rather than omitting a line, so a reviewer can tell the question was considered:

```
BE?
FE?
Mobile?
AI?
Contract?
Database?
Events?
```

### BE?
Does this require a new/changed endpoint, entity, migration, permission code, or business rule in `caseflow-be`? `caseflow-be` is the dependency root ([../../docs/architecture/system-overview.md](../../docs/architecture/system-overview.md)) — most non-trivial features touch it, and most other repos' work is blocked on it landing first (see [../../workflows/TASK-DEPENDENCIES.md](../../workflows/TASK-DEPENDENCIES.md) for when that's *not* true).

### FE?
Does a web UI need to expose this? Check whether `caseflow-fe` already has a relevant admin/settings surface to extend (see [../../repos/frontend/README.md](../../repos/frontend/README.md)) before assuming a new page is needed.

### Mobile?
Does `caseflow-mobil` need this too, or is it explicitly out of scope for a read-mostly mobile client? Check [../../docs/architecture/mobile.md](../../docs/architecture/mobile.md) and don't assume mobile parity is required by default — CaseFlow's mobile app has a deliberately narrower scope than the web frontend today.

### AI?
Does this intersect with `caseflow-ai-service` (e.g., a new document type to ingest, a new AI-assist capability)? If so, check whether the request needs synchronous REST ingestion, the optional Kafka lane, or a new AI-assist endpoint — these are different sizes of change (see [../../docs/architecture/ai-service.md](../../docs/architecture/ai-service.md)).

### Contract?
Does this add or change a REST endpoint, DTO, enum, or permission code? If yes, this also requires the [../contract-change/SKILL.md](../contract-change/SKILL.md) workflow — run it before finalizing the task graph, not after.

### Database?
Does this require a new entity/column/migration in `caseflow-be` and/or `caseflow-ai-service`'s Postgres, or a new MongoDB field? Flyway is forward-fix-only in both backend repos ([../../docs/operations/deployment.md](../../docs/operations/deployment.md)) — plan migrations accordingly, and note whether the change is backward-compatible with a running previous-version deployment (needed for rollback safety).

### Events?
Does this need to flow through the optional Kafka lane, or the internal `IntegrationJob` durable queue (Jira/Slack/Teams/webhook pattern)? Most new async work should reuse the `IntegrationJob` pattern already established in `caseflow-be` rather than introducing a new mechanism — see [../../repos/backend/module-map.md](../../repos/backend/module-map.md#12-integration).

## Output
Once each question above is answered, build the task graph exactly as in [../analysis/SKILL.md](../analysis/SKILL.md) steps 7–10: determine dependencies, assign agents per [../../agents/AGENT-OWNERSHIP.md](../../agents/AGENT-OWNERSHIP.md), express it in the format from [../../workflows/TASK-GRAPH-FORMAT.md](../../workflows/TASK-GRAPH-FORMAT.md), and instantiate [../../tasks/templates/CROSS-REPOSITORY-TASK.md](../../tasks/templates/CROSS-REPOSITORY-TASK.md).

## Non-negotiable rules
- Answer every question above explicitly, even "no."
- Do not assume every feature needs all four repositories — CaseFlow's own history includes real features (e.g., Jira integration, notification-channel admin) that are BE+FE-only by design, with no mobile or AI-service involvement.
- A "Contract?" answer of "yes" always routes through [../contract-change/SKILL.md](../contract-change/SKILL.md) before the task graph is finalized.
