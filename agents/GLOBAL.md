# Global Agent Instructions

## Purpose
Defines baseline instructions that apply to all CaseFlow AI agents.

## What belongs here
Shared rules, terminology, source-of-truth usage expectations, and cross-repository coordination guidance.

## Who should use this
All agents working with `caseflow-be`, `caseflow-fe`, `caseflow-ai-service`, and `caseflow-mobile`.

## What should NOT be stored here
Repository-specific implementation instructions that belong in service-specific agent files ([BACKEND.md](BACKEND.md), [FRONTEND.md](FRONTEND.md), [AI-SERVICE.md](AI-SERVICE.md), [MOBILE.md](MOBILE.md)).

## Baseline
- Read this repository (`caseflow-central-brain`) before making cross-repository assumptions. **No other repository should contain its own architecture/rules/prompt `.md` files** — all of that content lives here, under `repos/<name>/` for repo-internal detail and `docs/`/`contracts/`/`decisions/` for cross-repo agreements. If you find a stray `.md` file reappearing in an app repo, migrate its content here and leave only a one-line pointer README behind, per the pattern in [repos/backend/](../repos/backend/), [repos/frontend/](../repos/frontend/), [repos/ai-service/](../repos/ai-service/), [repos/mobile/](../repos/mobile/).
- Follow contracts ([contracts/](../contracts/)) and ADRs ([decisions/](../decisions/)) before proposing interface changes.
- `caseflow-be` is the source of truth for domain data and the API contract; `caseflow-fe`, `caseflow-mobile`, and `caseflow-ai-service` are clients of it (`caseflow-ai-service` is called only by `caseflow-be`, never by FE/mobile directly — see [ADR-0002](../decisions/0002-ai-service-no-auth-p1.md)).
- Authorization is permission-code based (`permissionCodes`) — never gate behavior on role name in any repository.
- If required context is missing, write: `TODO: Define this decision.`

## Multi-agent coordination
- Before starting cross-repository work, check [tasks/active/](../tasks/active/) for related in-flight work from another agent, and [features/](../features/) for the feature's declared scope and repos affected.
- When you learn something that changes shared understanding (a contract shape, a domain rule, an architectural decision), update the matching file here in the same change — do not leave it only in your own conversation/PR description.
- Prefer linking to an existing doc over duplicating its content in a new one; if you find duplicated or conflicting information across files, resolve it and note which file is now authoritative.
