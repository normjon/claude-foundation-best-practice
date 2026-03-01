# ADR-008: Coordinated PDB, NodePool Budget, and Rolling Update Strategy

**Status:** Accepted
**Date:** 2026-03-01
**Scope:** Infrastructure + Application
**Tags:** availability, karpenter, kubernetes, disruption, rolling-update, ha

---

## Context

High availability for a Kubernetes workload depends on three independent disruption controls
that must work together. When they are configured independently without coordination, two
failure modes emerge:

**Deadlock during rolling updates:** If `maxSurge > 0`, Kubernetes tries to schedule a new
pod before terminating an old one. With strict zone-based pod anti-affinity and all AZ slots
occupied, there is nowhere to place the surge pod. The rollout hangs indefinitely.

**PDB violation during Karpenter consolidation:** Karpenter's consolidation replaces
underutilized nodes. If the NodePool allows multiple concurrent node replacements and the
PDB `minAvailable` is too permissive, Karpenter can evict pods from multiple AZs
simultaneously, violating the PDB and potentially causing an outage.

The three controls are:
1. **PodDisruptionBudget (PDB)** — guarantees a minimum number of pods remain available during
   voluntary disruptions (node drains, evictions)
2. **NodePool disruption budget** — limits how many nodes Karpenter can replace concurrently
3. **Deployment rolling update strategy** — controls how many pods are created/terminated
   during a `helm upgrade` or `kubectl rollout`

---

## Decision

All production workloads with replicated deployments **must** configure all three disruption
controls in coordination. For a 3-replica, 3-AZ deployment the canonical configuration is:
- PDB: `minAvailable: 2`
- NodePool: `budgets: [{nodes: "1"}]`
- Rolling update: `maxSurge: 0, maxUnavailable: 1`

---

## Options Considered

### Option 1: PDB only

**Description:** Define a PDB with `minAvailable: 2`. Leave NodePool and rolling update
strategy at defaults.

**Pros:**
- Simple — one resource to manage

**Cons:**
- Default `maxSurge: 25%` causes scheduling deadlock with strict anti-affinity
- Karpenter's default consolidation policy has no node-count budget — can drain multiple
  nodes simultaneously and violate the PDB between the drain decision and the enforcement

---

### Option 2: NodePool disruption budget only

**Description:** Set `budgets: [{nodes: "1"}]` in the NodePool. Rely on Karpenter to respect
pod scheduling constraints.

**Pros:**
- Karpenter handles disruption without a separate PDB resource

**Cons:**
- NodePool budget only limits Karpenter-initiated disruptions — rolling updates and
  manual node drains are not covered
- Does not prevent rolling update deadlock

---

### Option 3: All three controls coordinated ✓ *Selected*

**Description:** Configure PDB, NodePool budget, and rolling update strategy together so
that no single disruption mechanism can violate availability guarantees or cause scheduling
deadlock.

For 3 replicas across 3 AZs:

```
PDB:            minAvailable: 2      → at least 2 pods survive any voluntary disruption
NodePool:       nodes: "1"           → Karpenter replaces at most 1 node at a time
Rolling update: maxSurge: 0          → no new pod created before old one is removed
                maxUnavailable: 1    → one pod removed at a time (frees its AZ slot)
```

**Pros:**
- No deadlock: `maxSurge: 0` ensures a slot is freed before a new pod needs one
- No PDB violation: `nodes: "1"` limits Karpenter to one node drain at a time;
  `minAvailable: 2` ensures at least 2 pods survive that drain
- Rolling updates always succeed: one old pod terminates, freeing its AZ; new pod
  schedules into the vacated slot

**Cons:**
- Rolling updates are serial (one pod at a time) — slightly slower than parallel rollouts
- Three separate resources to keep in sync (PDB, NodePool, Deployment)
- If replica count changes, all three controls must be re-evaluated together

---

## Consequences

### Positive
- Rolling updates complete reliably without hanging or requiring manual intervention
- Karpenter consolidation never simultaneously evicts enough pods to violate the PDB
- The three controls together provide deterministic, verifiable availability guarantees

### Trade-offs
- `maxUnavailable: 1` means rolling updates process one pod at a time — for large replica
  counts this is slow. Acceptable for 3 replicas; may need adjustment for larger deployments
- The PDB is managed by Terraform (infrastructure-owned), while the Deployment strategy is
  Helm-owned (app-owned) — changing replica count requires coordinating across both repos

---

## Implementation Rules

> These rules apply to all Claude agents and engineers configuring application workloads.

**PodDisruptionBudget (managed by Terraform, in `app-resources.tf`):**

```hcl
resource "kubernetes_manifest" "user_app_pdb" {
  manifest = {
    apiVersion = "policy/v1"
    kind       = "PodDisruptionBudget"
    metadata   = { name = "user-app-pdb", namespace = "default" }
    spec = {
      minAvailable = 2
      selector = { matchLabels = { "app.kubernetes.io/name" = "user-app" } }
    }
  }
}
```

**NodePool disruption budget (managed by Terraform, in `karpenter.tf`):**

```yaml
disruption:
  consolidationPolicy: WhenEmptyOrUnderutilized
  consolidateAfter: 30s
  budgets:
    - nodes: "1"
```

**Deployment rolling update strategy (managed by Helm, in `values.yaml`):**

```yaml
replicaCount: 3
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 0
    maxUnavailable: 1
```

**DO:**
- Recalculate all three values whenever `replicaCount` changes — the constraints are
  interdependent
- Keep PDB `minAvailable` at `replicaCount - 1` to allow at least one pod to be evicted
- Keep NodePool `nodes: "1"` unless you have documented justification for concurrent drains

**DO NOT:**
- Set `maxSurge > 0` when using `requiredDuringSchedulingIgnoredDuringExecution` pod
  anti-affinity with one pod per AZ — scheduling deadlock is guaranteed
- Remove the PDB without also reviewing the NodePool budget — the two work as a pair
- Use `minAvailable: 100%` or `maxUnavailable: 0` — this prevents all voluntary disruptions
  including node upgrades and Karpenter consolidation

---

## Verification

```bash
# Confirm PDB exists and is healthy
kubectl get pdb user-app-pdb
# Expected: ALLOWED DISRUPTIONS = 1 (with 3 running pods)

# Confirm NodePool budget
kubectl get nodepool default -o jsonpath='{.spec.disruption.budgets}' | jq .
# Expected: [{nodes: "1"}]

# Confirm rolling update strategy
kubectl get deployment user-app -o jsonpath='{.spec.strategy}' | jq .
# Expected: maxSurge=0, maxUnavailable=1

# Trigger a rolling update and confirm it completes without hanging
helm upgrade user-app ./charts/user-app \
  --set image.tag=<new-tag> --wait --timeout 5m
# Expected: completes in <5 minutes without hanging
```

---

## References

- [Kubernetes PodDisruptionBudget](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/)
- [Karpenter disruption documentation](https://karpenter.sh/docs/concepts/disruption/)
- [Kubernetes rolling update strategy](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#rolling-update-deployment)
- Related: [ADR-004 — arm64 Architecture](./ADR-004-arm64-graviton-architecture.md)
