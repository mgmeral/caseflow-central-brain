# CaseFlow — CI/CD Guide

## Overview

CI runs on **GitHub Actions**. The pipeline builds, tests, and publishes a Docker image.

```
push/PR → test (Java 21 + Maven) → docker build (Dockerfile.ci) → push to GHCR (main only)
```

---

## Pipeline File

`.github/workflows/ci.yml`

### Jobs

| Job | Trigger | What it does |
|-----|---------|-------------|
| `test` | every push / PR | Checkout → JDK 21 → `mvn verify` → upload test reports + JAR |
| `docker` | after `test` passes | Download JAR → build image → push to GHCR (main/master only) |

### Job: `test`

- Uses `actions/setup-java@v4` with `cache: maven` — Maven dependencies are cached across runs
- Runs `mvn verify --batch-mode --no-transfer-progress` (compile + test + package)
- All 46 tests are unit / `@WebMvcTest` — no real database required
- Uploads Surefire XML reports as `surefire-reports` artifact (7-day retention)
- Uploads `target/caseflow-*.jar` as `app-jar` artifact (1-day retention, reused by docker job)

### Job: `docker`

- Downloads the pre-built JAR from `test` job
- Builds a Docker image using `Dockerfile.ci` (runtime-only, no internal Maven build)
- Tags image with: short commit SHA, `latest` (main/master only), branch name
- Pushes to `ghcr.io/<owner>/<repo>` **only on push to `main` or `master`**
- Uses `GITHUB_TOKEN` for GHCR auth — no extra secrets needed

---

## Branch Behavior

| Event | Test | Docker Build | Docker Push |
|-------|------|-------------|------------|
| Push to `main` / `master` | ✓ | ✓ | ✓ (ghcr.io) |
| Push to `develop` | ✓ | ✓ | ✗ |
| Pull Request to `main` / `master` | ✓ | ✓ (build only) | ✗ |

---

## Image Registry

Default: **GitHub Container Registry (GHCR)**

| Tag format | When applied | Example |
|-----------|-------------|---------|
| `sha-<short-sha>` | Every build | `sha-a1b2c3d` |
| `latest` | main/master push | `latest` |
| `<branch-name>` | Branch push | `develop` |

Full image name: `ghcr.io/<github-owner>/<repo-name>:latest`

Example pull:
```bash
docker pull ghcr.io/your-org/caseflow:latest
```

### Required Secrets for GHCR

GHCR push uses the built-in `GITHUB_TOKEN` — **no secrets setup required**.

The repository must be public, or the package visibility must be set to match your org settings. If the first push fails with a permission error, go to `Settings → Actions → General → Workflow permissions` and enable "Read and write permissions".

---

## Custom Registry (Docker Hub, AWS ECR, etc.)

To push to a different registry, update `.github/workflows/ci.yml`:

```yaml
# In the 'docker' job, replace the login and metadata steps:

- name: Log in to Docker Hub
  uses: docker/login-action@v3
  with:
    username: ${{ secrets.DOCKER_USERNAME }}
    password: ${{ secrets.DOCKER_PASSWORD }}

- name: Docker metadata
  id: meta
  uses: docker/metadata-action@v5
  with:
    images: your-org/caseflow          # Docker Hub
    # images: 123456789.dkr.ecr.us-east-1.amazonaws.com/caseflow  # AWS ECR
```

Add these secrets to the GitHub repo (`Settings → Secrets and variables → Actions`):

| Secret | Description |
|--------|-------------|
| `DOCKER_USERNAME` | Registry username |
| `DOCKER_PASSWORD` | Registry token or password |

---

## Dockerfiles

| File | Used by | Description |
|------|---------|-------------|
| `Dockerfile` | Local dev / `docker compose` | Multi-stage: builds JAR with Maven inside Docker |
| `Dockerfile.ci` | GitHub Actions | Runtime-only: copies pre-built JAR — faster CI builds |

`Dockerfile.ci` uses `Dockerfile.ci.dockerignore` which **does not exclude `target/`** so the JAR from the Maven job is available in the build context.

---

## Local Development

### One-command start

```bash
cp .env.example .env          # optional — all defaults are safe
docker compose up -d
```

### Services started

| Service | URL | Default credentials |
|---------|-----|-------------------|
| API | http://localhost:8080 | — |
| Swagger UI | http://localhost:8080/swagger-ui.html | admin / admin123 |
| Health check | http://localhost:8080/actuator/health | — (public) |
| PostgreSQL | localhost:5432 | caseflow / caseflow_dev |
| MongoDB | localhost:27017 | — (no auth) |
| MinIO API | localhost:9000 | minioadmin / minioadmin123 |
| MinIO Console | http://localhost:9001 | minioadmin / minioadmin123 |

### With seed data

```bash
SPRING_PROFILES_ACTIVE=dev docker compose up -d
```

### Stop / clean

```bash
docker compose down          # stop (keep data)
docker compose down -v       # stop + delete all volumes
```

---

## Build Without Docker

Requires: Java 21, Maven 3.9+, running PostgreSQL on :5432, MongoDB on :27017.

```bash
mvn package -DskipTests
java -jar target/caseflow-0.0.1-SNAPSHOT.jar
```

---

## Run Image Manually

```bash
docker run -d \
  --name caseflow \
  -p 8080:8080 \
  -e SPRING_DATASOURCE_URL=jdbc:postgresql://host.docker.internal:5432/caseflow \
  -e SPRING_DATASOURCE_USERNAME=caseflow \
  -e SPRING_DATASOURCE_PASSWORD=caseflow \
  -e SPRING_DATA_MONGODB_URI=mongodb://host.docker.internal:27017/caseflow \
  -e CASEFLOW_STORAGE_PROVIDER=local \
  ghcr.io/your-org/caseflow:latest
```

---

## Build Image Manually

### Using the self-contained Dockerfile (no local Maven required):

```bash
docker build -t caseflow:local .
```

### Using the CI Dockerfile (requires pre-built JAR):

```bash
mvn package -DskipTests
docker build -f Dockerfile.ci -t caseflow:local .
```

---

## Health Verification

```bash
curl http://localhost:8080/actuator/health
# Expected: {"status":"UP",...}
```

K8s probes (enabled in containerized deployments):
- Readiness: `GET /actuator/health/readiness`
- Liveness: `GET /actuator/health/liveness`

---

## Environment Variables

All variables have safe defaults for local development. Required for production:

| Variable | Default | Required in prod | Description |
|----------|---------|-----------------|-------------|
| `SPRING_DATASOURCE_URL` | localhost:5432 | Yes | PostgreSQL JDBC URL |
| `SPRING_DATASOURCE_USERNAME` | `caseflow` | Yes | DB username |
| `SPRING_DATASOURCE_PASSWORD` | `caseflow` | Yes | DB password |
| `SPRING_DATA_MONGODB_URI` | localhost:27017 | Yes | MongoDB connection URI |
| `CASEFLOW_STORAGE_PROVIDER` | `local` | Recommended | `local` or `minio` |
| `CASEFLOW_MINIO_ENDPOINT` | http://localhost:9000 | If minio | MinIO API endpoint |
| `CASEFLOW_MINIO_ACCESS_KEY` | `minioadmin` | If minio | MinIO root user |
| `CASEFLOW_MINIO_SECRET_KEY` | `minioadmin` | If minio | MinIO root password |
| `CASEFLOW_MINIO_BUCKET` | `caseflow` | If minio | Bucket name |
| `CASEFLOW_CORS_ALLOWED_ORIGINS` | localhost:3000, :5173 | Yes | Allowed FE origins |
| `SERVER_PORT` | `8080` | No | App port |
| `SPRING_PROFILES_ACTIVE` | (empty) | No | Set `dev` for seed data |
| `LOG_LEVEL_APP` | `INFO` | No | Log level for com.caseflow |

---

## CI Artifact Retention

| Artifact | Retention | Purpose |
|----------|-----------|---------|
| `surefire-reports` | 7 days | Test failure investigation |
| `app-jar` | 1 day | Passed to docker job (internal only) |

Test reports can be viewed in GitHub Actions → workflow run → Artifacts.

---

## Known Limitations

- **No integration tests**: All tests are unit / `@WebMvcTest`. Real DB connectivity is not tested in CI. Add Testcontainers for PostgreSQL + MongoDB to catch schema/query issues.
- **In-memory auth**: Default security config uses hardcoded dev passwords. Replace with DB-backed `UserDetailsService` before any non-local deployment.
- **First Docker build is slow**: Downloading all Maven dependencies inside Docker takes ~2–3 min. Subsequent builds reuse BuildKit GHA cache.
- **GHCR visibility**: If the package is private by default, pull requires authentication. Set the package to public in GitHub or configure an image pull secret.

---

## Recommended Next Steps

1. Add `CODEOWNERS` file if the repo moves to a team
2. Add a branch protection rule requiring `test` to pass before merge
3. Add Testcontainers integration tests (PostgreSQL + MongoDB)
4. Replace in-memory auth with `UserDetailsService` backed by the `users` table
5. Add a separate staging environment deploy step after the `docker` job
