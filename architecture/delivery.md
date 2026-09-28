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
update image reference in pvz-infra
   |
   +--> staging
   |
   +--> production after promotion
```

## Image publication

Each application repository should publish its own image.

Planned image names:

```text
ghcr.io/0b-ivan/pvz-game
ghcr.io/0b-ivan/pvz-backend
```

At minimum, publish a commit-addressable tag:

```text
sha-54ee421...
```

A release may additionally receive a semantic version tag.

Do not deploy mutable `latest` tags to production.

## Environment promotion

Promotion should happen in `pvz-infra`, not by rebuilding the application.

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
4. image publish after approved merge
5. optional vulnerability/SBOM checks

Infrastructure repository:

1. Kustomize render validation
2. schema validation
3. policy checks
4. staging deployment
5. smoke tests
6. explicit production promotion
