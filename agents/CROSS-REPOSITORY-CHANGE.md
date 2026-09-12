# Cross-Repository Change Workflow

## Purpose
Standard workflow for any change that touches more than one CaseFlow repository, or changes a shared contract. Applies on top of [GLOBAL.md](GLOBAL.md).

## Who should use this
Any agent about to implement a feature or fix that spans `caseflow-be`, `caseflow-fe`, `caseflow-ai-service`, and/or `caseflow-mobil`, or that changes an API/event/schema contract even within one repository.

## Workflow

1. **Identify the feature.** Check [../features/](../features/) for an existing feature doc; if none exists and the change is genuinely cross-repository, create one from [../features/template.md](../features/template.md) before writing code.
2. **Identify affected repositories.** Use [../repos/repository-map.md](../repos/repository-map.md) and [../repos/dependency-map.md](../repos/dependency-map.md) to see who calls whom; don't assume — verify against the actual dependency direction (e.g., `caseflow-be` is always the dependency root; FE/mobile never call `caseflow-ai-service` directly).
3. **Read relevant architecture.** [../docs/architecture/](../docs/architecture/) for the repos involved, plus the repo-specific agent file ([BACKEND.md](BACKEND.md), [FRONTEND.md](FRONTEND.md), [AI-SERVICE.md](AI-SERVICE.md), [MOBILE.md](MOBILE.md)).
4. **Read relevant contracts.** [../contracts/api/](../contracts/api/) and [../contracts/events/](../contracts/events/) for the endpoints/events involved; [../repos/integration-map.md](../repos/integration-map.md) for the cross-repo endpoint inventory.
5. **Read relevant ADRs.** [../decisions/](../decisions/) — do not reverse a documented decision without superseding it explicitly.
6. **Determine implementation order.** `caseflow-be` (or whichever repo owns the contract) first, then its clients. Do not build a frontend/mobile feature ahead of a backend contract that doesn't exist yet — see the roadmap's dependency notes ([../docs/product/roadmap.md](../docs/product/roadmap.md)).
7. **Implement repository-local changes**, one repository at a time, respecting that repository's own module boundaries ([../repos/backend/module-map.md](../repos/backend/module-map.md), etc.).
8. **Update contracts if required.** Any DTO/enum/permission-code/event-payload shape change is a contract change — update [../contracts/api/README.md](../contracts/api/README.md) or [../contracts/events/README.md](../contracts/events/README.md) and the relevant `repos/<name>/` doc **in the same change**, not as follow-up work.
9. **Validate dependent repositories.** If a contract changed, check every repo listed as a consumer in [../repos/integration-map.md](../repos/integration-map.md) for breakage, even if you don't have write access to fix it there — at minimum, flag it.
10. **Run tests** in every repository you touched. Do not skip a repository's test suite because "it's probably fine."
11. **Update Central Brain.** Any cross-repository architectural fact that changed (a new dependency, a new integration, a contract shape, a domain concept) must be reflected here in the same change — see [GLOBAL.md](GLOBAL.md)'s multi-agent coordination rule.
12. **Report all affected repositories** in the PR/task description, even ones you didn't modify but verified were unaffected.

## Breaking vs. non-breaking changes

- **Breaking API change** (removed/renamed field, changed type, removed enum value, changed required→optional or vice versa, changed permission code an existing feature relied on): must be called out explicitly in the contract doc and the PR description, and requires an ADR if it reflects an architectural decision (not just a shape fix). Coordinate deployment order — `caseflow-be` must ship before dependent clients if the change is additive-then-cleanup; simultaneously (or backend-first with backward compatibility) if it's a hard break.
- **Non-breaking API change** (new optional field, new endpoint, new enum value clients can ignore): still update the contract doc, but no special coordination required beyond the normal PR.
- **Event change** (new Kafka topic, changed payload shape, changed producer/consumer default-enabled state): update [../contracts/events/README.md](../contracts/events/README.md). Given both Kafka producer (`caseflow-be`) and consumer (`caseflow-ai-service`) currently default to disabled, enabling one side without the other is a deployment misconfiguration — flag this explicitly in any change that touches `caseflow.ai.async.enabled`.
- **Database/migration change**: Flyway-only, forward-fix (no destructive `migrate down`) per current tooling in both `caseflow-be` and `caseflow-ai-service`. A schema change that removes/renames a column a DTO exposes is a breaking API change too — treat it as both.
- **Authentication/authorization change**: any change to JWT claim shape, permission-code catalog, or `ticketScope` semantics is breaking for every client. `caseflow-ai-service` currently has no auth at all (by design, [ADR-0002](../decisions/0002-ai-service-no-auth-p1.md)) — enabling its scaffolded internal-API-key auth is itself a cross-repo change requiring `caseflow-be`'s client to start sending the header in the same change, not a follow-up.
- **AI behavior change** (model swap, prompt change, RAG vs. non-RAG for an endpoint): update [../docs/architecture/ai-service.md](../docs/architecture/ai-service.md) and [../repos/ai-service/README.md](../repos/ai-service/README.md) — these currently document, per endpoint, which of the 4 AI-assist capabilities are genuinely RAG (similar-cases, policy-guidance) versus plain completion (summary, reply-draft); don't let that distinction go stale.
- **Mobile compatibility**: mobile follows the exact same auth/API contract as web FE with no mobile-only flow — a backend change that would require a mobile-only auth variant (e.g., OIDC/PKCE) is out of scope unless a new ADR authorizes it.
- **Rollback considerations**: no automated rollback pipeline exists in any repository yet — rollback is a manual re-deploy of a known-good image tag (see [../docs/operations/deployment.md](../docs/operations/deployment.md)). A migration that isn't backward-compatible with the previous image tag blocks rollback; consider that before shipping a breaking migration.
