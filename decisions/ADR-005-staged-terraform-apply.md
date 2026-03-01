# ADR-005: Staged Terraform Apply for CRD-Dependent Resources

**Status:** Accepted
**Date:** 2026-03-01
**Scope:** Infrastructure
**Tags:** terraform, eks, karpenter, automation, iac, deployment

---

## Context

Terraform's `kubernetes_manifest` resource type validates the manifest structure against the
live Kubernetes API server at **plan time**, not only at apply time. When deploying Karpenter,
the `EC2NodeClass` and `NodePool` custom resources must be declared as `kubernetes_manifest`
objects. However, the CRDs that define these resource kinds (`ec2nodeclasses.karpenter.k8s.aws`,
`nodepools.karpenter.sh`) do not exist in the cluster until the Karpenter Helm release runs.

A bare `terraform apply` on a fresh cluster therefore fails at the plan phase with:
```
no matches for kind "EC2NodeClass" in group "karpenter.k8s.aws"
```

A second complication is the LBC webhook race: if Karpenter and LBC Helm releases are applied
in the same unordered pass, Karpenter's Service object is intercepted by the LBC mutating
webhook before the LBC pods are running, causing:
```
no endpoints available for service "aws-load-balancer-webhook-service"
```

These are not bugs — they are fundamental sequencing constraints of how Kubernetes CRDs and
admission webhooks work. They require an explicit apply sequence.

---

## Decision

All EKS pattern deployments that include `kubernetes_manifest` CRD-dependent resources must use
a **four-step staged apply sequence**. A bare `terraform apply` must never be documented or
used as the deployment procedure for a fresh cluster.

---

## Options Considered

### Option 1: Bare `terraform apply`

**Description:** Run a single `terraform apply` and rely on Terraform's dependency graph to
order resource creation correctly.

**Pros:**
- Simplest developer experience — one command

**Cons:**
- Fails at plan time on fresh clusters because `kubernetes_manifest` validates CRDs that
  don't yet exist
- No workaround exists within a single apply: the CRDs can't be created before the plan
  that checks them completes
- Misleads engineers into thinking Terraform handles this automatically

---

### Option 2: `depends_on` chains and provider configuration only

**Description:** Use `depends_on = [helm_release.karpenter]` on all `kubernetes_manifest`
resources to force ordering. Rely on Terraform's provider initialization to handle CRD readiness.

**Pros:**
- Single `terraform apply` if it worked
- Dependency chain is explicit in code

**Cons:**
- Terraform still validates `kubernetes_manifest` resources at **plan time** using the current
  Kubernetes API — `depends_on` only affects apply ordering, not plan-time validation
- Plan fails before any resources are created, making `depends_on` irrelevant for this case
- This is a known Terraform limitation with the `kubernetes_manifest` resource type

---

### Option 3: Four-step staged apply ✓ *Selected*

**Description:** Decompose the full apply into four discrete, targeted steps:
1. VPC (no dependencies)
2. EKS cluster + Fargate profiles + core add-ons
3. Helm releases + IAM/SQS + infrastructure (excluding `kubernetes_manifest` CRs)
4. `kubernetes_manifest` resources (CRDs now registered; plan succeeds)

Between steps 3 and 4, explicitly wait for CRDs to be established.

**Pros:**
- Reliable: each step targets only resources that can succeed at that point
- The CRD wait step provides a clear signal that the cluster is ready before proceeding
- Each step is independently re-runnable if it fails (idempotent)
- Maps naturally to the logical deployment phases (network → cluster → controllers → workloads)

**Cons:**
- More complex to document and automate than a single command
- Engineers must follow the sequence exactly; skipping steps causes failures that can be
  hard to diagnose

---

## Consequences

### Positive
- Deployments succeed consistently on fresh clusters without manual intervention
- The staged structure makes it easy to re-run a single failed step without re-applying everything
- CI/CD pipelines can implement each step as a discrete job with clear success criteria

### Trade-offs
- Subsequent `terraform apply` runs (after initial deployment) can use bare `terraform apply`
  safely — the staged sequence is only required on a fresh cluster where CRDs don't yet exist
- The wait command between steps 3 and 4 (`kubectl wait --for=condition=established`) requires
  the kubeconfig to be up to date — must run `aws eks update-kubeconfig` first

---

## Implementation Rules

> The exact command sequence for the `patterns/karpenter` pattern. Adapt resource names for
> other patterns, but preserve the four-step structure.

**DO:**
- Use the following exact sequence for a fresh cluster deployment:

```bash
# Step 1 — VPC
terraform apply -target="module.vpc" -auto-approve

# Step 2 — EKS cluster
terraform apply -target="module.eks" -auto-approve

# Step 3 — Helm releases + IAM/SQS + infrastructure (no kubernetes_manifest)
terraform apply \
  -target=aws_iam_policy.lbc \
  -target=aws_iam_role.lbc \
  -target=aws_iam_role_policy_attachment.lbc \
  -target=helm_release.aws_load_balancer_controller \
  -target=module.karpenter \
  -target=helm_release.karpenter \
  -target=aws_dynamodb_table.app_users \
  -target=aws_ecr_repository.user_app \
  -target=aws_iam_policy.user_app_dynamodb \
  -target=aws_iam_role.user_app \
  -target=aws_iam_role_policy_attachment.user_app_dynamodb \
  -target=aws_cloudwatch_log_group.fargate \
  -target=aws_cloudwatch_log_group.application \
  -target=aws_iam_role_policy.fargate_cloudwatch_kube_system \
  -target=aws_iam_role_policy.fargate_cloudwatch_karpenter \
  -target=kubernetes_namespace_v1.aws_observability \
  -target=kubernetes_config_map_v1.aws_logging \
  -auto-approve

# Sync kubeconfig and wait for Karpenter CRDs
aws eks --region <region> update-kubeconfig --name <cluster-name>
kubectl wait --for=condition=established \
  crd/ec2nodeclasses.karpenter.k8s.aws --timeout=120s

# Step 4 — kubernetes_manifest resources (CRDs now registered)
terraform apply -auto-approve
```

- Document the staged sequence in every pattern's README.md under a "Deploy" section with
  clear explanation of *why* the stages are required
- Add the staged apply to CI/CD workflows as discrete steps (see ADR-014)

**DO NOT:**
- Document `terraform apply` as the deployment procedure for a fresh cluster
- Skip the `kubectl wait` between steps 3 and 4 — plan will fail if CRDs aren't yet registered
- Run steps out of order or combine steps that have sequencing constraints
- Destroy without following the staged destroy sequence (see the pattern README Destroy section)

---

## Verification

```bash
# After Step 2: cluster exists and Fargate nodes are Ready
kubectl get nodes
# Expected: 2+ fargate-ip-* nodes in Ready state

# After Step 3: Helm releases are deployed
helm list -n karpenter    # karpenter release
helm list -n kube-system  # aws-load-balancer-controller release

# CRD readiness check (must pass before Step 4)
kubectl get crd ec2nodeclasses.karpenter.k8s.aws -o jsonpath='{.status.conditions[?(@.type=="Established")].status}'
# Expected: True

# After Step 4: kubernetes_manifest resources exist
kubectl get ec2nodeclass,nodepool
# Expected: both resources exist with READY: True
```

---

## References

- [Terraform kubernetes_manifest known limitations](https://registry.terraform.io/providers/hashicorp/kubernetes/latest/docs/resources/manifest#before-you-use-this-resource)
- [Karpenter getting started](https://karpenter.sh/docs/getting-started/)
- Related: [ADR-002 — Upstream Reference Architecture](./ADR-002-upstream-reference-architecture.md)
- Related: [ADR-012 — IAM Policy Gap Remediation](./ADR-012-iam-policy-gap-remediation.md)
