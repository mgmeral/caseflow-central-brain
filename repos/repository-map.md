# Repository Map

## Purpose
One-page ground-truth summary of what each CaseFlow repository is, in current implementation terms. Verified against each repository's actual source as of **2026-09-12** (backend/frontend/AI-service/mobile inspection pass). Where a claim could not be directly verified from source, it is marked `TODO: Verify` or `UNKNOWN: Not established in source repository`.

## What belongs here
Per-repository facts: language, framework, runtime, architecture style, major modules, dependencies, data stores, external integrations, testing approach, configuration approach, maturity. Cross-repository relationships belong in [dependency-map.md](dependency-map.md) and [integration-map.md](integration-map.md) instead.

---

## caseflow-be (Backend)

- **Responsibility:** Single source of truth for tickets, customers, users/roles, email ingestion/dispatch, SLA tracking, tagging, automation rules, and third-party integrations (Jira, Slack/Teams/webhooks). Owns all relational and email-document data.
- **Language / framework:** Java 21, Spring Boot 3.3.0.
- **Architecture style:** **Modular monolith** — one deployable Spring Boot application (one `pom.xml`, one Docker image, one `k8s/deployment.yaml`), internally organized into ~15 domain packages under `com.caseflow`. **Not microservices** — no service registry/discovery, no API gateway config in-repo, no `spring-cloud` dependency. It does act as one of two cooperating services in the broader system (the other being `caseflow-ai-service`, called via REST + optional Kafka).
- **Major modules (IMPLEMENTED):** `ticket`, `customer`, `identity` (users/roles/groups), `workflow` (assignment/transfer/state/history), `note`, `email` (ingestion/routing/threading/dispatch/templates/scheduled-send), `storage` (object storage abstraction), `sla` (policy config + breach checking), `automation` (rules engine — `UNKNOWN: exact trigger/action model`, not inspected in depth), `notification` (in-app user notifications), `integration/jira` (Jira issue creation/linking), `integration/notification` (Slack/Teams/generic webhook channel config + delivery), `ai` (client to `caseflow-ai-service`: REST + optional Kafka producer, circuit breaker, retry, Postgres response cache), `auth` (JWT), `common` (cross-cutting: exceptions, security, config, dev seed data).
- **Full module detail:** [backend/module-map.md](backend/module-map.md).
- **Major dependencies:** Spring Web/Security/Validation/Data JPA/Data MongoDB, JJWT 0.12.6, Flyway, MinIO SDK 8.5.11, springdoc-openapi 2.5.0, Micrometer + Prometheus registry, Bucket4j 8.10.1 (rate limiting), Resilience4j 2.2.0 (circuit breaker), Spring Retry, MapStruct, Lombok, `spring-kafka` (optional). **No Spring AI dependency** — AI integration is a hand-written REST/Kafka client, not an embedded LLM client. **No Redis dependency anywhere** (verified by repo-wide grep) — rate limiting and AI response caching are implemented without it (in-process Bucket4j buckets; a Postgres table).
- **Data stores:** PostgreSQL (primary relational store, 44 Flyway migrations, `ddl-auto=validate`), MongoDB (`EmailDocument` — full email bodies/attachments metadata only), MinIO/S3-compatible object storage (attachment binaries; local-filesystem alternative for plain `mvn spring-boot:run`). **Redis: not used.**
- **External integrations:** SMTP (per-mailbox), IMAP polling (incl. Microsoft 365/Entra app-only OAuth2 client-credentials flow), Jira REST API (`UNKNOWN: exact endpoints` — not inspected in depth), Slack/Teams/generic webhook delivery, outbound calls to `caseflow-ai-service` (REST, synchronous) and optionally Kafka (async ingestion lane, disabled by default).
- **Testing:** JUnit 5, Testcontainers (PostgreSQL + MongoDB) present in `pom.xml` and used by `CaseFlowIntegrationTest`/`TicketRepositoryIntegrationTest`; ~95 test files. CI (`.github/workflows/ci.yml`) comment claims "no real databases required," which is in tension with the Testcontainers dependencies present — `UNKNOWN: whether Testcontainers-based tests actually run in the default CI gate` (flagged for backend-team follow-up, not resolved by this pass).
- **Configuration:** `application.properties` (base) + `-dev`/`-prod` profile overrides, env-var driven with local defaults, k8s manifests (`namespace, configmap, secret, deployment, service, ingress, hpa`) plus local-only `k8s/infra/{postgres,mongo,minio}`.
- **Maturity:** Mature/production-shaped for its documented scope — 44 migrations, ~95 test files, structured error handling, security hardening (rate limiting, account lockout, security audit log, CORS, security headers). Known gap: `@Scheduled` jobs (IMAP poll, SLA breach check, email retry/dispatch, integration job worker, AI ingest retry) have no distributed-lock guard beyond DB-level `SKIP LOCKED`/`PESSIMISTIC_WRITE` claiming on the job-queue-shaped ones — a real risk if `k8s/hpa.yaml` ever scales this deployment beyond 1 replica.
- **No multi-tenancy:** `Customer` is a business entity (the external company whose users send tickets), not a SaaS tenant/isolation boundary — one shared Postgres/Mongo database for all customers, no `tenantId` discriminator anywhere.

## caseflow-fe (Frontend)

- **Responsibility:** Web UI for agents/admins/supervisors/viewers. Holds no domain data of its own.
- **Language / framework:** React 18.3.1 + TypeScript ~5.7.2, Vite 6. Package name `crm-fe` (repo/product is branded CaseFlow; the package.json name itself has not been renamed — `TODO: Verify` whether this is intentional).
- **Architecture style:** Client-rendered SPA, hand-written `fetch`-based API layer (no generated client, no axios), React Router v6, Zustand (auth/UI/filter state) + TanStack Query v5 (server state), Tailwind CSS, no UI kit.
- **Major modules/areas (IMPLEMENTED, real backend calls):** Tickets (list/detail/create/update/status/close/reopen/assign/transfer/tags/notes), Customers + Contacts, Email (thread view, compose/reply, scheduled send, mailbox admin, customer email settings, mail templates with live preview), Notifications (in-app, polling), Dashboard stats, Reports (per-customer + admin aggregate, client-side PDF export), User/Role/Group admin, Jira integration UI (status/create/retry/config/test), Notification-channel (Slack/Teams/webhook) admin UI, Ingress-event admin (retry/quarantine/release/process), AI Summary, AI Reply Draft.
- **Explicit stubs (PLANNED, not IMPLEMENTED in FE):** `/admin/sla-policy` page exists, is permission-gated, and looks like a real settings screen, but its source is a static explainer — **no SLA policy CRUD UI exists** despite the route. `ticketService.addPublicReply` always throws 501 (superseded by the real email-reply flow). AI "similar cases" and "policy guidance" are explicitly commented in source as "Phase 2 — not implemented until BE is ready," even though both are already implemented and callable on `caseflow-ai-service`/`caseflow-be`.
- **Known gap:** No access-token refresh flow — `refreshToken` is stored (from login) but never used to renew a session; any 401 immediately force-logs-out the user. This differs from `caseflow-mobil`, which does implement silent refresh-and-retry.
- **Full detail:** [frontend/README.md](frontend/README.md).
- **Testing:** Vitest 4 + Testing Library + jsdom, ~44 test files (services, hooks, components/pages). No E2E framework (no Cypress/Playwright).
- **Maturity:** Broadly complete for its documented feature surface; the SLA-policy and AI-Phase-2 gaps above are the main "looks done, isn't" traps for anyone judging completeness from the routing table alone.

## caseflow-ai-service (AI Service)

- **Responsibility:** Stateless AI orchestration for ticket summaries, reply drafts, similar-case retrieval, and policy guidance. No independent system-of-record data — indexes a derived copy of ticket/policy/template text for retrieval. Called only by `caseflow-be`.
- **Language / framework:** Java 21, Spring Boot 3.4.1, Spring AI 1.0.0-M6 (a **milestone/pre-GA** release — requires Spring Milestones/Snapshots Maven repos, not on Maven Central).
- **Architecture style:** Single Spring Boot module. `ChatClient` (Spring AI) → Ollama; `VectorStore` (Spring AI's Qdrant starter) → Qdrant. No Spring Security dependency at all.
- **AI capability status (IMPORTANT — verified per-endpoint):**
  - **Ticket summary** and **reply draft**: plain LLM completion calls (Ollama via `ChatClient`), built entirely from caller-supplied request fields. **No vector retrieval — these are NOT RAG**, despite being served from the same "AI assist" surface as the two below.
  - **Similar cases**: pure retrieval (Qdrant `VectorStore.similaritySearch`, filtered `sourceType=TICKET`) — no LLM call, no synthesis step.
  - **Policy guidance**: genuine retrieval-augmented generation — retrieves `sourceType=POLICY` chunks, injects into an LLM prompt, and **explicitly refuses to call the LLM if zero policy docs are retrieved** (anti-hallucination guard; returns a static "no policy found" response instead).
- **Ingestion:** Real, shared pipeline (`VectorIngestionService`) for TICKET/POLICY/TEMPLATE text via `POST /api/ai/ingest/documents` and `/tickets`, plus 3 optional Kafka consumers (see [dependency-map.md](dependency-map.md)). Chunking is a **naive fixed-size character splitter** (500 chars / 50 overlap) — not sentence/token-aware. No delete-before-reindex/dedup (re-syncing adds duplicate chunks). Every ingest attempt is tracked in a Postgres `ai_ingestion_job` table (the service's only PostgreSQL use).
- **Auth:** Disabled by default. An internal API-key filter (`X-Internal-Api-Key` header) exists in code but is gated off by `caseflow.ai.auth.enabled=false`. As shipped, every endpoint is unauthenticated — consistent with [ADR-0002](../decisions/0002-ai-service-no-auth-p1.md), except that P2 scaffolding now partially exists (not yet enabled, and `caseflow-be`'s client does not send the header even if it were enabled — `TODO: wire up when P2 auth is turned on`).
- **Caching:** None. Every AI-assist call re-invokes the LLM/vector store from scratch; no persisted history of generated summaries/drafts exists anywhere (stateless).
- **Multi-tenant retrieval isolation:** Structurally present (customerId/groupId metadata fields) but **not enforced** — no Qdrant payload indexing/filtering wired up yet; explicitly called out as deferred in source.
- **Dead code:** `RagSearchService` — a second, unused retrieval helper superseded by `RetrievalService`; never injected/called anywhere.
- **Full detail:** [ai-service/README.md](ai-service/README.md).
- **Testing:** JUnit 5, ~10 test classes covering controllers/services/chunking/sanitization; no test exists for the 3 Kafka consumers' actual message-handling logic (only the "async disabled" boot path is tested).
- **Maturity:** A real, working service for 2 of its 4 headline capabilities (similar-cases, policy-guidance are genuine RAG); the other 2 (summary, reply-draft) are simpler LLM-completion features that happen to live on the same "AI assist" surface. Async/Kafka and auth are both built but off by default.

## caseflow-mobil (Mobile)

- **Responsibility:** Read-mostly mobile client for agents/supervisors — same backend auth/API contract as the web frontend, no mobile-only auth flow.
- **Language / framework:** Expo SDK ~57, React Native 0.86.3, React 19.2.3, TypeScript ~6.0.3 (strict). Managed Expo workflow (no native `ios`/`android` project dirs, no `eas.json`).
- **Architecture style:** Feature-folder structure (`auth`, `cases`, `conversations`, `core`, `customers`, `inbox`, `notifications`, `settings`, `shared`, `types`), React Navigation (native-stack + bottom-tabs), Zustand (session, persisted to `expo-secure-store`) + TanStack Query (server state, incl. infinite-query pagination).
- **Major features (IMPLEMENTED, real backend calls):** Login/refresh/logout with automatic 401-retry-after-refresh (better than the web FE's no-refresh behavior), case list + case detail + SLA display, email-thread conversation view (read-only), customer list, admin-pool "Inbox" queue + stats (gated on `ADMIN_POOL_VIEW`), in-app notifications (15s polling, mark-read), dashboard stats, permission-based tab visibility.
- **Explicit gaps (verified NOT implemented, despite appearances):**
  - **Push notifications**: `EXPO_PUBLIC_ENABLE_PUSH` env flag exists but is never read by any code; no push library (`expo-notifications` etc.), no permission request, no device-token registration. In-app notifications are polling-only.
  - **Biometric unlock**: `expo-local-authentication` is used only to check device capability and drive a stored preference toggle; `authenticateAsync` is never called anywhere — it does not gate anything. The Profile screen's own copy admits "preference only."
  - **`EXPO_PUBLIC_ENABLE_AI`** flag exists (default `true`) but is never consumed anywhere in `src/` — there is no AI feature in the mobile app.
  - **Ticket workflow is entirely read-only** — no status change, assignment, or reply/compose UI, even though a `getCaseTransitions()` API hook is already defined and unused.
- **Full detail:** [mobile/README.md](mobile/README.md).
- **Testing:** Minimal — 3 test files (~4 test cases) total; no coverage of navigation, session/refresh logic, or any list/detail screen.
- **Maturity:** Functional MVP-scale app — every screen that exists calls real endpoints (no mock data in production code paths), but the feature surface is narrower than the env-flag names suggest.

## caseflow-central-brain (this repository)

No application code. Cross-repository source of truth per [../README.md](../README.md).
