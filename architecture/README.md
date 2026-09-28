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
| `pvz-infra` | **Planned:** Compose integration environment, Kubernetes/Kustomize manifests and environment overlays |

See [Repository Boundaries](repository-boundaries.md) for the ownership rules.

## Current implementation status

The documentation was verified against the active container implementation on **2026-09-28**.

| Component | Implementation branch / PR | Verified contract | CI state |
| --- | --- | --- | --- |
| Frontend | `feature/docker-runtime` / `pvz-game#3` | multi-stage build, unprivileged Nginx, port `8080`, `GET /healthz` | Docker image build passed |
| Backend | `feature/docker-runtime` / `pvz-backend#1` | Deno, non-root `deno`, port `3000`, `/data`, `GET /api/health` | Docker image build passed |

These changes are still in open application PRs at the time of this verification. They are therefore **implemented and build-validated, but not yet part of the target branches**.

The current Docker CI validates that an image can be built. It does **not yet** start the image and assert that its health endpoint becomes healthy. Runtime smoke tests belong in the next CI hardening step.

The following is **not yet implemented** and therefore must not be treated as current production state:

* GHCR publishing
* `pvz-infra`
* Docker Compose integration stack
* Kubernetes manifests
* staging/production cluster deployment
* automated image promotion
* container runtime smoke tests in CI

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
