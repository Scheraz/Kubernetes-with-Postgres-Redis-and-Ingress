# order-api — Kubernetes Order Management Service

A production-shaped REST API deployed on Kubernetes, demonstrating multi-tier
architecture, externalized configuration, persistent storage, and ingress
routing — built and debugged end-to-end on a local Docker Desktop (kind)
cluster.

## Architecture

```
Browser/curl → order-api.local
  → NGINX Ingress Controller
    → order-api Service → order-api pod (Node.js/Express)
      → postgres-svc → Postgres pod (PVC-backed persistent storage)
      → redis-svc → Redis pod (cache layer)
```

## Stack

- **API**: Node.js + Express — REST endpoints for order management
- **Database**: PostgreSQL 16 (Alpine) — persistent storage via PVC
- **Cache**: Redis 7 (Alpine) — read-through cache with write invalidation
- **Orchestration**: Kubernetes (Docker Desktop / kind, 3-node cluster)
- **Ingress**: NGINX Ingress Controller

## Features

- `GET /health` — liveness probe (process-level, no external dependencies)
- `GET /ready` — readiness probe (verifies live Postgres + Redis connectivity)
- `POST /orders` — create an order (writes to Postgres, invalidates cache)
- `GET /orders` — list orders (Redis cache-first, 30s TTL, falls back to Postgres)

## Kubernetes objects

| Component | Objects |
|---|---|
| Postgres | Deployment, Secret (credentials), PVC (1Gi persistent storage), Service |
| Redis | Deployment, Service |
| API | Deployment, ConfigMap (non-secret config), Service, Ingress |

Config is fully externalized — the same container image runs unmodified
across environments; only the ConfigMap/Secret values change.

## Running locally

Prerequisites: Docker Desktop with Kubernetes enabled, `kubectl`, `kind` CLI.

```bash
# 1. Namespace
kubectl create namespace order-app
kubectl config set-context --current --namespace=order-app

# 2. Postgres
kubectl apply -f k8s/postgres/

# 3. Redis
kubectl apply -f k8s/redis/

# 4. Build and load the API image (Docker Desktop's kind nodes have a
#    separate image cache from the host — must be explicitly loaded)
docker build -t order-api:v1 .
kind load docker-image order-api:v1 --name desktop

# 5. API + Ingress
kubectl apply -f k8s/api/

# 6. Local DNS
echo "127.0.0.1 order-api.local" | sudo tee -a /etc/hosts

# 7. Test
curl http://order-api.local/health
curl http://order-api.local/ready
curl -X POST http://order-api.local/orders \
  -H "Content-Type: application/json" \
  -d '{"customer_name":"Test","item":"Widget","quantity":1}'
curl http://order-api.local/orders
```

## Notable engineering decisions

- **Non-root container user** — the Dockerfile runs the app as an unprivileged
  user, not root, reducing attack surface in a shared cluster.
- **Liveness vs. readiness probes** — liveness never checks external
  dependencies (a stalled DB shouldn't cause a restart loop); readiness does,
  so the pod is correctly pulled from the Service's endpoint list when it
  can't actually serve traffic.
- **Graceful shutdown** — the app handles `SIGTERM` to close the HTTP server
  and DB pool cleanly, avoiding dropped connections during rollouts.
- **Cache invalidation on write** — `POST /orders` explicitly invalidates the
  Redis cache to prevent serving stale data after a mutation.
- **Single Postgres replica by design** — a single unclustered Postgres
  instance is not safely horizontally scaled by increasing `replicas`; real
  HA would require a replication-aware operator.
- **`imagePullPolicy: Never`** — the API image is locally built, not pulled
  from a registry; `kind load docker-image` distributes it to all cluster
  nodes explicitly.

## Debugging log (real issues hit and resolved during this build)

- `CrashLoopBackOff` from a missing `POSTGRES_PASSWORD` — diagnosed via
  `kubectl logs`, fixed by wiring the Postgres Secret into the Deployment.
- YAML tab-character indentation error breaking `kubectl edit` saves —
  resolved by forcing the editor to expand tabs to spaces.
- Service `selector` name mismatch (`postgres` vs. `postgres-svc`) causing
  DNS resolution failures for dependent pods.
- Env var naming mismatch between the official Postgres image
  (`POSTGRES_PASSWORD`) and the `pg` npm driver (`PGPASSWORD`) — resolved by
  aliasing both in the Secret.
- `ErrImageNeverPull` on a multi-node kind cluster — the locally built image
  wasn't present on every node's containerd cache; fixed with
  `kind load docker-image`.
- PVC stuck in `Pending` due to a stray character in `claimName` — diagnosed
  via `kubectl describe pod` events.
- ConfigMap changes not propagating to a running pod — env vars are set at
  container creation time; fixed by forcing a pod restart after config
  changes.

