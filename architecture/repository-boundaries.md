# Repository Boundaries

The split between repositories is deliberate. Keeping deployment state out of the application repositories makes upstream synchronization, testing and environment promotion easier to reason about.

## `pvz-game`

Owns:

* browser game source
* HTML/CSS/JavaScript
* game assets
* PWA/mobile runtime
* frontend Dockerfile
* frontend image build CI
* frontend health endpoint exposed by the web server

Does **not** own:

* Kubernetes Deployments or Services
* production hostnames
* cluster namespaces
* TLS/Ingress configuration
* backend credentials

The frontend image is a static artifact. Runtime environment differences should be minimized. Where runtime configuration is required later, it should be exposed through a small configuration endpoint/file rather than rebuilding game logic for every environment.

## `pvz-backend`

Owns:

* Deno API
* API routes and validation
* SQLite schema/data access
* uploaded/custom level storage logic
* admin UI and authentication integration
* backend Dockerfile
* backend image build CI
* `/api/health`

Does **not** own:

* PVC definitions
* Kubernetes Secrets
* Ingress
* cluster-specific storage classes
* production credentials

### Persistent boundary

All mutable backend data should be reachable below:

```text
/data
├── database.db
└── levels/
```

This gives Docker and Kubernetes a single persistent mount boundary.

## `pvz-docs`

Owns:

* system architecture
* operational contracts
* API documentation
* engine documentation
* deployment decisions and constraints

Documentation should explicitly distinguish current state from planned state.

## `pvz-Infra`

The fourth repository owns deployment intent.

Implemented layout:

```text
pvz-Infra/
├── compose.yaml
├── .env.example
├── .github/workflows/validate.yml
├── kubernetes/
│   ├── base/
│   │   ├── game/
│   │   ├── backend/
│   │   └── ingress.yaml
│   └── overlays/
│       ├── staging/
│       └── production/
└── README.md
```

It owns:

* image references
* namespaces
* Deployments
* Services
* PVCs
* Ingress/Gateway resources
* ConfigMaps
* references to Secrets
* replica counts
* resource requests/limits
* staging/production differences

It should **not** contain copies of application source code.

## Dependency direction

```text
pvz-game --------+
                 |
                 +--> images --> pvz-Infra --> K3s
                 |
pvz-backend -----+

pvz-docs documents all three
```

Infrastructure may depend on published application images. Application repositories must not depend on the infrastructure repository to compile or run their unit/build tests.
