---
icon: diagram-project
---

# Architecture & Operations

This section describes the target architecture for the `0b-ivan` Plants vs. Zombies browser-game fork and, where relevant, the implementation state as of **2026-09-28**.

The project is intentionally split into application code, backend services, documentation, and infrastructure. Application repositories own their build artifacts. Deployment configuration belongs in a separate infrastructure repository.

## System overview

```text
Browser / PWA
     |
     v
pvz-game
static frontend
     |
     | HTTPS / JSON API
     v
pvz-backend
Deno API
     |
     +--> SQLite database
     |
     +--> uploaded level data
```

The long-term runtime platform is **K3s/Kubernetes**. Docker is the packaging format and local validation layer, not the final orchestration model.

## Repositories

| Repository | Responsibility |
| --- | --- |
| `pvz-game` | Browser game, static assets, PWA/mobile runtime |
| `pvz-backend` | API, custom levels, persistence, optional external integrations |
| `pvz-docs` | Architecture, API, engine and operational documentation |
| `pvz-Infra` | Docker Compose integration, Kubernetes/Kustomize manifests, environment overlays and infra validation |

See [Repository Boundaries](repository-boundaries.md) for the ownership rules.

## Current implementation status

The documentation was verified against the active container implementation on **2026-09-28**.

| Component | Implementation branch / PR | Verified contract | CI state |
| --- | --- | --- | --- |
| Frontend | merged to `staging` via `pvz-game#3`, `#4`, `#5` | multi-stage build, unprivileged Nginx, port `8080`, `GET /healthz`, runtime `PVZ_BACKEND_URL`, GHCR publishing | build + runtime + backend integration + registry publish passed |
| Backend | merged to `main` via `pvz-backend#1`, `#2` | Deno, non-root `deno`, port `3000`, `/data`, `GET /api/health`, restart persistence, GHCR publishing | build + runtime + frontend integration + registry publish passed |
| Infrastructure | merged via `pvz-Infra#1` | digest-pinned Compose, Kustomize base/overlays, Traefik routing, PVC contract | render + real published-image Compose smoke test passed |

Both container implementations and their GHCR publishing workflows are now part of their target branches.

The smoke tests now start the actual containers and assert their runtime contracts. The backend test additionally restarts the container with the same Docker volume and verifies persisted data remains available.

The frontend/backend integration test now builds both application images together, starts them simultaneously, verifies the frontend runtime configuration points to the selected backend, calls the real `/api/levels` route with a browser Origin, and validates CORS plus preflight behavior.

## Published container baseline

The first registry-published baseline is:

```text
ghcr.io/0b-ivan/pvz-game:sha-51e3a642528275b9bfff79763c950563aae8a996
digest: sha256:ad652f80c6d7df441cdfcb298db328cfe67bc103ba794c60f10085c4d2af7cbd

ghcr.io/0b-ivan/pvz-backend:sha-627d39263f5599bf762d21ac700018c8647782f8
digest: sha256:81bb473de4ce31a2a2fdd34eb95f8e817fe47daea7a943eacb97371505bb5642
```

Future infrastructure should prefer the digest when pinning an immutable deployment.

Infrastructure implementation now exists in `0b-ivan/pvz-Infra` and has passed its first CI validation:

* digest-pinned Docker Compose integration stack
* real GHCR image pull and Compose startup smoke test
* Kubernetes base manifests
* staging and production Kustomize overlays
* Traefik Ingress model with same-origin `/api` routing
* backend PVC contract and `Recreate` strategy
* rendered-manifest validation that rejects floating/unconfigured images

The following is **not yet implemented** and must not be treated as current production state:

* actual K3s staging deployment
* real staging/production DNS hostnames
* TLS/Cloudflare edge wiring
* automated image promotion
* production deployment

## Design principles

1. **Application repositories build images; infrastructure deploys images.**
2. **Images are immutable.** Environments select an image tag or digest; containers are not modified after build.
3. **Configuration is external.** Environment-specific URLs, feature flags and secrets are injected at runtime.
4. **Frontend is stateless.** It can be horizontally replicated without shared storage.
5. **Backend is stateful for now.** SQLite and level files require persistent storage and initially constrain the backend to one replica.
6. **Kubernetes manifests do not live in application repositories.**
7. **Health checks are part of the application contract.** Infrastructure consumes them rather than inventing application-specific probes.

## Target request flow

```text
                       +--------------------+
Internet / Cloudflare | Ingress / Gateway  |
                       +---------+----------+
                                 |
               +-----------------+-----------------+
               |                                   |
               v                                   v
      +------------------+                +------------------+
      | pvz-game Service |                | backend Service  |
      +--------+---------+                +--------+---------+
               |                                   |
         +-----+-----+                             |
         |           |                             v
         v           v                    +------------------+
      game pod    game pod                | backend pod      |
                                         | replica: 1       |
                                         +--------+---------+
                                                  |
                                                  v
                                             PersistentVolume
                                             /data
```

The frontend can scale independently. The initial backend cannot safely be treated the same way because SQLite, file storage and in-memory sessions are process-local concerns. See [Kubernetes Target](kubernetes-target.md).
