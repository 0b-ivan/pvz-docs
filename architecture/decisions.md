# Architecture Decisions

This page records the initial decisions so later implementation work does not silently change the architecture.

## ADR-001: Separate infrastructure repository

**Decision:** Deployment configuration will live in `pvz-infra`.

**Reasoning:** The game is a fork of an active upstream project. Keeping cluster-specific resources outside the fork reduces merge noise and keeps infrastructure changes independent from gameplay changes.

## ADR-002: Docker as artifact, Kubernetes as target

**Decision:** Both applications produce OCI/Docker images. K3s/Kubernetes is the long-term runtime.

**Reasoning:** The same image can be validated in CI, run with Docker/Compose and then deployed unchanged to K3s.

## ADR-003: Kustomize before Helm

**Decision:** Start with Kustomize bases and staging/production overlays.

**Reasoning:** The system initially contains only two internal workloads. Kustomize provides environment overlays without introducing chart templating before it is needed.

Revisit this decision if the platform becomes reusable across many installations or gains enough configurable components that a chart becomes materially simpler.

## ADR-004: Frontend is stateless

**Decision:** The frontend container stores no persistent application state.

**Consequence:** It can run multiple replicas and use rolling deployments.

Browser-local save state is client state, not container state.

## ADR-005: Backend starts as one replica

**Decision:** Run one backend replica with a PVC and `Recreate` deployment strategy.

**Reasoning:** SQLite, filesystem level storage and memory-backed sessions are not yet designed for arbitrary horizontal replicas.

## ADR-006: One backend persistence boundary

**Decision:** Mutable backend data is mounted below `/data`.

**Reasoning:** This simplifies Docker volumes, Kubernetes PVCs, backups and future migrations.

## ADR-007: Environment configuration is runtime configuration

**Decision:** URLs, feature flags and credentials are not baked into deployment-specific images.

**Reasoning:** One tested image should be promotable from staging to production.

## ADR-008: Existing application health routes are contracts

**Decision:** Infrastructure uses `/healthz` for the game and `/api/health` for the backend.

**Reasoning:** Probe behavior should be owned by the application/container contract and stay stable across orchestrators.
