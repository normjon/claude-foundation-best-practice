# ADR-007: APP_BASE_PATH as Single Source of Truth for Path Routing

**Status:** Accepted
**Date:** 2026-03-01
**Scope:** Application + Infrastructure
**Tags:** routing, helm, configuration, single-source-of-truth, alb, kubernetes

---

## Context

AWS ALB does **not** strip path prefixes before forwarding requests to backend services.
When the ALB routes `/user-app/*` to a Kubernetes service, the pod receives the full path
including the prefix — `/user-app/health`, not `/health`.

This creates a configuration surface where the same path prefix value must be consistent
across at minimum five independent locations:

1. ALB Ingress path rule (`.spec.rules[*].http.paths[*].path`)
2. ALB health check annotation (`alb.ingress.kubernetes.io/healthcheck-path`)
3. Pod environment variable (`APP_BASE_PATH`) used by the application to mount routes
4. Kubernetes liveness probe path
5. Kubernetes readiness probe path

If any of these diverges, the result is a silent failure: the ALB marks targets unhealthy,
liveness probes fail, or routes return 404 — each indistinguishable from an application bug
without reading configuration carefully.

---

## Decision

A single value in `charts/<app>/values.yaml` (`ingress.basePath`) is the **only** place the
path prefix is defined. All five dependent locations are derived from this value via Helm
templating. No location may hardcode the path prefix independently.

---

## Options Considered

### Option 1: Hardcode in each location independently

**Description:** Set the path prefix explicitly in the Ingress spec, health check annotation,
deployment env var, and probe configs as separate literal values.

**Pros:**
- Explicit and readable — no template indirection

**Cons:**
- Any single change requires updating five files — high probability of partial update
- Divergence is silent: `/health` liveness probe passes locally; `/user-app/health` fails
  in production with no obvious error message
- Code review cannot easily verify consistency across five locations

---

### Option 2: Environment variable injected at deploy time

**Description:** Pass the path prefix as a `--set basePath=user-app` Helm override at
deploy time. Each location references `{{ .Values.basePath }}`.

**Pros:**
- Deploy-time flexibility — can change path without modifying values files

**Cons:**
- Requires the deployer to always pass the flag — forgetting it uses whatever the chart default is
- Path prefix is not visible in the chart source — harder to audit
- `--set` overrides are not recorded in version control

---

### Option 3: Single value in `values.yaml`, all locations derived via Helm ✓ *Selected*

**Description:** Define `ingress.basePath: user-app` once in `values.yaml`. All five
locations reference `{{ .Values.ingress.basePath }}` in Helm templates.

```yaml
# values.yaml
ingress:
  basePath: user-app

livenessProbe:
  httpGet:
    path: /{{ .Values.ingress.basePath }}/health   # derived
readinessProbe:
  httpGet:
    path: /{{ .Values.ingress.basePath }}/health   # derived
```

The Ingress template derives both the path rule and the health check annotation from the
same value. The Deployment template injects `APP_BASE_PATH` from the same source.

**Pros:**
- Change in one place propagates to all five locations atomically
- `helm template` or `helm lint` can verify consistency before deployment
- Code review of `values.yaml` alone is sufficient to understand routing configuration
- The pattern extends naturally to any number of additional locations

**Cons:**
- Requires Helm — not directly usable in raw Kubernetes YAML without a rendering step
- Template indirection adds one layer of abstraction

---

## Consequences

### Positive
- Routing misconfiguration is structurally impossible as long as the Helm template is correct
- A single PR changing `ingress.basePath` updates all five dependent locations simultaneously
- New engineers see one value to change when onboarding — no documentation required for this

### Trade-offs
- The application code must also read `APP_BASE_PATH` from the environment — the contract
  between Helm and the application must be maintained (see Implementation Rules)
- Local development has no `APP_BASE_PATH` set — routes are at `/health`, not
  `/user-app/health`. Tests must set `APP_BASE_PATH` explicitly (via `test-setup.ts` or
  equivalent) to match production behavior

---

## Implementation Rules

> These rules apply to all Claude agents and engineers building applications for this platform.

**Helm Chart (`values.yaml`):**

**DO:**
- Define the path prefix once:
  ```yaml
  ingress:
    basePath: <app-name>   # e.g., user-app
  ```
- Reference it in every dependent location via `{{ .Values.ingress.basePath }}`:
  ```yaml
  # templates/ingress.yaml
  - path: /{{ .Values.ingress.basePath }}
  annotations:
    alb.ingress.kubernetes.io/healthcheck-path: /{{ .Values.ingress.basePath }}/health

  # templates/deployment.yaml (env var injection)
  env:
    - name: APP_BASE_PATH
      value: /{{ .Values.ingress.basePath }}

  # values.yaml (probe paths — must include leading slash)
  livenessProbe:
    httpGet:
      path: /user-app/health   # set to match ingress.basePath at chart authoring time
  ```

**Application Code:**

**DO:**
- Read `APP_BASE_PATH` from the environment at startup:
  ```typescript
  const BASE_PATH = process.env.APP_BASE_PATH || '';
  app.use(BASE_PATH, router);
  ```
- Mount all routes on the `router` (not directly on `app`) so the prefix applies uniformly
- Set `APP_BASE_PATH` in test setup to match EKS behavior:
  ```typescript
  // test-setup.ts
  process.env.APP_BASE_PATH = '/user-app';
  ```

**DO NOT:**
- Hardcode `/user-app` in any Helm template — always derive from `{{ .Values.ingress.basePath }}`
- Register routes directly on `app` — they will be unreachable under the prefix
- Leave `APP_BASE_PATH` unset in tests — tests will pass against routes that don't exist in EKS

---

## Verification

```bash
# Confirm all five locations derive from the same value
helm template <release> ./charts/<app> | grep -E 'path:|APP_BASE_PATH|healthcheck-path'
# Expected: all path references contain the same prefix value

# Confirm ALB healthcheck path matches liveness probe path
kubectl get ingress <name> -o jsonpath='{.metadata.annotations.alb\.ingress\.kubernetes\.io/healthcheck-path}'
# Compare to:
kubectl get deployment <name> -o jsonpath='{.spec.template.spec.containers[0].livenessProbe.httpGet.path}'
# Both must be identical

# Confirm APP_BASE_PATH is injected into pods
kubectl exec <pod> -- env | grep APP_BASE_PATH
# Expected: APP_BASE_PATH=/user-app (with leading slash)

# Smoke test the full routing path
ALB=$(kubectl get ingress <name> -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
curl -s -o /dev/null -w "%{http_code}" http://${ALB}/<base-path>/health
# Expected: 200
```

---

## References

- [AWS ALB path-based routing](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-listeners.html#path-conditions)
- [Helm templating guide](https://helm.sh/docs/chart_template_guide/values_files/)
- Related: [ADR-006 — Shared ALB via IngressGroup](./ADR-006-shared-alb-ingressgroup.md)
