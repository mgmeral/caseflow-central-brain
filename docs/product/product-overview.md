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
CaseFlow is a ticket and mail-based case management system: customer emails become tickets (or link to existing ones), support agents triage/assign/work them through a defined lifecycle, and an AI layer assists with summarization, reply drafting, similar-case lookup, and policy guidance.

## Core user outcomes
- **Support agents** can see every customer conversation (email + internal notes) as one ticket timeline, across web and mobile.
- **Admins/supervisors** can configure mailboxes, customer email routing rules, and users/groups without touching code.
- **Agents** get AI-assisted drafts and policy guidance without leaving the ticket view (backend-mediated, never a direct AI-service call from the client).
- Inbound email is never lost or duplicated: idempotent ingestion, deterministic routing, and quarantine-for-review rather than silent drop for unmatched senders.

## Success metrics
TODO: Define this decision. *(No agreed product metrics have been recorded yet — define alongside the next roadmap planning pass.)*

## Non-goals
- CaseFlow is not a general-purpose CRM — contact/customer management exists only to support ticket routing and ownership, not sales/marketing workflows.
- `caseflow-ai-service` is not a standalone product surface — it is only ever consumed through `caseflow-be`, never exposed to end users directly.
- Outbound channels other than email (chat widgets, SMS, social) are out of scope unless a future ADR says otherwise.
