# CaseFlow — Kubernetes Deployment Guide

## Prerequisites

- `kubectl` configured against a cluster (minikube, kind, or real cluster)
- Docker image pushed to a registry accessible by the cluster
- `nginx` Ingress controller installed (for `k8s/ingress.yaml`)
- Metrics Server installed (for HPA)

---

## Directory Layout

```
k8s/
├── namespace.yaml       — Namespace: caseflow
├── configmap.yaml       — Non-sensitive env vars (DB URLs, storage config)
├── secret.yaml          — Credentials (DB password, MinIO keys) — DO NOT COMMIT with real values
├── deployment.yaml      — App Deployment with readiness/liveness probes
├── service.yaml         — ClusterIP Service on port 8080
├── ingress.yaml         — nginx Ingress, host: caseflow.local
├── hpa.yaml             — HorizontalPodAutoscaler (CPU 70%, Memory 80%)
└── infra/               — Local-only backing services (not for production)
    ├── postgres.yaml    — PostgreSQL 16 (Deployment + Service + PVC)
    ├── mongo.yaml       — MongoDB 7 (Deployment + Service + PVC)
    └── minio.yaml       — MinIO (Deployment + Service + PVC)
```

---

## Quick Start (minikube / kind)

### 1. Build and load the image

```bash
# Build
mvn package -DskipTests
docker build -t caseflow:local .

# Load into minikube (skip if using a registry)
minikube image load caseflow:local
# Or for kind:
# kind load docker-image caseflow:local
```

### 2. Create namespace

```bash
kubectl apply -f k8s/namespace.yaml
```

### 3. Edit secrets before applying

```bash
# Edit k8s/secret.yaml and replace all "change-me" values, then:
kubectl apply -f k8s/secret.yaml
```

### 4. Start backing services (local dev only)

```bash
kubectl apply -f k8s/infra/postgres.yaml
kubectl apply -f k8s/infra/mongo.yaml
kubectl apply -f k8s/infra/minio.yaml

# Wait for them to be ready
kubectl wait --namespace=caseflow --for=condition=ready pod -l app=postgres --timeout=60s
kubectl wait --namespace=caseflow --for=condition=ready pod -l app=mongo --timeout=60s
kubectl wait --namespace=caseflow --for=condition=ready pod -l app=minio --timeout=60s
```

### 5. Apply app manifests

```bash
kubectl apply -f k8s/configmap.yaml
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
kubectl apply -f k8s/ingress.yaml
kubectl apply -f k8s/hpa.yaml
```

### 6. Verify

```bash
# Check pods
kubectl get pods -n caseflow

# Follow logs
kubectl logs -n caseflow -l app=caseflow -f

# Port-forward if Ingress is not set up
kubectl port-forward -n caseflow svc/caseflow-svc 8080:8080

# Health check
curl http://localhost:8080/actuator/health
```

---

## Ingress Setup (optional)

Add `caseflow.local` to your `/etc/hosts`:

```
127.0.0.1  caseflow.local
```

For minikube, use `minikube tunnel` to expose the Ingress IP.

Then access: `http://caseflow.local/swagger-ui.html`

---

## Environment Variables

Non-sensitive vars live in `k8s/configmap.yaml`. Update the values for your environment:

| Key | Default in ConfigMap | Purpose |
|-----|----------------------|---------|
| `SPRING_DATASOURCE_URL` | `jdbc:postgresql://postgres-svc:5432/caseflow` | Postgres URL |
| `SPRING_DATA_MONGODB_URI` | `mongodb://mongo-svc:27017/caseflow` | MongoDB URI |
| `CASEFLOW_STORAGE_PROVIDER` | `minio` | `local` or `minio` |
| `CASEFLOW_MINIO_ENDPOINT` | `http://minio-svc:9000` | MinIO API endpoint |
| `CASEFLOW_MINIO_BUCKET` | `caseflow` | Bucket name |
| `MANAGEMENT_HEALTH_PROBES_ENABLED` | `true` | Enable `/health/readiness` and `/health/liveness` |
| `CASEFLOW_CORS_ALLOWED_ORIGINS` | `http://localhost:3000,...` | Allowed CORS origins |

Sensitive vars live in `k8s/secret.yaml`:

| Key | Purpose |
|-----|---------|
| `SPRING_DATASOURCE_USERNAME` | Postgres username |
| `SPRING_DATASOURCE_PASSWORD` | Postgres password |
| `CASEFLOW_MINIO_ACCESS_KEY` | MinIO root user |
| `CASEFLOW_MINIO_SECRET_KEY` | MinIO root password |

---

## Seed Data

To load sample data on startup, add `SPRING_PROFILES_ACTIVE: dev` to `k8s/configmap.yaml` before applying.

---

## Scaling

The HPA in `k8s/hpa.yaml` scales between 1–5 replicas based on CPU (70%) and memory (80%) utilization. Requires Metrics Server:

```bash
# minikube
minikube addons enable metrics-server

# kind / other
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

---

## Health Probes

The app exposes dedicated K8s probe endpoints (enabled via `MANAGEMENT_HEALTH_PROBES_ENABLED=true`):

| Path | Purpose |
|------|---------|
| `/actuator/health/readiness` | Readiness probe — app is ready to accept traffic |
| `/actuator/health/liveness` | Liveness probe — app is not deadlocked |
| `/actuator/health` | Full health (public) |

---

## Production Notes

- Replace `secret.yaml` with Sealed Secrets, External Secrets Operator, or Vault Agent
- Replace in-cluster Postgres/Mongo/MinIO with managed cloud services
- Replace in-memory auth with a DB-backed `UserDetailsService`
- Configure a real TLS certificate on the Ingress (`cert-manager` + Let's Encrypt)
- Set `imagePullPolicy: Always` and use a versioned image tag instead of `caseflow:local`

---

## Apply Everything at Once

```bash
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/secret.yaml          # edit credentials first
kubectl apply -f k8s/infra/
kubectl apply -f k8s/
```
