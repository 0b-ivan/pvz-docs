# Build, Release and Promotion

Application delivery should be based on immutable container images. Git branches are development workflow tools; they should not be the runtime state of a Kubernetes environment.

## Desired pipeline

```text
code change
   |
   v
pull request
   |
   +--> lint/test
   +--> docker build
   |
   v
merge
   |
   v
publish immutable image to GHCR
   |
   v
update image reference in pvz-Infra
   |
   +--> staging
   |
   +--> production after promotion
```

## Image publication

Each application repository publishes its own image to GHCR.

Implemented image names:

```text
ghcr.io/0b-ivan/pvz-game
ghcr.io/0b-ivan/pvz-backend
```

The current workflows publish a full commit-addressable tag:

```text
sha-<full-git-sha>
```

Publish triggers:

* `pvz-game`: push to `staging`
* `pvz-backend`: push to `main`
* pull requests execute the same build path but do not log in or push packages

First published baseline:

```text
pvz-game
  tag:    sha-51e3a642528275b9bfff79763c950563aae8a996
  digest: sha256:ad652f80c6d7df441cdfcb298db328cfe67bc103ba794c60f10085c4d2af7cbd

pvz-backend
  tag:    sha-627d39263f5599bf762d21ac700018c8647782f8
  digest: sha256:81bb473de4ce31a2a2fdd34eb95f8e817fe47daea7a943eacb97371505bb5642
```

A release may additionally receive a semantic version tag later.

Do not deploy mutable `latest` tags to production.

## Environment promotion

Promotion should happen in `pvz-Infra`, not by rebuilding the application.

Example:

```text
staging:
  pvz-game@sha256:AAA
  pvz-backend@sha256:BBB

production:
  pvz-game@sha256:OLD
  pvz-backend@sha256:OLD
```

After validation, production is changed to the already-tested digests:

```text
production:
  pvz-game@sha256:AAA
  pvz-backend@sha256:BBB
```

This guarantees that staging and production use the same artifact.

## Branching

The current repositories are not yet fully standardized:

* `pvz-game` has `main` and a long-lived `staging` branch
* `pvz-backend` currently uses `main`

Long term, environment deployment should not require matching environment branches in every application repository. A cleaner model is:

* feature branches for development
* PR validation
* `main` as the releasable application line
* environment state controlled by `pvz-infra`

Until that transition is intentionally made, the existing game `staging` branch can continue as an integration branch.

## Rollback

Rollback is an infrastructure change, not a rebuild:

```text
current image digest -> previous known-good image digest
```

This makes rollback deterministic and fast.

For the backend, rollback must also consider database/schema compatibility. A container rollback cannot automatically reverse persisted data migrations.

## Required future CI stages

Application repositories:

1. lint/static checks
2. tests where available
3. Docker build
4. image publish after approved merge ✅
5. optional vulnerability/SBOM checks

Infrastructure repository:

1. Kustomize render validation ✅
2. Docker Compose render validation ✅
3. published-image Compose smoke test ✅
4. schema validation
5. policy checks
6. K3s staging deployment
7. staging smoke tests
8. explicit production promotion
