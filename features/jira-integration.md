# Feature: Jira Integration

## Purpose
Let agents create and track a linked Jira issue directly from a CaseFlow ticket, for cases that need to be escalated into engineering's own tracker.

## Scope
Ticket-to-Jira-issue creation, linking, and retry of failed creation attempts; per-deployment Jira connection configuration (admin-side). Does not cover bidirectional sync (Jira status changes are not confirmed to flow back into CaseFlow — `UNKNOWN`, not verified this pass) or Jira comment mirroring.

## Repositories Affected
- caseflow-be — `integration/jira` module: `JiraController` (`/api/tickets/{ticketPublicId}/jira`), `JiraAdminController` (config/test), `JiraConfig`/`TicketJiraLink` entities, delivered via the shared `IntegrationJob` durable job queue (`JiraJobProcessor`)
- caseflow-fe — `jiraIntegration.service.ts`, `JiraIntegrationCard.tsx` (ticket-level status/create/retry), `JiraIntegrationSettingsPage.tsx` (admin config + connection test)
- caseflow-ai-service — not affected
- caseflow-mobil — not affected (no Jira UI exists)

## Contract Impact
Ticket identity uses the ticket's `publicId` (UUID), consistent with [ADR-0003](../decisions/0003-sequential-ticket-numbers-public-uuid.md). `UNKNOWN: exact outbound Jira REST API endpoints/base-URL config` — not verified against `integration/jira/service/*` in the 2026-09-12 pass; a follow-up documentation pass should read that package directly and populate [repos/integration-map.md](../repos/integration-map.md) with the confirmed detail.

## Decision Dependencies
None recorded. Worth an ADR if/when bidirectional sync or a specific Jira API version/auth mechanism becomes a load-bearing architectural choice.

## Implementation Tasks
None active — see [tasks/active/](../tasks/active/).

## Rollout / Compatibility Notes
Delivery goes through the same durable `IntegrationJob` queue used by Slack/Teams/webhook notifications (`SKIP LOCKED`/`PESSIMISTIC_WRITE` claiming, idempotency keys, attempt/backoff) — Jira API outages degrade to retryable job failures, not lost requests, and the FE surfaces a retry action.

## Done Criteria
This feature is functionally complete for one-way ticket→Jira-issue creation as verified; remaining work is documentation (exact Jira REST contract) rather than implementation, per this pass's findings.
