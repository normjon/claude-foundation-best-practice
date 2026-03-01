# ADR-004: arm64 (Graviton) as the Required Container Architecture

**Status:** Accepted
**Date:** 2026-03-01
**Scope:** Infrastructure + Application
**Tags:** compute, cost, arm64, graviton, karpenter, docker

---

## Context

Karpenter can provision EC2 instances of any CPU architecture. AWS Graviton (arm64) instances
deliver approximately 20% better price-performance than equivalent x86 (amd64) instances for
most workloads. However, when a cluster provisions arm64 nodes, container images must also be
built for `linux/arm64` — an amd64 image on an arm64 node crashes immediately with
`exec format error`. The architecture constraint must be enforced at both the infrastructure
layer (NodePool) and the application layer (Docker build) simultaneously, or the mismatch
is only discovered at runtime.

---

## Decision

All container images built for this platform **must** target `linux/arm64`. The Karpenter
NodePool **must** include `kubernetes.io/arch: arm64` as a required constraint. No amd64
instances may be provisioned by Karpenter for application workloads.

---

## Options Considered

### Option 1: x86 (amd64) only

**Description:** Build all images for the default `linux/amd64` platform; allow Karpenter
to provision standard Intel/AMD instances.

**Pros:**
- Default behavior — no special build flags required
- Maximum compatibility with third-party container images

**Cons:**
- Pays 20% premium over Graviton for equivalent compute
- Misses spot instance savings (Graviton spot instances are more abundant and cheaper)
- Not future-aligned: AWS continues to expand Graviton as its primary compute family

---

### Option 2: Multi-architecture (amd64 + arm64)

**Description:** Publish multi-arch manifests (`docker buildx`) targeting both platforms;
allow Karpenter to provision whichever is available.

**Pros:**
- Maximum flexibility for Karpenter instance selection
- Works with any image pulled from public registries

**Cons:**
- Doubles CI build time (two native builds or cross-compilation)
- Cross-compilation introduces subtle bugs for native extension modules (e.g., node-gyp, C bindings)
- Karpenter may still provision amd64 instances for cost or availability reasons, making
  behavior non-deterministic
- Third-party images in the cluster must also support both architectures

---

### Option 3: arm64 (Graviton) only ✓ *Selected*

**Description:** Build all images with `--platform linux/arm64`; constrain the NodePool to
`kubernetes.io/arch: arm64`. Bottlerocket's `@latest` AMI alias resolves the correct arm64
AMI automatically.

**Pros:**
- Consistent: every node and every image is arm64 — no runtime architecture mismatches possible
- Graviton spot instances are cheaper and more available than equivalent x86 spot
- Single build target simplifies CI pipelines
- Bottlerocket on Graviton provides a minimal, security-hardened OS image

**Cons:**
- Any third-party image that does not publish an arm64 variant cannot run on this cluster
  (must be verified before adopting a new dependency)
- Developers on Intel Macs must explicitly pass `--platform linux/arm64` to `docker build`

---

## Consequences

### Positive
- Predictable cost: every Karpenter-provisioned node is Graviton
- No architecture-mismatch crashes in production — the constraint is enforced before the pod runs
- Graviton instances consistently available in us-east-2 in all three AZs

### Trade-offs
- All future third-party workloads (databases, sidecars, operators) must be verified for
  arm64 support before deployment
- Local development on Intel machines requires `--platform linux/arm64` on every `docker build`

---

## Implementation Rules

> These rules apply to all Claude agents and engineers building applications for this platform.

**Infrastructure (Terraform / Karpenter):**

**DO:**
- Include `kubernetes.io/arch: arm64` as the **first** requirement in every NodePool spec:
  ```yaml
  requirements:
    - key: kubernetes.io/arch
      operator: In
      values: ["arm64"]
  ```
- Use the Bottlerocket `@latest` AMI alias in `EC2NodeClass` — it resolves the correct arm64
  AMI automatically per region:
  ```yaml
  amiSelectorTerms:
    - alias: bottlerocket@latest
  ```

**Application (Docker):**

**DO:**
- Always specify the target platform on build:
  ```bash
  docker build --platform linux/arm64 -t ${IMAGE}:${TAG} .
  ```
- Verify the built image architecture before pushing:
  ```bash
  docker inspect ${IMAGE}:${TAG} --format '{{.Architecture}}'
  # Must output: arm64
  ```
- Use `node:20-alpine` (or equivalent multi-arch base image) — Alpine publishes arm64 variants

**DO NOT:**
- Omit `--platform linux/arm64` from `docker build` — the default is the host architecture,
  which on Intel Macs is amd64
- Pull base images without verifying arm64 availability in their manifests
- Add `amd64` to NodePool `values` — this defeats the constraint and allows mixed provisioning

---

## Verification

```bash
# Confirm NodePool has arm64 constraint
kubectl get nodepool default -o jsonpath='{.spec.template.spec.requirements}' | jq .
# Expected: first requirement is kubernetes.io/arch = arm64

# Confirm all running EC2 nodes are arm64
kubectl get nodes -l karpenter.sh/nodepool=default \
  -o custom-columns='NAME:.metadata.name,ARCH:.status.nodeInfo.architecture'
# Expected: all nodes show arm64

# Verify image architecture (before push)
docker inspect ${IMAGE}:${TAG} --format '{{.Architecture}}'
# Expected: arm64

# Verify running pod has correct binary format
POD=$(kubectl get pods -l app.kubernetes.io/name=<app> -o jsonpath='{.items[0].metadata.name}')
kubectl exec $POD -- uname -m
# Expected: aarch64 (Linux representation of arm64)
```

---

## References

- [AWS Graviton overview](https://aws.amazon.com/ec2/graviton/)
- [Karpenter NodePool requirements](https://karpenter.sh/docs/concepts/nodepools/)
- [Bottlerocket AMI aliases](https://karpenter.sh/docs/concepts/nodeclasses/#specamiselectorterms)
- Related: [ADR-005 — Staged Terraform Apply](./ADR-005-staged-terraform-apply.md)
