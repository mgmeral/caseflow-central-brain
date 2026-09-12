# Feature: Notification Channels (Slack / Teams / Webhook)

## Purpose
Let admins configure outbound delivery of system-originated notifications (e.g. SLA breaches, new high-priority tickets — exact trigger catalog `UNKNOWN`, not verified this pass) to Slack, Microsoft Teams, or a generic webhook, distinct from the in-app per-user Notification feature.

## Scope
Channel configuration CRUD, an "event catalog" the FE surfaces for selecting which events trigger delivery, and delivery itself via the backend's durable job queue. Does not cover in-app notifications (`UserNotification`/`NotificationController`) — see that as a separate concept in [docs/product/domain-model.md](../docs/product/domain-model.md).

## Repositories Affected
- caseflow-be — `integration/notification` module: `NotificationChannelConfigController`, `NotificationChannelConfig` entity, `{Slack,Teams,Webhook}NotificationJobProcessor` (via `IntegrationJob` queue)
- caseflow-fe — `channelIntegration.service.ts`, `ChannelIntegrationSettingsPage.tsx`
- caseflow-ai-service — not affected
- caseflow-mobil — not affected

## Contract Impact
`/api/admin/integrations/channels` (CRUD) + `/event-catalog`. Per-field request/response shapes are `TODO: Verify` — not documented in per-field detail anywhere in this repository yet; see [repos/integration-map.md](../repos/integration-map.md).

## Decision Dependencies
None recorded.

## Implementation Tasks
None active — see [tasks/active/](../tasks/active/).

## Rollout / Compatibility Notes
Same durable job-queue delivery mechanism as Jira (see [jira-integration.md](jira-integration.md)) — channel outages degrade to retryable failures, not lost notifications.

## Done Criteria
This feature appears functionally complete (real backend CRUD + delivery, real FE admin UI) as verified 2026-09-12; the exact event catalog and per-field contract shapes remain a documentation gap, not an implementation one.
