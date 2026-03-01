# ADR-012: Upstream IAM Policy Gap Remediation Pattern

**Status:** Accepted
**Date:** 2026-03-01
**Scope:** Infrastructure
**Tags:** security, iam, lbc, terraform, third-party, maintainability

---

## Context

Third-party Kubernetes controllers (AWS Load Balancer Controller, ExternalDNS, Cert-Manager
with IRSA, etc.) ship versioned IAM policy documents. These policies are maintained by their
respective upstream teams and are referenced in the controller's installation documentation.

In practice, these policies sometimes have gaps — permissions that the controller needs but
that were omitted from the published policy. The LBC v2.8.1 IAM policy is a documented
example: it is missing `ec2:GetSecurityGroupsForVpc`, which the LBC needs to list VPC
security groups when reconciling `target-type: ip` target groups. Without it, the controller
logs a permission error on every reconciliation loop and retries indefinitely.

Three choices exist when encountering a gap: maintain a private fork of the policy, use a
broad wildcard policy, or apply a targeted inline policy patch alongside the official policy.

---

## Decision

When an upstream IAM policy has a documented gap, apply a **targeted inline policy** that
adds only the missing permissions. Continue to fetch and apply the official upstream policy
as the base. Version-lock the controller Helm chart and the upstream policy URL to the same
version. Document the gap and the expected fix version in code comments.

---

## Options Considered

### Option 1: Fork and maintain a private copy of the IAM policy

**Description:** Copy the upstream IAM policy JSON into the repository. Manually update it
when the upstream policy is updated.

**Pros:**
- Full control over the policy content

**Cons:**
- Creates a maintenance burden — must track upstream policy changes and merge them manually
- Diverges from the upstream-recommended policy surface over time
- Easy to miss upstream security fixes if the upstream policy is updated infrequently

---

### Option 2: Use a broad wildcard policy (`"Resource": "*"` with wide Action list)

**Description:** Write a permissive policy that grants everything the controller is likely
to need, plus more, to avoid permission errors.

**Pros:**
- Never fails due to missing permissions

**Cons:**
- Violates least-privilege principle (see ADR-001)
- Grants more access than the controller needs — blast radius of a controller compromise
  is wider than necessary
- Fails a security review against any standard IAM policy analyzer

---

### Option 3: Official upstream policy + targeted inline patch ✓ *Selected*

**Description:**
1. Fetch the official upstream IAM policy JSON at Terraform plan time from the versioned
   upstream URL (e.g., the LBC policy at the exact tagged release on GitHub)
2. Attach the fetched policy as the primary managed policy
3. Add a minimal inline policy containing only the missing permissions
4. Comment the inline policy with the gap description, the controller version where it
   was observed, and the expected upstream fix version

```hcl
# Fetch the official LBC v2.8.1 policy from upstream
data "http" "lbc_iam_policy" {
  url = "https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.8.1/docs/install/iam_policy.json"
}

resource "aws_iam_policy" "lbc" {
  name   = "${local.name}-lbc"
  policy = data.http.lbc_iam_policy.response_body
}

# Gap: ec2:GetSecurityGroupsForVpc missing from official v2.8.1 policy.
# LBC needs this to list VPC security groups for target-type: ip target groups.
# Check if fixed in v2.9.0+ before upgrading — if included, remove this resource.
resource "aws_iam_role_policy" "lbc_extra" {
  name   = "${local.name}-lbc-extra"
  role   = aws_iam_role.lbc.name
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = ["ec2:GetSecurityGroupsForVpc"]
      Resource = ["*"]
    }]
  })
}
```

**Pros:**
- The primary policy stays in sync with upstream — security fixes are inherited automatically
  when the version is updated
- The patch is minimal and documented — easy to remove when upstream fixes the gap
- Code review sees a clear explanation of why the patch exists
- Fetching at plan time ensures the applied policy always matches the declared version

**Cons:**
- Requires internet access during `terraform plan` to fetch the policy URL (mitigated by
  Terraform provider caching in CI)
- If the upstream URL changes format or the version tag is deleted, the plan fails

---

## Consequences

### Positive
- Security posture remains as close to the upstream recommendation as possible
- The gap and its expected fix are documented in code — future maintainers know to check
  when upgrading the controller version
- Upgrading the controller version also upgrades its base IAM policy automatically

### Trade-offs
- Terraform plan requires network access to the upstream policy URL
- The inline policy must be manually reviewed and potentially removed on controller upgrades

---

## Implementation Rules

> These rules apply to all Claude agents and engineers managing third-party controller IAM policies.

**DO:**
- Fetch the upstream IAM policy using the `http` provider with a version-locked URL:
  ```hcl
  data "http" "controller_iam_policy" {
    url = "https://raw.githubusercontent.com/<org>/<repo>/v<X.Y.Z>/docs/install/iam_policy.json"
  }
  ```
- Pin the Helm chart version and the IAM policy URL to the **same** upstream version
- Add a comment on every inline patch resource with:
  - The missing permission and why it is needed
  - The controller version where the gap was observed
  - The minimum upstream version expected to fix it
  - The action required when upgrading past that version
- Check upstream release notes when upgrading a controller — remove the inline patch if
  the official policy now includes the missing permission

**DO NOT:**
- Use `"Resource": "*"` in the base policy — scope resources to specific ARNs where possible
- Add permissions to the inline patch speculatively — only add permissions that have been
  confirmed as missing from a specific error log
- Forget to update the IAM policy URL when upgrading the controller Helm chart version —
  version mismatch between the policy and the controller can cause subtle permission errors

**Upgrade checklist:**
```bash
# When upgrading a controller (e.g., LBC v2.8.1 → v2.9.0):
# 1. Update the Helm chart version in lbc.tf
# 2. Update the IAM policy URL to match the new version
# 3. Diff the new policy against the old one:
curl -s https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v<NEW>/docs/install/iam_policy.json \
  | diff - <(terraform show -json | jq -r '.values.root_module.resources[] | select(.name=="lbc") | .values.policy')
# 4. Check if any inline patch permissions are now in the official policy — remove if so
# 5. Plan and apply
```

---

## Verification

```bash
# Confirm the fetched policy version matches the installed Helm chart version
HELM_VERSION=$(helm list -n kube-system -o json | jq -r '.[] | select(.name=="aws-load-balancer-controller") | .app_version')
# Expected: v2.8.1 (or the current pinned version)

# Confirm the IAM policy URL contains the same version string
grep 'url' lbc.tf | grep "v${HELM_VERSION}"
# Expected: URL contains matching version

# Confirm the inline patch exists and is documented
grep -A 10 'lbc_extra' lbc.tf
# Expected: comment explaining the gap and expected fix version

# Confirm LBC is not logging permission errors
kubectl logs -n kube-system \
  -l app.kubernetes.io/name=aws-load-balancer-controller \
  --tail=50 | grep -i "AccessDenied\|not authorized"
# Expected: no permission errors
```

---

## References

- [AWS LBC IAM policy (v2.8.1)](https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.8.1/docs/install/iam_policy.json)
- [AWS IAM least-privilege best practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)
- Related: [ADR-001 — IRSA](./ADR-001-irsa-over-node-instance-profiles.md)
- Related: [ADR-002 — Upstream Reference Architecture](./ADR-002-upstream-reference-architecture.md)
