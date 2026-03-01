# ADR-001: IRSA over Node Instance Profiles for AWS Credential Delivery

**Status:** Accepted
**Date:** 2026-03-01
**Scope:** Infrastructure + Application
**Tags:** security, iam, eks, credentials

---

## Context

Pods running on EKS often need AWS credentials to call services such as DynamoDB, S3, SQS, and
Secrets Manager. Three delivery mechanisms exist: attaching IAM policies to the EC2 node's
instance profile (shared across all pods on the node), injecting credentials as environment
variables or Secrets Manager values, or using IAM Roles for Service Accounts (IRSA).

The choice affects blast radius on credential compromise, operational burden of secret rotation,
and auditability via CloudTrail.

---

## Decision

All application and controller workloads running on EKS **must** use IRSA to obtain AWS
credentials. Node instance profiles are reserved for node-level AWS API calls only (e.g.,
EC2 auto-discovery by Karpenter). No application-level AWS credentials may be hardcoded,
stored in environment variables, or retrieved from Secrets Manager at container startup.

---

## Options Considered

### Option 1: Node Instance Profile (EC2 IAM Role)

**Description:** Attach IAM policies directly to the EC2 instance role. All pods on the node
share the same credentials implicitly via the EC2 metadata service (IMDS).

**Pros:**
- Zero application code changes required
- No per-workload IAM configuration

**Cons:**
- All pods on a node share the same permissions — violates least privilege
- A compromised pod gains all permissions of every other workload on that node
- No per-pod CloudTrail attribution — cannot determine which pod made an API call
- Cannot scope to individual DynamoDB tables, S3 buckets, or SQS queues
- Does not work with Fargate (no EC2 instance profile)

---

### Option 2: Credentials as Environment Variables or Secrets Manager

**Description:** Generate long-lived IAM access keys; store in Secrets Manager or as Kubernetes
Secrets; inject into pod via environment variables at startup.

**Pros:**
- Works on any platform (not EKS-specific)
- Familiar pattern for teams without EKS experience

**Cons:**
- Long-lived credentials require rotation; rotation failure = outage or stale access
- Credentials appear in pod environment — visible via `kubectl describe pod` or process inspection
- Kubernetes Secrets are base64-encoded, not encrypted at rest by default
- Rotation requires redeployment or external secret sync (ESO, ASM agent)
- Adds operational overhead: secret creation, versioning, access control, audit

---

### Option 3: IAM Roles for Service Accounts (IRSA) ✓ *Selected*

**Description:** EKS projects a short-lived OIDC token into the pod's filesystem. The AWS SDK
exchanges this token for temporary STS credentials scoped to a specific IAM role via
`AssumeRoleWithWebIdentity`. Each Kubernetes ServiceAccount maps to exactly one IAM role.

**Pros:**
- Zero credentials in code, environment, or Secrets Manager
- AWS SDK resolves credentials automatically via the default provider chain
- Scoped per-workload: each ServiceAccount maps to a distinct IAM role
- Tokens are short-lived (1 hour) and automatically rotated by EKS
- Full CloudTrail attribution: API calls include the ServiceAccount identity
- Works on both EC2 (Karpenter nodes) and Fargate

**Cons:**
- Requires EKS OIDC provider (enabled by default in EKS >= 1.13)
- IAM role trust policy must be exact — typos silently reject all tokens
- Per-workload IAM role increases IAM resource count

---

## Consequences

### Positive
- No credential rotation burden — tokens expire and refresh automatically
- Blast radius is bounded: a compromised pod can only access its own IAM role's permissions
- CloudTrail shows per-ServiceAccount API calls, enabling precise incident attribution
- Least privilege is enforceable at the individual table/bucket/queue level

### Trade-offs
- IRSA trust policy errors fail silently: `sts:AssumeRoleWithWebIdentity` is rejected and the
  pod logs only a generic auth error, not a trust policy mismatch message
- The OIDC provider endpoint must be accessible; air-gapped environments need additional config

---

## Implementation Rules

> These rules apply to all Claude agents and engineers building on this platform.

**DO:**
- Create one IAM role per Kubernetes workload (one role per Deployment/StatefulSet)
- Scope the trust policy `sub` condition to the exact ServiceAccount:
  ```json
  "StringEquals": {
    "oidc.eks.<region>.amazonaws.com/id/<OIDC_ID>:sub": "system:serviceaccount:<namespace>:<sa-name>",
    "oidc.eks.<region>.amazonaws.com/id/<OIDC_ID>:aud": "sts.amazonaws.com"
  }
  ```
- Annotate the Kubernetes ServiceAccount with the role ARN:
  ```yaml
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::<account>:role/<role-name>
  ```
- Scope IAM policies to specific resource ARNs (table ARN, bucket ARN), not `"Resource": "*"`
- Use the AWS SDK default credential provider chain — do not configure credentials explicitly

**DO NOT:**
- Use `system:serviceaccounts:` (plural) in trust policy — AWS `StringEquals` is exact-match;
  the plural form silently rejects every `AssumeRoleWithWebIdentity` call
- Attach application-level permissions (DynamoDB, S3) to the Karpenter node IAM role
- Store `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` in Kubernetes Secrets, ConfigMaps,
  Helm values, or Dockerfiles
- Use a wildcard `"Resource": "*"` for data-plane services

---

## Verification

```bash
# Confirm the ServiceAccount has the IRSA annotation
kubectl get sa <sa-name> -n <namespace> -o jsonpath='{.metadata.annotations}'
# Expected: {"eks.amazonaws.com/role-arn":"arn:aws:iam::..."}

# Decode the projected token sub claim from a running pod
POD=$(kubectl get pods -l app.kubernetes.io/name=<app> -o jsonpath='{.items[0].metadata.name}')
kubectl exec $POD -- sh -c 'cat $AWS_WEB_IDENTITY_TOKEN_FILE' \
  | cut -d. -f2 | base64 -d 2>/dev/null | python3 -m json.tool | grep -E 'sub|aud'
# Expected sub: "system:serviceaccount:<namespace>:<sa-name>"

# Verify the IAM trust policy condition is singular
aws iam get-role --role-name <role-name> \
  --query 'Role.AssumeRolePolicyDocument.Statement[0].Condition' --output json
# Confirm "StringEquals" key contains "system:serviceaccount:" (singular)

# Confirm the app can call AWS services (no auth errors)
kubectl logs -l app.kubernetes.io/name=<app> --tail=20 | grep -i "error\|unauthorized\|denied"
# Expected: no auth errors
```

---

## References

- [AWS IRSA Documentation](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html)
- [EKS Best Practices: Security](https://aws.github.io/aws-eks-best-practices/security/docs/iam/)
- Related: [ADR-002 — Upstream Reference Architecture as Foundation](./ADR-002-upstream-reference-architecture.md)
