# Agent Guidance

## Purpose
Provides AI-agent-facing instructions for consistent behavior across CaseFlow repositories.

## What belongs here
Global rules, repository-specific context, and boundaries for backend, frontend, AI service, and mobile agents.

## Who should use this
AI agents and engineers authoring prompts or workflows that rely on shared context.

## What should NOT be stored here
Secrets, private credentials, or implementation code.

See `GLOBAL.md` and repository-specific files ([BACKEND.md](BACKEND.md), [FRONTEND.md](FRONTEND.md), [AI-SERVICE.md](AI-SERVICE.md), [MOBILE.md](MOBILE.md)) for scoped guidance, [CODE-REVIEW.md](CODE-REVIEW.md) for the cross-repository code review agent persona/checklist, and [CROSS-REPOSITORY-CHANGE.md](CROSS-REPOSITORY-CHANGE.md) for the standard workflow when a change spans more than one repository.

See [AGENT-OWNERSHIP.md](AGENT-OWNERSHIP.md) for which agent provider is responsible for which repository (Phase 1 of the AI Agent Orchestration system — manual, not automatic execution), and [../skills/](../skills/) + [../workflows/](../workflows/) for how a request becomes a task graph and how a task gets handed off.
