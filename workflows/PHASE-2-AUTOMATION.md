# Phase 2 Automation (Design Only — Not Implemented)

## Purpose
Describes how the manual Phase 1 process (analysis → task graph → human-mediated handoff, see [AGENT-HANDOFF.md](AGENT-HANDOFF.md)) could eventually be automated. **This is a design document, not a build plan for right now.** Nothing in this document should be implemented without a separate, explicit decision to start Phase 2 — see [../docs/product/roadmap.md](../docs/product/roadmap.md) for where that decision would be tracked.

## Target shape

```
Central Brain
      ↓
Task Graph
      ↓
GitHub Issues / Pull Requests
      ↓
Agent Runner / Orchestrator
      ↓
Claude / Copilot / Codex
      ↓
Branches
      ↓
Pull Requests
      ↓
CI
      ↓
Integration
      ↓
Central Brain
```

Central Brain remains the loop's start and end point: it produces the task graph, and the results (contract changes, new capabilities, discovered gaps) flow back into it — this round-trip is the same one Phase 1 already does by hand.

## Task triggering
A task-graph node reaching `READY` (per [TASK-LIFECYCLE.md](TASK-LIFECYCLE.md)) is the trigger condition. In Phase 2, an orchestrator watching the task graph (rather than a human reading it) would detect this transition and invoke the node's assigned agent runner directly, using the same handoff content a human currently assembles by hand from [AGENT-HANDOFF.md](AGENT-HANDOFF.md) — the handoff format is designed to be machine-assemblable already (it's built entirely from fields already present in the task file and linked docs), so this is a change in *who* assembles and delivers it, not in its content.

## Repository isolation
Each agent runner operates inside its own isolated checkout — its own branch, and ideally its own worktree or container — of exactly the one repository named in its task-graph node. No agent runner should have write access to a repository it wasn't assigned. This mirrors the "Do not modify" constraint already present in every Phase 1 handoff, made structurally enforced rather than instruction-enforced.

## Dependency unlocking
When a node's status is set to `DONE` (in Phase 2: when its PR merges and passes CI), the orchestrator re-evaluates every node listing it in `depends_on` (see [TASK-DEPENDENCIES.md](TASK-DEPENDENCIES.md#dependency-unlocking)) and flips any node whose dependencies are now all `DONE` from `BLOCKED`/`PLANNED` to `READY`, which in turn triggers it per "Task triggering" above. This is the same manual re-check a human does in Phase 1, made event-driven.

## Pull requests
Every agent runner produces a pull request in its assigned repository rather than pushing directly to a default branch — even for a repository-internal, low-risk change. The PR description should reference the task-graph node id so the relationship is traceable (see Auditability below).

## CI
A PR must pass its repository's own CI (existing test suites — see each repo's testing approach in [../repos/repository-map.md](../repos/repository-map.md)) before it's eligible to move toward `INTEGRATION`. CI failure does not retry silently — see Failure handling below.

## Cross-repository validation
Once every sub-task's PR in a graph is green and merged, an `INTEGRATION` step (see [TASK-LIFECYCLE.md](TASK-LIFECYCLE.md#integration)) runs whatever validates the *combined* change — e.g., an end-to-end test that exercises the new FE UI against the new BE endpoint together, not just each side's own unit tests. This step's design (what tooling, what environment) is intentionally left open here — it depends on what integration-testing infrastructure exists by the time Phase 2 is actually built, which is outside this document's scope.

## Failure handling
An agent runner failing (can't produce a working implementation, produces a PR that fails CI and isn't fixed after reasonable retries, or hits an ambiguity it can't resolve) must move its node to `BLOCKED` or `REVIEW` — **never silently continue, and never silently mark itself `DONE`.** This is the same principle already stated for Phase 1 in [TASK-LIFECYCLE.md](TASK-LIFECYCLE.md#rules); Phase 2 just needs to enforce it structurally (the orchestrator, not the agent itself, is what ultimately decides a node reached `DONE`, based on objective signals like "PR merged and CI green" — an agent claiming it's done is not sufficient by itself).

## Human approval
Human approval remains mandatory at, at minimum:
- Merging any PR that changes a shared contract (see [../skills/contract-change/SKILL.md](../skills/contract-change/SKILL.md)) — never auto-merge a breaking or even non-breaking contract change without a human reviewing the contract-doc update alongside the code.
- Moving a task graph's `INTEGRATION` result to `DONE` for anything touching authentication, permissions, or a production deployment path.
- Enabling any previously-disabled-by-default capability in production (e.g., the Kafka async lane, AI-service auth — see [ADR-0004](../decisions/0004-optional-kafka-ai-ingestion-lane.md), [ADR-0002](../decisions/0002-ai-service-no-auth-p1.md)).
- Any action this document doesn't explicitly say can be automated — default to requiring approval, not the other way around.

## Security
- Agent runners should be issued credentials scoped to exactly the repository (and, ideally, branch) they're working in — never a broadly-scoped token usable across all four application repositories plus Central Brain.
- No agent runner should hold credentials to a production deployment target, a secrets store, or any repository it isn't actively assigned to at that moment.
- Credentials should be issued per-task-execution where possible, not as a long-lived standing credential per agent provider.

## Auditability
Every automated action must be traceable to the full chain:
```
Task ID → Agent → Repository → Branch → Commit → Pull Request → CI result
```
In practice this means: the task-graph node id appears in the branch name and/or PR title/description, the PR references the task file, and the task file (once the node reaches `DONE`) links back to the merged PR. This traceability requirement holds in Phase 1 too, informally (a task file should already reference the PRs/commits it produced) — Phase 2 just needs to make it structurally guaranteed rather than a matter of an agent remembering to do it.

## Must support multiple agent providers
Do not design Phase 2 around a single AI vendor. The architecture must treat the agent as a pluggable property of a task-graph node:
```yaml
agent:
  provider: copilot
  repository: caseflow-mobil
```
rather than hardcoding provider-specific logic into the task system itself (see [TASK-GRAPH-FORMAT.md](TASK-GRAPH-FORMAT.md), which already models `agent.provider` this way in Phase 1 for exactly this reason). An orchestrator implementation may need a provider-specific adapter per agent (Claude, Copilot, Codex, and whatever comes after), but the task graph, dependency model, and lifecycle states must remain identical regardless of which provider a given node uses.

## Central Brain should not run code
Central Brain's role stays: **source of truth, planner, coordinator, policy engine.** It defines what `READY` means, what a valid task graph looks like, and what contract/security rules apply — it does not itself execute application code, run builds, or hold deployment credentials. Execution is delegated entirely to isolated agent runners (see Repository isolation above). This separation matters because Central Brain is meant to remain simple, auditable, and low-privilege even as the rest of the system gains automation — conflating "the plan" with "the execution engine" would make Central Brain itself a security-sensitive, stateful system, which defeats its purpose as a lightweight, always-current documentation and coordination layer.
