# Deployment

## Purpose
This document describes shared deployment principles across CaseFlow repositories.

## What belongs here
Release responsibilities, deployment flow expectations, and cross-repository coordination points.

## Who should use this
Engineers and AI agents planning or coordinating deployments that affect multiple repositories.

## What should NOT be stored here
Repository-specific pipeline implementation details or environment secrets — see [repos/backend/ci-cd.md](../../repos/backend/ci-cd.md), [repos/backend/deployment-local.md](../../repos/backend/deployment-local.md), and [repos/backend/deployment-k8s.md](../../repos/backend/deployment-k8s.md) for the backend's full guides.

## Deployment model
- Each repository builds and ships its own container image independently; there is no single combined deployment artifact.
- `caseflow-be`: GitHub Actions → `mvn verify` → Docker build → push to GHCR on `main`/`master` (`GITHUB_TOKEN`, no extra secrets). Local: `docker compose up -d` (app + postgres + mongo + minio). K8s manifests under `k8s/` (namespace, configmap, secret, deployment, service, ingress, hpa, plus local-only `infra/{postgres,mongo,minio}`).
- `caseflow-ai-service`: `docker-compose up --build` (service + ollama + ollama-init + qdrant); optional `--profile dev` adds Open WebUI (dev-only, no auth — never expose).
- `caseflow-fe`: Vite build with `VITE_API_URL`/`VITE_USE_MOCKS` baked in at image build time; served by Nginx with SPA fallback.
- `caseflow-mobile`: Expo build (`npm run android` / `ios` / `web`); `EXPO_PUBLIC_*` env vars.

## Release coordination
- `caseflow-be` is the dependency root: deploy/verify it first when a change touches the API contract, then `caseflow-fe`/`caseflow-mobile`, then `caseflow-ai-service` if the ingest/query contract changed.
- A backend API contract change must be reflected in [contracts/api/README.md](../../contracts/api/README.md) and [repos/backend/frontend-contract.md](../../repos/backend/frontend-contract.md) **before** or **in the same change** as the frontend/mobile work that depends on it — do not let contract docs drift from what's deployed.
- `caseflow-ai-service` must never be deployed reachable from anything other than `caseflow-be` until P2 auth ships ([ADR-0002](../../decisions/0002-ai-service-no-auth-p1.md)).

## Rollback expectations
- `caseflow-be` uses Flyway migrations — rollbacks are forward-fix (new migration), not destructive `migrate down`, per current tooling.
- Docker image tags are `sha-<short-sha>`, `latest` (main/master only), and branch name — roll back a bad `main` deploy by redeploying the previous `sha-<short-sha>` tag rather than rebuilding from a reverted commit under time pressure.
- No automated rollback pipeline exists yet for any repository — rollback is a manual re-deploy of a known-good image tag.
