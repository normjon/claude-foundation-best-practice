# ADR-006: Shared ALB via IngressGroup for Cluster-Wide Ingress

**Status:** Accepted
**Date:** 2026-03-01
**Scope:** Infrastructure + Application
**Tags:** networking, cost, alb, ingress, lbc, kubernetes

---

## Context

The AWS Load Balancer Controller (LBC) creates an Application Load Balancer (ALB) for each
Kubernetes `Ingress` resource by default. Each ALB carries a fixed cost (~$16–22/month)
regardless of traffic, plus an hourly LCU charge for active connections. In a cluster with
many services, per-service ALBs multiply this baseline cost linearly.

Additionally, managing per-app ALBs means each app has its own DNS hostname, making
cross-service routing or a unified entry point impossible without DNS gymnastics or a
separate reverse proxy.

The LBC's `IngressGroup` feature allows multiple `Ingress` resources to share a single ALB,
with each app owning a unique path prefix.

---

## Decision

All internet-facing application ingresses in this cluster **must** annotate with a shared
`group.name: platform`. A single ALB serves all applications. Each application owns an
exclusive path prefix (e.g., `/user-app`, `/admin-api`). No application may create a
dedicated ALB unless it has a specific, documented justification (e.g., WAF with distinct
rule sets, separate TLS certificate domain).

---

## Options Considered

### Option 1: One ALB per application (LBC default)

**Description:** Each `Ingress` resource creates its own ALB. No IngressGroup annotation.

**Pros:**
- Complete isolation between apps — one app's config cannot affect another's ALB
- Simpler per-app configuration (dedicated listener rules, separate security groups)
- Independent scaling of ALB LCU per app

**Cons:**
- Fixed cost per ALB (~$16–22/month baseline) scales linearly with application count
- Each app has its own DNS hostname — no unified entry point
- ALB provisioning is slow (~30–60s) — each new app requires a new ALB to be created

---

### Option 2: One ALB per namespace

**Description:** Each namespace gets a dedicated ALB. Apps within a namespace share the ALB.

**Pros:**
- Balances isolation (per-team/namespace) with cost efficiency
- Natural boundary follows team ownership

**Cons:**
- Namespace boundaries are not always aligned with cost or security domains
- Still multiplies ALB count with team/namespace growth
- Requires coordination of path prefixes within a namespace but not across

---

### Option 3: One ALB per cluster via IngressGroup ✓ *Selected*

**Description:** All `Ingress` resources annotate with the same `group.name: platform`.
The LBC creates one ALB and adds listener rules for each app's path prefix.

```yaml
annotations:
  alb.ingress.kubernetes.io/group.name: platform
```

**Pros:**
- Single fixed ALB cost regardless of number of applications deployed
- Unified DNS entry point — one hostname for all apps
- New app deployments add path rules to the existing ALB (no provisioning delay)
- Consistent security group and listener configuration across all apps

**Cons:**
- All apps share ALB limits (listeners, rules, certificates) — must monitor rule count at scale
- A misconfigured `Ingress` in one app could affect routing for all apps in the group
  (mitigated by requiring PR review for ingress changes)
- IngressGroup name must be consistent across all apps — a typo creates a second ALB

---

## Consequences

### Positive
- ALB cost is fixed at one unit regardless of how many applications are deployed
- New applications appear immediately after `kubectl apply` — no ALB provisioning wait
- Single DNS name simplifies client configuration, DNS management, and TLS certificate scope

### Trade-offs
- Path prefix must be unique per application — coordinate across teams before deploying
- The `group.name` value is a shared contract; changing it for one app creates a new ALB
  for that app (old ALB persists until all members of the old group are removed)
- If the shared ALB is deleted (e.g., by destroying EKS), all apps lose their entry point
  simultaneously — ALB is not tracked by Terraform, so deletion must be handled manually

---

## Implementation Rules

> These rules apply to all Claude agents and engineers deploying applications to this platform.

**DO:**
- Always include both IngressGroup annotations on every `Ingress` resource:
  ```yaml
  annotations:
    alb.ingress.kubernetes.io/group.name: platform
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/healthcheck-path: /<app-base-path>/health
  ```
- Register the app's path prefix in team documentation before deploying to avoid collisions
- Derive the `healthcheck-path` annotation from the same value as `APP_BASE_PATH`
  (see ADR-007) — never hardcode a different value
- Tag public subnets with `kubernetes.io/role/elb = 1` in Terraform (VPC module) so LBC
  auto-discovers where to place the ALB

**DO NOT:**
- Omit the `group.name` annotation — this creates a dedicated ALB for the app
- Use `target-type: instance` (NodePort) — use `target-type: ip` for direct pod routing,
  which works with Fargate and is required for `target-type: ip` security group management
- Hardcode the ALB DNS hostname anywhere — retrieve it dynamically:
  ```bash
  kubectl get ingress <name> -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
  ```
- Assume the ALB hostname is stable across cluster recreation — it changes when the ALB
  is destroyed and recreated

---

## Verification

```bash
# Confirm IngressGroup annotation is present
kubectl get ingress <name> -o jsonpath='{.metadata.annotations.alb\.ingress\.kubernetes\.io/group\.name}'
# Expected: platform

# Confirm only one ALB exists for the cluster VPC
VPC_ID=$(terraform output -raw vpc_id)
aws elbv2 describe-load-balancers --region <region> \
  --query "LoadBalancers[?VpcId=='${VPC_ID}'].LoadBalancerName" --output text
# Expected: one ALB name

# Confirm all apps route through the same ALB
aws elbv2 describe-rules \
  --listener-arn $(aws elbv2 describe-listeners \
    --load-balancer-arn <alb-arn> \
    --query 'Listeners[0].ListenerArn' --output text) \
  --query 'Rules[*].Conditions[*].Values' --output text
# Expected: one rule per app path prefix
```

---

## References

- [AWS LBC IngressGroup documentation](https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/guide/ingress/annotations/#ingressgroup)
- [ALB pricing](https://aws.amazon.com/elasticloadbalancing/pricing/)
- Related: [ADR-007 — APP_BASE_PATH Single Source of Truth](./ADR-007-app-base-path-ssot.md)
- Related: [ADR-001 — IRSA](./ADR-001-irsa-over-node-instance-profiles.md)
