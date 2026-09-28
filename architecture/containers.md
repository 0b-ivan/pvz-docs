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

### Runtime backend configuration

The frontend no longer hard-codes the upstream backend hostname. The container accepts:

```text
PVZ_BACKEND_URL=https://api.example.invalid
```

At startup, the Nginx container generates `/runtime-config.js`. The game loads this file before `Cfunction.js`, and `$User.Server.URL` uses the configured backend URL.

`/runtime-config.js` is served with `no-store` and is explicitly excluded from service-worker caching. This allows one immutable frontend image to move between local, staging and production environments without rebuilding application assets.

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

Both application images now pass runtime smoke tests in GitHub Actions.

Frontend validation covers:

* Docker image build
* real container startup
* Docker HEALTHCHECK reaching `healthy`
* `GET /healthz`
* `GET /game/`

Backend validation covers:

* Docker image build
* real container startup
* Docker HEALTHCHECK reaching `healthy`
* `GET /api/health`
* SQLite database creation below `/data`
* persistence sentinel written to `/data`
* container removal and recreation with the same Docker volume
* health after restart
* persistence still present after restart

The backend image also prewarms the native SQLite library during image build. Runtime startup therefore no longer depends on downloading `libsqlite3.so` from GitHub.

The cross-repository integration test additionally proves:

* frontend and backend images can run at the same time
* frontend runtime configuration points to the selected backend
* `GET /api/health` succeeds
* `GET /api/levels` succeeds with the configured browser Origin
* CORS response headers are correct
* OPTIONS preflight succeeds

The tests still do **not** prove:

* custom-level upload/download behavior across restart as one end-to-end workflow
* browser gameplay behavior
* Kubernetes Service/Ingress/PVC behavior

Those belong in the planned Compose integration tests and later the `pvz-infra` staging deployment.

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

## Image publication

GitHub Container Registry is now the implemented application registry:

```text
ghcr.io/0b-ivan/pvz-game:<tag>
ghcr.io/0b-ivan/pvz-backend:<tag>
```

Every merge to the configured publish branch produces a full-SHA tag. Recommended immutable references:

```text
ghcr.io/0b-ivan/pvz-game:sha-<git-sha>
ghcr.io/0b-ivan/pvz-backend:sha-<git-sha>
```

Human-friendly version tags may also exist later, but Kubernetes environments should pin a version that cannot silently change, preferably an image digest.

First published artifacts:

```text
ghcr.io/0b-ivan/pvz-game:sha-51e3a642528275b9bfff79763c950563aae8a996
@sha256:ad652f80c6d7df441cdfcb298db328cfe67bc103ba794c60f10085c4d2af7cbd

ghcr.io/0b-ivan/pvz-backend:sha-627d39263f5599bf762d21ac700018c8647782f8
@sha256:81bb473de4ce31a2a2fdd34eb95f8e817fe47daea7a943eacb97371505bb5642
```

No `latest` tag is required for deployment.

## Local integration

The implemented `pvz-Infra/compose.yaml` runs the published artifacts together and is exercised in CI using the real GHCR digests:

```text
localhost:8080 -> pvz-game:8080
localhost:3000 -> pvz-backend:3000
                     |
                     +--> named volume -> /data
```

Compose is the integration test and developer convenience layer. The infra CI logs into GHCR, starts both digest-pinned images, waits for health, and verifies frontend/backend runtime connectivity. Kubernetes remains the production target.
