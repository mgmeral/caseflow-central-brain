# Contract-Change Skill

## Purpose
Specialization of [../analysis/SKILL.md](../analysis/SKILL.md) triggered whenever a request touches a REST API shape, a DTO, an event (Kafka payload/topic), authentication/authorization, or a shared domain model/enum. This is the highest-risk category of change in CaseFlow because it can silently break a client that isn't in the same PR.

## Who should use this
Any agent about to plan or review a change to: a controller's request/response shape, an enum's values, a permission code, a JWT claim, an event topic/payload, or a canonical schema listed in [../../contracts/schemas/README.md](../../contracts/schemas/README.md).

## Phase
Phase 1 (manual, planning-only). This skill determines *what kind* of contract change is being proposed and what must happen as a result — it does not modify application code.

## Workflow
Determine, explicitly:

```
Breaking change?
Non-breaking?
Affected repositories?
Migration required?
Backward compatibility?
```

### Breaking vs. non-breaking
Follow the classification already established in [../../agents/CROSS-REPOSITORY-CHANGE.md](../../agents/CROSS-REPOSITORY-CHANGE.md#breaking-vs-non-breaking-changes):
- **Breaking**: removed/renamed field, changed type, removed enum value, required↔optional flip, changed/removed permission code, changed event payload shape or topic name, changed producer/consumer default-enabled state.
- **Non-breaking**: new optional field, new endpoint, new enum value a client can ignore.

### Affected repositories
Use [../../repos/integration-map.md](../../repos/integration-map.md) and [../../repos/dependency-map.md](../../repos/dependency-map.md) to find every caller of the endpoint/event/enum being changed — not just the repository where the change originates. A backend DTO change affects every client listed as a caller in the integration map, even ones not mentioned in the original request.

### Migration required?
- **Database**: does this need a Flyway migration in `caseflow-be` and/or `caseflow-ai-service`? Both are forward-fix-only (no destructive `migrate down`) — see [../../docs/operations/deployment.md](../../docs/operations/deployment.md).
- **Data**: does existing data need backfilling to satisfy a new constraint (e.g., a new required field)?
- **Client**: does an existing client need a code change just to keep working (not to gain new behavior)?

### Backward compatibility
Can the previous client version keep working against the new backend version, at least for one deployment cycle? If not, deployment order matters and must be stated explicitly in the resulting task file (see [../../docs/operations/deployment.md](../../docs/operations/deployment.md#release-coordination) — `caseflow-be` deploys first, but a hard break still needs an explicit note about what breaks in between).

## Special cases already established in this system
- **Kafka async lane** (`caseflow.ai.async.enabled`): enabling this on only one of `caseflow-be`/`caseflow-ai-service` is a deployment misconfiguration, not a valid intermediate state — any contract-change task touching this must include both sides or explicitly justify a one-sided change. See [ADR-0004](../../decisions/0004-optional-kafka-ai-ingestion-lane.md).
- **AI-service auth** (`caseflow.ai.auth.enabled`): enabling this requires `caseflow-be`'s AI client to start sending the internal API key header in the same change — see [ADR-0002](../../decisions/0002-ai-service-no-auth-p1.md).
- **Ticket identifiers**: new cross-repository references should default to `publicId` (UUID), not the numeric PK or `ticketNo`, per [ADR-0003](../../decisions/0003-sequential-ticket-numbers-public-uuid.md) — a contract-change task introducing a new ticket-referencing field should call this out explicitly rather than defaulting to whatever an adjacent endpoint happens to use (the current API contract has both, inconsistently — see [tasks/active/MOBILE-FE-ALIGNMENT.md](../../tasks/active/MOBILE-FE-ALIGNMENT.md)).

## Output
Every contract change must, in the same task/change:
1. Update the relevant contract document ([../../contracts/api/README.md](../../contracts/api/README.md), [../../contracts/events/README.md](../../contracts/events/README.md), [../../contracts/schemas/README.md](../../contracts/schemas/README.md), or the specific `repos/<name>/` doc).
2. Be flagged as breaking or non-breaking in the task file's `## Contract Impact` section.
3. List every affected repository from the integration map, not just the one being changed first.
4. Get an ADR ([../../decisions/](../../decisions/)) if it reflects a real architectural decision, not just a shape fix.

## Non-negotiable rules
- Do not modify application code from this skill — it classifies and plans.
- Do not let a contract change ship without updating its contract document in the same change (per [../../agents/CROSS-REPOSITORY-CHANGE.md](../../agents/CROSS-REPOSITORY-CHANGE.md)).
- When in doubt about breaking vs. non-breaking, treat it as breaking — the cost of over-cautious coordination is much lower than an undetected client break.
