# CaseFlow Backend (`caseflow-be`)

Ticket and mail-based case management system.

**Stack:** Java 21 · Spring Boot 3.3 · PostgreSQL · MongoDB · Flyway · MapStruct · Spring Security (JWT)

## Local run (Docker Compose)

```bash
cp .env.example .env          # optional — all values have defaults
docker compose up -d           # starts app + postgres + mongo + minio
curl http://localhost:8080/actuator/health
```

Swagger UI: http://localhost:8080/swagger-ui.html
Seed credentials (dev profile): `alice / admin123` (ADMIN) · `bob / agent123` (AGENT) · `carol / viewer123` (VIEWER)

See [deployment-local.md](deployment-local.md) for the full guide, [deployment-k8s.md](deployment-k8s.md) for Kubernetes, and [ci-cd.md](ci-cd.md) for the GitHub Actions pipeline.

## Build without Docker

```bash
./mvnw package -DskipTests
java -jar target/caseflow-0.0.1-SNAPSHOT.jar
```

Requires local PostgreSQL on `localhost:5432` and MongoDB on `localhost:27017`.

## AI context index

These files are the source of truth for backend structure and rules — read them before generating backend code:

- [architecture.md](architecture.md) — modular monolith module list and layering rules
- [module-map.md](module-map.md) — full module boundaries, package placement guide, dependency direction
- [backend-rules.md](backend-rules.md) — general Spring Boot / layering / error-handling conventions
- [ticket-rules.md](ticket-rules.md) — ticket lifecycle and state transition rules
- [email-flow.md](email-flow.md) — email ingestion (webhook + IMAP), routing, threading, dispatch
- [storage-rules.md](storage-rules.md) — object storage abstraction rules
- [domain-specs/](domain-specs/) — per-entity field/rule specs (ticket, customer, contact, user, group, note, assignment, transfer, attachment, email-documents)
- [prompts/](prompts/) — reusable prompts for scaffolding controllers/services/entities/tests/etc.
- [skills/](skills/) — higher-level scaffolding recipes (CRUD domain, aggregate domain, document domain, workflow, API slice, module scaffold)
- [current-state.md](current-state.md) — snapshot of what is implemented and passing as of the last update
- [remaining-issues.md](remaining-issues.md) — resolved-issue history and open gaps by priority

## API documentation

- [frontend-contract.md](frontend-contract.md) — compact frontend contract (auth flow, all endpoints, response shapes, enums) — **primary FE-facing reference**
- [api-endpoints.md](api-endpoints.md) — full per-endpoint request/response examples
- [api-notes.md](api-notes.md) — endpoint overview, error codes, enum values, CORS, seed data

Always follow these rules when writing backend code.
