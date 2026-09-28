# Kubernetes Target

The target platform is K3s running on the existing Kubernetes-capable homelab infrastructure.

The first Kubernetes version should stay intentionally small. The goal is a predictable deployment model, not maximum abstraction.

## Namespace model

Use separate namespaces for environments:

```text
pvz-staging
pvz-production
```

This gives independent:

* configuration
* Secrets
* Services
* PVCs
* replica counts
* rollout history

## Frontend

Initial resource model:

```text
Deployment/pvz-game
Service/pvz-game
```

Characteristics:

* stateless
* port `8080`
* readiness probe: `/healthz`
* liveness probe: `/healthz`
* safe to run multiple replicas
* no PVC

A reasonable production starting point is two replicas once deployment automation is stable.

## Backend

Initial resource model:

```text
Deployment/pvz-backend
Service/pvz-backend
PersistentVolumeClaim/pvz-backend-data
```

Characteristics:

* port `3000`
* readiness probe: `/api/health`
* liveness probe: `/api/health`
* one replica initially
* `/data` mounted from a PVC

### Why one replica?

The current backend has three constraints:

1. SQLite is stored on the filesystem.
2. Uploaded level data is stored on the filesystem.
3. Web sessions currently use an in-memory session store.

Running multiple arbitrary backend replicas would therefore introduce shared-state and consistency problems.

The initial deployment should use:

```text
replicas: 1
strategy:
  type: Recreate
```

`Recreate` avoids two application instances writing the same SQLite/PVC state during a rollout.

### Path to horizontal scaling

Before increasing backend replicas, migrate shared state:

```text
SQLite          -> PostgreSQL
level files     -> shared/object storage
memory sessions -> Redis/database-backed sessions
```

Only after those changes should the backend use rolling multi-replica deployments.

## Configuration

Use ConfigMaps for non-sensitive values:

```text
PORT
GAME_URL
BACKEND_URL
ALLOWED_ORIGINS
feature flags
```

Use Secrets for sensitive values:

```text
SESSION_SECRET
GITHUB_CLIENT_SECRET
TURNSTILE_SECRET
OPENAI_API_KEY
Discord credentials
Bluesky credentials
PostHog credentials
```

The infrastructure repository should reference Secrets without storing secret values in Git.

## Kustomize layout

Kustomize is preferred initially over Helm because there are only two internal workloads and the main differences are environment-specific values.

Target layout:

```text
kubernetes/
├── base/
│   ├── namespace.yaml
│   ├── game/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── kustomization.yaml
│   ├── backend/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   ├── pvc.yaml
│   │   └── kustomization.yaml
│   └── kustomization.yaml
└── overlays/
    ├── staging/
    │   └── kustomization.yaml
    └── production/
        └── kustomization.yaml
```

Base resources define application contracts. Overlays define environment differences such as hostnames, replica counts, storage sizes and image versions.

## Traffic

The final edge path may use the existing Cloudflare/K3s setup, but the application must not depend on Cloudflare-specific behavior internally.

Conceptually:

```text
Internet
   |
Cloudflare / edge
   |
Ingress or Gateway
   |
   +--> game Service
   |
   +--> backend Service
```

TLS should terminate at the edge or cluster ingress layer. The containers themselves serve HTTP.

## Resource policy

Do not guess final resource limits before measurement. Start with conservative requests, observe actual usage, then set limits based on data.

The infrastructure repository should eventually add:

* CPU/memory requests
* CPU/memory limits
* PodDisruptionBudget for multi-replica frontend
* NetworkPolicy if the cluster policy requires it
* backup policy for backend data
