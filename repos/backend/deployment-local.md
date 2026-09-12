# CaseFlow — Local Docker Compose Deployment

## Prerequisites

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) (includes Docker Compose v2)
- Java 21 + Maven installed locally if you want to build/run without Docker

---

## Quick Start

```bash
# 1. Clone or navigate to the project
cd caseflow

# 2. (Optional) Copy and customise environment overrides
cp .env.example .env

# 3. Start everything
docker compose up -d

# 4. Verify the app is healthy
curl http://localhost:8080/actuator/health
# Expected: {"status":"UP",...}

# 5. Open Swagger UI
# http://localhost:8080/swagger-ui.html
```

The stack includes:

| Service       | Host:Port       | Notes                              |
|---------------|-----------------|-------------------------------------|
| App           | localhost:8080  | Spring Boot API + Swagger UI       |
| Postgres      | localhost:5432  | Database `caseflow`                |
| MongoDB       | localhost:27017 | Database `caseflow`                |
| MinIO API     | localhost:9000  | S3-compatible object storage       |
| MinIO Console | localhost:9001  | Web UI — `minioadmin / minioadmin123` |

---

## Environment Variables

All variables have safe defaults. Create a `.env` file (copy from `.env.example`) to override:

| Variable | Default | Description |
|----------|---------|-------------|
| `APP_PORT` | `8080` | Host port for the application |
| `POSTGRES_PORT` | `5432` | Host port for PostgreSQL |
| `MONGO_PORT` | `27017` | Host port for MongoDB |
| `MINIO_PORT` | `9000` | Host port for MinIO API |
| `MINIO_CONSOLE_PORT` | `9001` | Host port for MinIO web console |
| `POSTGRES_USER` | `caseflow` | PostgreSQL username |
| `POSTGRES_PASSWORD` | `caseflow_dev` | PostgreSQL password |
| `MINIO_ACCESS_KEY` | `minioadmin` | MinIO root user |
| `MINIO_SECRET_KEY` | `minioadmin123` | MinIO root password |
| `MINIO_BUCKET` | `caseflow` | MinIO bucket name |
| `SPRING_PROFILES_ACTIVE` | *(empty)* | Set to `dev` to load seed data |
| `CORS_ORIGINS` | `http://localhost:3000,http://localhost:5173` | Allowed CORS origins |
| `LOG_LEVEL_APP` | `INFO` | Log level for `com.caseflow.*` |

---

## Seed Data (optional)

To pre-populate the database with sample groups, users, customers, tickets and notes:

```bash
SPRING_PROFILES_ACTIVE=dev docker compose up -d
```

Or add `SPRING_PROFILES_ACTIVE=dev` to your `.env` file.

Seed data is idempotent — it checks if tickets already exist before inserting.

---

## Authentication

The app uses **JWT Bearer authentication**. Obtain a token via `POST /api/auth/login`, then pass it as `Authorization: Bearer <token>`.

Default dev credentials (requires `SPRING_PROFILES_ACTIVE=dev` to seed):

| Username | Password   | Role   |
|----------|------------|--------|
| alice    | admin123   | ADMIN  |
| bob      | agent123   | AGENT  |
| carol    | viewer123  | VIEWER |

Test with curl:
```bash
# 1. Login to get a token
TOKEN=$(curl -s -X POST http://localhost:8080/api/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"bob","password":"agent123"}' | jq -r .accessToken)

# 2. Use the token
curl -H "Authorization: Bearer $TOKEN" http://localhost:8080/api/tickets
```

---

## Ports and Endpoints

| Path | Description |
|------|-------------|
| `GET /actuator/health` | Health check (public, no auth needed) |
| `GET /actuator/info` | App info |
| `GET /swagger-ui.html` | Interactive API docs |
| `GET /v3/api-docs` | OpenAPI JSON |
| `GET /api/tickets` | Ticket list (requires VIEWER+) |

---

## Storage (File Uploads)

Uploaded attachments are stored in **MinIO** (`caseflow_minio_data` named volume). The app connects to MinIO inside the Docker network via `http://minio:9000`.

To browse stored files, open the MinIO console: [http://localhost:9001](http://localhost:9001) (default: `minioadmin / minioadmin123`)

To inspect via CLI:
```bash
# List objects in the caseflow bucket
docker exec caseflow-minio mc ls local/caseflow
```

---

## Logs

```bash
# Follow all services
docker compose logs -f

# App only
docker compose logs -f app

# Last 100 lines of app
docker compose logs --tail=100 app
```

---

## Useful Commands

```bash
# Start in background
docker compose up -d

# Start and load seed data
SPRING_PROFILES_ACTIVE=dev docker compose up -d

# Stop everything (keep volumes)
docker compose down

# Stop and wipe all data volumes
docker compose down -v

# Rebuild app image (after code changes)
docker compose build app
docker compose up -d app

# Connect to PostgreSQL
docker exec -it caseflow-postgres psql -U caseflow -d caseflow

# Connect to MongoDB
docker exec -it caseflow-mongo mongosh caseflow
```

---

## How It Works

1. `docker compose up -d` starts PostgreSQL and MongoDB first
2. Health checks poll both until they respond
3. The app container starts only after both DBs are healthy
4. On first start, Flyway runs `V1__init_schema.sql` to create the relational schema
5. MongoDB collections and indexes are created automatically by Spring Data MongoDB
6. If `SPRING_PROFILES_ACTIVE=dev`, the `DevDataLoader` seeds sample data

---

## Build the Image Manually

```bash
# From project root
mvn package -DskipTests
docker build -t caseflow:local .
```

Or let Compose handle it:
```bash
docker compose build
```

---

## Assumptions

- Docker Desktop v2.x or later (uses `docker compose` v2 plugin, not `docker-compose` v1)
- Ports 8080, 5432, 27017, 9000, 9001 are free on your machine (configure alternatives in `.env`)
