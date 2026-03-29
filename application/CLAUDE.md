# CLAUDE.md — Application

## Purpose
This folder governs container image tagging, path routing configuration, and container traceability for all application workloads on this platform.

## Key Rules

1. **Tag every container image with the short git SHA: `TAG=$(git rev-parse --short HEAD)`.** Never push to `latest`, `stable`, or any mutable tag. Never reuse a tag. (ADR-009)

2. **Set `imagePullPolicy: Always` in every Helm chart.** Never use `imagePullPolicy: IfNotPresent` on this platform — Karpenter provisions nodes without a local image cache. (ADR-009)

3. **Define `ingress.basePath` once in `values.yaml`.** All five dependent locations (Ingress path rule, health check annotation, pod env var `APP_BASE_PATH`, liveness probe, readiness probe) must be derived from this single value via Helm templating. Never hardcode the path prefix in any template. (ADR-007)

4. **Read `APP_BASE_PATH` from the environment in application code and mount all routes on a prefixed router.** Set `APP_BASE_PATH` in test setup to match EKS behaviour — tests that omit it will pass against routes that don't exist in production. (ADR-007)

5. **All container images must be tagged with the git SHA of the commit that produced them.** The image tag must be recorded in the deployment record alongside the manifest version. ECR lifecycle policies must retain at least 30 image versions. (ADR-019)

6. **Never deploy a `latest`-tagged image to staging or production.** (ADR-009, ADR-019)

## ADR Index

| ADR | Title | One-line Summary |
|-----|-------|-----------------|
| [ADR-007](./ADR-007-app-base-path-ssot.md) | APP_BASE_PATH as Single Source of Truth for Path Routing | Define path prefix once in values.yaml; derive all five dependent locations from it |
| [ADR-009](./ADR-009-git-sha-image-tags.md) | Git SHA Image Tags for Container Images | Tag images with git SHA — never latest; set imagePullPolicy: Always |
| [ADR-019](./ADR-019-container-image-traceability.md) | Container Image Traceability | Git SHA tagging and deployment record traceability for all platform container images |

## When to Read These ADRs

- Before writing any Dockerfile, CI build step, or Helm chart image section → read ADR-009, ADR-019
- Before writing any Kubernetes Ingress, Deployment, or probe configuration → read ADR-007
- Before writing application route registration code → read ADR-007
- Before writing any ECR lifecycle policy → read ADR-019
- When debugging routing errors (404s, health check failures) in EKS → read ADR-007
