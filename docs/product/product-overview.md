# Product Overview

## Purpose
This document captures shared product context across all CaseFlow repositories.

## What belongs here
Product vision, user problems, business goals, and cross-repository product scope.

## Who should use this
Product, engineering, and AI agents that need a shared understanding of product intent.

## What should NOT be stored here
Implementation design, service topology, or code-level architecture — see [docs/architecture/](../architecture/).

## Product statement
CaseFlow is a ticket and mail-based case management system: customer emails become tickets (or link to existing ones), support agents triage/assign/work them through a defined lifecycle with SLA tracking and tagging, optionally link tickets to Jira, and an AI layer assists with summarization and reply drafting today (with similar-case lookup and policy guidance already built server-side but not yet surfaced in any client UI — see below).

## Core user outcomes
- **Support agents** can see every customer conversation (email + internal notes) as one ticket timeline, across web and mobile (mobile is read-only for the timeline; web supports full workflow actions).
- **Admins/supervisors** can configure mailboxes, customer email routing rules, users/groups, mail templates, Jira integration, and notification channels (Slack/Teams/webhook) without touching code. **SLA policy configuration is backend-complete but has no working frontend UI yet** (see [docs/architecture/frontend.md](../architecture/frontend.md)) — treat it as an operational/API-only capability today.
- **Agents** get AI-assisted summaries and reply drafts without leaving the ticket view (backend-mediated, never a direct AI-service call from the client). AI-assisted "similar cases" and "policy guidance" are implemented end-to-end on `caseflow-be`/`caseflow-ai-service` but not yet consumed by any client — this is a **PLANNED-for-the-frontend, IMPLEMENTED-on-the-backend** capability today, not a fully shipped user outcome.
- Inbound email is never lost or duplicated: idempotent ingestion, deterministic routing, and quarantine-for-review rather than silent drop for unmatched senders.
- Tickets can be tagged, linked to a Jira issue, and tracked against an SLA due date; supervisors can be notified via Slack/Teams/webhook on configured events.

## Success metrics
TODO: Define this decision. *(No agreed product metrics have been recorded yet — define alongside the next roadmap planning pass.)*

## Non-goals
- CaseFlow is not a general-purpose CRM — contact/customer management exists only to support ticket routing and ownership, not sales/marketing workflows.
- `caseflow-ai-service` is not a standalone product surface — it is only ever consumed through `caseflow-be`, never exposed to end users directly.
- Outbound channels other than email (chat widgets, SMS, social) are out of scope unless a future ADR says otherwise. (Slack/Teams/webhook integrations are *outbound notification delivery*, not a customer-facing conversation channel — do not conflate the two.)
- CaseFlow is not a multi-tenant SaaS product as currently built — `Customer` is a business-data concept inside one shared database, not an isolation boundary between separate tenants.
