# Environments

## Purpose
This document defines shared environment strategy across CaseFlow repositories.

## What belongs here
Environment names, intended usage, cross-repository mapping, and lifecycle expectations.

## Who should use this
Engineers and AI agents coordinating behavior across development, test, staging, and production-like environments.

## What should NOT be stored here
Environment secrets, credentials, or repository-specific local setup instructions — see each repo's guide under [repos/](../../repos/).

## Environment catalog
| Environment | Backend | Frontend | AI service | Mobile |
|---|---|---|---|---|
| **Local (no Docker)** | `mvn package` + local Postgres/Mongo on default ports | `npm run dev`, mock mode optional (`VITE_USE_MOCKS=true`) | `mvn package` + local Ollama/Qdrant | Expo dev server, `EXPO_PUBLIC_API_BASE_URL` → local backend |
| **Local (Docker Compose)** | `docker compose up -d` — app:8080, postgres:5432, mongo:27017, minio:9000/9001 | Docker image built with `VITE_API_URL`/`VITE_USE_MOCKS` build args | `docker-compose up --build` — service:8081, ollama:11434, qdrant:6333/6334. Kafka+Zookeeper are available behind a `--profile async` compose profile (matches `caseflow.ai.async.enabled`, off by default); an Open WebUI dev tool is available behind `--profile dev` (never expose — no auth). | n/a (device/simulator only) |
| **CI** | GitHub Actions: `mvn verify` (unit/`@WebMvcTest` only, no real DB) → Docker build | (repo-specific CI, not yet documented here) | (repo-specific CI, not yet documented here) | (repo-specific CI, not yet documented here) |
| **Kubernetes (dev/staging-shaped)** | `k8s/` manifests; `infra/{postgres,mongo,minio}` are **local-only**, not for production | TODO: Define this decision. | TODO: Define this decision. | n/a |
| **Production** | Managed Postgres/Mongo/object storage (not the in-cluster `infra/` manifests); real TLS via cert-manager; `SPRING_PROFILES_ACTIVE` unset (no dev seed data) | TODO: Define this decision. | Must stay unreachable except from `caseflow-be` (see [ADR-0002](../../decisions/0002-ai-service-no-auth-p1.md)) | TODO: Define this decision. |

## Promotion strategy
TODO: Define this decision. *(No formal environment promotion pipeline — e.g. dev → staging → prod gates — has been recorded yet.)*

## Data handling per environment
- Dev/local: `SPRING_PROFILES_ACTIVE=dev` seeds sample groups/users/customers/tickets/notes/email (idempotent) via `DevDataLoader`. Never enable in production.
- CI: no real database — all backend tests are unit/`@WebMvcTest`; no seed data, no persistent state between runs.
- Production: no seed data; real customer data only. `k8s/infra/*` backing services are explicitly local-dev-only and must be replaced by managed services in any real deployment (see [repos/backend/deployment-k8s.md](../../repos/backend/deployment-k8s.md#production-notes)).
