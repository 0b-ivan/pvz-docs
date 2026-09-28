# Container Runtime

Docker images are the deployment artifact for both application repositories. The images are designed so the same artifact can run locally, in CI and later in K3s.

## Frontend image

### Verified implementation

The current implementation uses:

```text
build:   node:22-bookworm-slim
runtime: nginxinc/nginx-unprivileged:1.27-alpine
port:    8080
```

The build passes `BUILD_SHA` into the existing `CF_PAGES_COMMIT_SHA` build stamp so the container build does not depend on Cloudflare Pages being present.

### Runtime contract

| Property | Value |
| --- | --- |
| Process | unprivileged Nginx |
| Container port | `8080` |
| Health endpoint | `GET /healthz` |
| Persistent storage | none |
| Runtime state | stateless |
| Horizontal scaling | safe |

The build is multi-stage: Node tooling performs the static build, and the runtime image contains only the generated/static site plus Nginx.

The frontend should eventually be deployable with a read-only root filesystem. Any future feature that needs server-side mutable state belongs in the backend, not in the Nginx container.

### Health contract

```http
GET /healthz

200 OK
ok
```

This endpoint only proves that the frontend web server is alive and serving requests. It does not prove that the backend is reachable.

## Backend image

### Verified implementation

The current implementation uses:

```text
runtime: denoland/deno:2.5.2
user:    deno
port:    3000
volume:  /data
```

The backend intentionally does not use a container-local `.env` file. Deployment systems inject environment variables directly.

### Runtime contract

| Property | Value |
| --- | --- |
| Process | Deno |
| Container port | `3000` |
| Health endpoint | `GET /api/health` |
| Persistent storage | `/data` |
| Runtime user | non-root `deno` |
| Horizontal scaling | not yet |

Default container paths:

```text
DB_PATH=/data/database.db
DATA_FOLDER_PATH=/data/levels
PUBLIC_FOLDER_PATH=/app/public
```

### Health contract

```http
GET /api/health

200 OK
{
  "status": "ok",
  "timestamp": "...",
  "version": "..."
}
```

Kubernetes readiness/liveness probes should consume this existing route.

## CI validation boundary

Both application Dockerfiles have passed `docker build` in GitHub Actions.

That proves:

* Dockerfile syntax/build stages are valid
* dependencies can be resolved during the image build
* the final image can be assembled

It does **not yet** prove:

* the container reaches a healthy runtime state
* the health endpoint responds after startup
* frontend-to-backend connectivity works
* persistence survives a backend restart

Those checks should be added as container smoke tests and later as Compose/infrastructure integration tests.

## Minimal backend configuration

A container must be able to boot without third-party credentials. Optional integrations are therefore disabled in the container baseline and enabled explicitly by deployment configuration.

Typical baseline:

```text
USE_GITHUB_AUTH=false
USE_TURNSTILE=false
USE_OPENAI_MODERATION=false
USE_REPORTING=false
USE_UPLOAD_LOGGING=false
DISCORD_PROVIDER_ENABLED=false
BLUESKY_PROVIDER_ENABLED=false
USE_POSTHOG_ANALYTICS=false
```

This is a safe **bootability default**, not a statement about which features production should use.

## Production configuration

At minimum, an environment deployment should explicitly define:

```text
GAME_URL
BACKEND_URL
ALLOWED_ORIGINS
SESSION_SECRET
```

If optional integrations are enabled, their credentials must be supplied through a secret-management mechanism.

Do not commit credentials into:

* Dockerfiles
* image layers
* Compose files
* Kustomize bases
* Git history

## Image naming

The intended registry is GitHub Container Registry:

```text
ghcr.io/0b-ivan/pvz-game:<tag>
ghcr.io/0b-ivan/pvz-backend:<tag>
```

Recommended immutable references:

```text
ghcr.io/0b-ivan/pvz-game:sha-<git-sha>
ghcr.io/0b-ivan/pvz-backend:sha-<git-sha>
```

Human-friendly version tags may also exist, but Kubernetes environments should ultimately pin a version that cannot silently change, preferably an image digest.

## Local integration

The planned `pvz-infra/compose.yaml` will run the published artifacts together:

```text
localhost:8080 -> pvz-game:8080
localhost:3000 -> pvz-backend:3000
                     |
                     +--> named volume -> /data
```

Compose is the integration test and developer convenience layer. Kubernetes remains the production target.
