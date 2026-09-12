# Alignment Skill

## Purpose
Specialization of [../analysis/SKILL.md](../analysis/SKILL.md) for requests that are fundamentally about comparing two or more CaseFlow clients/services against each other and against the `caseflow-be` contract — API model alignment, authentication alignment, error handling, pagination, permissions, and feature-capability parity.

## Who should use this
Any agent asked to audit or reconcile behavior across `caseflow-be` / `caseflow-fe` / `caseflow-mobil` (and, where relevant, `caseflow-ai-service`'s contract with `caseflow-be`). A worked example of this skill's output exists at [../../tasks/active/MOBILE-FE-ALIGNMENT.md](../../tasks/active/MOBILE-FE-ALIGNMENT.md) — read it for the expected level of detail and evidence before producing a new alignment audit, but do not copy its conclusions forward without re-verifying if time has passed since it was written.

## Phase
Phase 1 (manual, planning-only). **This skill must NOT modify any application repository.** It produces a comparison and a set of remediation tasks; a human decides which remediation tasks to actually schedule and hands them to the owning agent per [../../agents/AGENT-OWNERSHIP.md](../../agents/AGENT-OWNERSHIP.md).

## Workflow

```
Inspect BE
Inspect FE
Inspect Mobile
Compare contracts
Compare models
Compare behavior
Identify mismatches
Create remediation tasks
Assign agents
Define dependencies
```

### Inspect BE
`caseflow-be` is the reference — its actual controllers/DTOs/permission catalog define what's *available*, not what any client currently does with it. Use [../../repos/backend/frontend-contract.md](../../repos/backend/frontend-contract.md) and [../../repos/integration-map.md](../../repos/integration-map.md) as the starting point, verifying against source where either is marked `TODO: Verify`.

### Inspect FE / Inspect Mobile
For each client, establish what it actually calls and does — not what its UI implies. A permission-gated page that renders a static explainer (see: `/admin/sla-policy` in `caseflow-fe`) is not a working feature. A screen that reads a feature flag no code path ever checks is not a working feature either. Distrust names and comments; verify against the actual service/API-client code.

### Compare contracts
Do both clients call the same endpoints the same way? Same identifier type (`id` vs `publicId` — see [ADR-0003](../../decisions/0003-sequential-ticket-numbers-public-uuid.md))? Same pagination shape? Same error-handling expectations?

### Compare models
Do both clients model the same domain concepts (ticket status, permission codes, SLA state, etc.) consistently, or has one drifted (e.g., a removed backend enum value still referenced in one client's types)?

### Compare behavior
Where the contract is identical but client behavior differs (e.g., one client implements token refresh and the other doesn't), that's a behavioral misalignment even though there's no contract mismatch. **Do not assume the client with more features is automatically "correct"** — compare both against the backend contract and note which one actually implements the contract's intended behavior correctly. A more full-featured client can still be behind on a specific point (see the refresh-token finding in the mobile/FE audit, where the *simpler* client had the more correct implementation).

### Identify mismatches
Categorize each finding as one of:
- **Gap** — one client hasn't built a capability the backend already supports and another client already uses.
- **Inconsistency** — both clients do something, but differently, in a way that isn't just a reasonable platform difference (e.g., inconsistent identifier usage, inconsistent permission-gating logic).
- **Aligned-but-incomplete** — both clients are equally behind the backend (not a cross-client inconsistency; note it, but don't file it as one client's fault).
- **Platform-appropriate difference** — not a misalignment at all (e.g., `localStorage` vs. `expo-secure-store` for token storage is a reasonable per-platform choice, not a bug, unless it demonstrably weakens security in a way worth flagging separately).

### Create remediation tasks
One task (or task graph, if cross-repository) per mismatch that's actually worth scheduling — not every finding needs a task immediately; some belong in `docs/product/roadmap.md` as a future consideration instead. Use [../../tasks/templates/CROSS-REPOSITORY-TASK.md](../../tasks/templates/CROSS-REPOSITORY-TASK.md).

### Assign agents / Define dependencies
Per [../../agents/AGENT-OWNERSHIP.md](../../agents/AGENT-OWNERSHIP.md) and [../../workflows/TASK-DEPENDENCIES.md](../../workflows/TASK-DEPENDENCIES.md).

## Non-negotiable rules
- Never modify an application repository from this skill — it plans, it does not implement.
- Never assume symmetry — the two clients being compared do not need to converge on identical behavior; the goal is identifying where divergence is a real gap versus an intentional platform/scope difference.
- Always reference the backend contract as ground truth, not either client's current behavior — a client can be "aligned with the other client" while both are wrong relative to the backend.
