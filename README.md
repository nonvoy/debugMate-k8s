> [!WARNING]
> **Work in Progress**
>
> DebugMate is currently under active development. Features, APIs, data models, and architecture may change as the project evolves. The current implementation represents an early MVP focused on event ingestion, event processing, incident detection, and AI-assisted incident analysis.

## About DebugMate

DebugMate is an AI-powered observability assistant designed to help engineers understand production issues faster.

Modern distributed systems generate a large volume of logs, infrastructure events, and deployment notifications. DebugMate collects these events, normalizes and groups similar occurrences, detects potential incidents, and generates concise summaries to help engineers identify the most likely root causes.

The project is currently being developed as a collection of microservices:

- **debugMate-api** — Event ingestion API built with FastAPI.
- **debugMate-worker** — Background processing service responsible for event normalization, fingerprint generation, grouping, incident detection, and AI-powered analysis.

## debugMate-k8s

`debugMate-k8s` holds the Kubernetes manifests used to deploy DebugMate and its backing services. It currently provides a **local** environment intended for development against a single-node cluster (e.g. kind, minikube, or Docker Desktop Kubernetes).

All manifests are plain YAML applied with `kubectl` — there is no Kustomize or Helm layer yet. Everything is deployed into the `debugmate-local` namespace.

### Topology

```text
                 Ingress (nginx)
                 host: debugmate.test
                        |
                        v
                  debugmate-api  (Deployment, :8000)
                   |          \
            publish|           \ read incidents
          (Celery) |            \
                   v             v
   redis ──────────┘      debugmate-postgres
   (broker)                (incidents + associations)
                   |
                   v
            debugmate-worker (Deployment)
                   |
                   +--> opensearch (normalized events)
                   +--> debugmate-postgres (incidents)
```

- **debugmate-api** — Deployment + ClusterIP Service on port `8000`. Exposed externally through the Ingress. Has HTTP liveness (`/health/live`) and readiness (`/health/ready`) probes.
- **debugmate-worker** — Deployment only (no Service; it consumes from the broker). Liveness is checked with `celery -A app:celery_app inspect ping`.
- **redis** — Deployment + Service on `6379`, used as the Celery broker in local mode.
- **opensearch** — single-node StatefulSet (`opensearchproject/opensearch:2.18.0`) with the security plugin disabled, a 2Gi volume, and a Service on `9200`/`9600`.
- **debugmate-postgres** — StatefulSet (`postgres:17`) with a 2Gi volume and a Service on `5432`. Database name `debugmate`.

Service-to-service configuration (broker URL, OpenSearch URL, DB URL, credentials) is injected from the `debugmate-local-secrets` Secret.

## Directory Layout

```text
local/
  namespace.yaml              debugmate-local namespace
  secret.example.yaml         Template for the shared Secret (committed)
  secret.yaml                 Real Secret with local values (git-ignored)
  ingress.yaml               nginx Ingress: debugmate.test -> debugmate-api:8000
  debugmate-api/
    deployment.yaml
    service.yaml
  debugmate-worker/
    deployment.yaml
  redis/
    deployment.yaml
    service.yaml
  opensearch/
    statefulset.yaml
    service.yaml
  postgres/
    statefulset.yaml
    service.yaml
```

## Prerequisites

- A local Kubernetes cluster (kind, minikube, or Docker Desktop).
- An NGINX Ingress controller installed in the cluster (the Ingress uses `ingressClassName: nginx`).
- `kubectl` configured to point at the local cluster.

### Application images

The api and worker Deployments use locally-built images with `imagePullPolicy: Never`, so the images must already be present in the cluster's node:

```text
debugmate-api:local
debugmate-worker:local
```

Build them from their respective repositories and load them into the cluster. For example, with kind:

```bash
# in debugMate-api
docker build -f dev/Dockerfile -t debugmate-api:local .
kind load docker-image debugmate-api:local

# in debugMate-worker
docker build -f dev/Dockerfile -t debugmate-worker:local .
kind load docker-image debugmate-worker:local
```

(With minikube, build inside its Docker daemon via `eval $(minikube docker-env)` instead of loading.)

## Configuration

Shared configuration and credentials live in the `debugmate-local-secrets` Secret. The real `secret.yaml` is git-ignored; only `secret.example.yaml` is committed.

Create your local Secret from the example and fill in the values:

```bash
cp local/secret.example.yaml local/secret.yaml
```

The Secret provides:

```text
OPENSEARCH_INITIAL_ADMIN_PASSWORD   Admin password for OpenSearch
OPENSEARCH_URL                      http://opensearch:9200
CELERY_BROKER_URL                   redis://redis:6379/0
POSTGRES_USER                       Postgres user
POSTGRES_PASSWORD                   Postgres password
DB_URL                              postgresql://<user>:<password>@debugmate-postgres:5432/debugmate
```

## Deploy

Apply the namespace and Secret first, then the data stores, then the applications and Ingress:

```bash
kubectl apply -f local/namespace.yaml
kubectl apply -f local/secret.yaml

kubectl apply -f local/postgres/
kubectl apply -f local/opensearch/
kubectl apply -f local/redis/

kubectl apply -f local/debugmate-api/
kubectl apply -f local/debugmate-worker/
kubectl apply -f local/ingress.yaml
```

Or apply everything at once into the namespace:

```bash
kubectl apply -R -f local/ -n debugmate-local
```

Check rollout status:

```bash
kubectl get pods -n debugmate-local
```

## Accessing the API

The Ingress routes the host `debugmate.test` to the api Service. Map the host to your ingress controller's address (often `127.0.0.1` for local clusters):

```bash
echo "127.0.0.1 debugmate.test" | sudo tee -a /etc/hosts
```

Then the API is reachable at `http://debugmate.test/` (e.g. docs at `http://debugmate.test/docs`).

Alternatively, skip the Ingress and port-forward directly:

```bash
kubectl port-forward -n debugmate-local svc/debugmate-api 8000:8000
```

## Teardown

Delete everything by removing the namespace:

```bash
kubectl delete namespace debugmate-local
```

> [!NOTE]
> StatefulSet PersistentVolumeClaims are not always removed with the namespace depending on your cluster. If you want a clean slate, also check:
> ```bash
> kubectl get pvc -n debugmate-local
> ```
