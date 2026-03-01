# ADR-011: Dual Observability Paths for Fargate and EC2 Node Types

**Status:** Accepted
**Date:** 2026-03-01
**Scope:** Infrastructure
**Tags:** observability, logging, cloudwatch, fargate, ec2, karpenter, fluent-bit

---

## Context

This platform runs two distinct compute types within the same EKS cluster:

- **Fargate nodes** — system workloads (Karpenter controller, LBC, core add-ons in
  `kube-system` and `karpenter` namespaces)
- **EC2 nodes (Karpenter-provisioned)** — application workloads in `default` and other
  app namespaces

DaemonSets — the standard mechanism for deploying per-node agents in Kubernetes — do not run
on Fargate. Fargate's managed compute model does not expose the node OS or allow scheduling
system-level agents. A single log collection approach based on a DaemonSet therefore cannot
cover both compute types.

Without addressing this gap, system pod logs (Karpenter controller, LBC) and application pod
logs end up in different states: either system logs are missing, or application logs require
a second collection mechanism.

---

## Decision

The cluster uses **two separate log collection paths**, one per compute type, converging on
CloudWatch Logs with distinct log groups. Neither path requires any application code changes.
Both log groups use 30-day retention and are pre-created by Terraform.

| Compute | Collection Mechanism | Log Group |
|---------|---------------------|-----------|
| Fargate | Built-in Fluent Bit sidecar via `aws-observability` namespace | `/aws/eks/<cluster>/fargate` |
| EC2 (Karpenter) | `amazon-cloudwatch-observability` EKS addon (DaemonSet) | `/aws/eks/<cluster>/application` |

---

## Options Considered

### Option 1: DaemonSet-only (e.g., Fluent Bit DaemonSet)

**Description:** Deploy a Fluent Bit or Fluentd DaemonSet on EC2 nodes for log collection.
Accept that Fargate pods have no log collection.

**Pros:**
- Single agent, single configuration
- Standard pattern in non-Fargate clusters

**Cons:**
- DaemonSets do not run on Fargate — Karpenter controller and LBC logs are lost
- System pod logs are critical for debugging node provisioning failures and ingress issues

---

### Option 2: Sidecar injection for all pods (manual or via admission webhook)

**Description:** Inject a Fluent Bit sidecar container into every pod via a mutating
admission webhook or manual template modification.

**Pros:**
- Uniform approach across Fargate and EC2

**Cons:**
- Doubles the container count in every pod — significantly increases resource consumption
- Requires maintaining a custom admission webhook or modifying every pod spec manually
- Sidecars add startup ordering complexity and restart coupling

---

### Option 3: External log shipping (e.g., Datadog, Splunk, CloudWatch agent on host)

**Description:** Use a third-party log aggregation platform with native EKS support.

**Pros:**
- Richer querying and alerting features than CloudWatch Logs Insights
- Unified view across compute types

**Cons:**
- Additional cost (per-GB ingestion fees on top of CloudWatch)
- External dependency for a core operational function
- More complex setup and credentials management

---

### Option 4: Dual-path — Fargate sidecar + EC2 DaemonSet ✓ *Selected*

**Description:**
- **Fargate:** EKS automatically injects a Fluent Bit sidecar into every Fargate pod when
  the `aws-observability` namespace exists with `aws-observability: enabled` label and an
  `aws-logging` ConfigMap with the Fluent Bit `[OUTPUT]` configuration. The sidecar assumes
  the Fargate pod execution role — no separate credentials required.
- **EC2:** The `amazon-cloudwatch-observability` EKS addon deploys CloudWatch agent and
  Fluent Bit as DaemonSets on EC2 nodes. Requires `CloudWatchAgentServerPolicy` on the
  Karpenter node IAM role.

Both mechanisms route logs to CloudWatch Logs. Both log groups support CloudWatch Logs
Insights queries with the same query syntax.

**Pros:**
- Complete coverage: all pods on both compute types are logged
- No application code changes required — collection is transparent to the workload
- Both mechanisms are AWS-managed with automatic updates
- Converges on CloudWatch — single query interface for all logs

**Cons:**
- Two log groups to query when debugging cross-plane issues (system vs. application logs)
- Fargate Fluent Bit configuration (`aws-logging` ConfigMap) is separate from EC2
  CloudWatch agent configuration — two configs to maintain

---

## Consequences

### Positive
- No log visibility gaps: Karpenter controller logs (provisioning decisions), LBC logs
  (ingress events), and application logs are all captured
- CloudWatch Logs Insights can be used across both log groups with identical syntax
- Both log groups are pre-created with 30-day retention — no race condition on first pod start
  (`auto_create_group false` in Fluent Bit config)

### Trade-offs
- When debugging an issue, engineers must determine which log group to query:
  `/fargate` for system pods; `/application` for app pods
- IAM permissions required on two separate roles: Fargate execution roles (inline policy for
  `CreateLogStream`, `PutLogEvents`) and Karpenter node role (`CloudWatchAgentServerPolicy`)

---

## Implementation Rules

> These rules apply to all Claude agents and engineers configuring observability for this platform.

**Fargate path — required Terraform resources:**

```hcl
# Namespace with the required label
resource "kubernetes_namespace_v1" "aws_observability" {
  metadata {
    name   = "aws-observability"
    labels = { "aws-observability" = "enabled" }
  }
}

# ConfigMap activates the Fluent Bit sidecar output
resource "kubernetes_config_map_v1" "aws_logging" {
  metadata {
    name      = "aws-logging"
    namespace = "aws-observability"
  }
  data = {
    "output.conf" = <<-EOT
      [OUTPUT]
          Name              cloudwatch_logs
          Match             *
          region            ${var.region}
          log_group_name    /aws/eks/${var.cluster_name}/fargate
          log_stream_prefix fargate-
          auto_create_group false
    EOT
  }
}

# Scoped CloudWatch Logs permissions on each Fargate execution role
resource "aws_iam_role_policy" "fargate_cloudwatch" {
  # Actions: CreateLogStream, PutLogEvents, DescribeLogStreams, CreateLogGroup
  # Resource: arn:aws:logs:*:*:log-group:/aws/eks/<cluster>/*
}
```

**EC2 path — required configuration:**

```hcl
# EKS addon (in cluster_addons)
"amazon-cloudwatch-observability" = {
  most_recent = true
}

# Node role permission (in karpenter.tf)
node_iam_role_additional_policies = {
  CloudWatchAgentServerPolicy = "arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy"
}
```

**DO:**
- Pre-create both CloudWatch log groups in Terraform with `retention_in_days = 30`
- Set `auto_create_group false` in the Fargate Fluent Bit ConfigMap to prevent race conditions
- Attach `CloudWatchAgentServerPolicy` to the Karpenter node IAM role — without it, the
  CloudWatch agent DaemonSet cannot write logs
- Query both log groups when debugging issues that span system and application pods

**DO NOT:**
- Use `auto_create_group true` in the Fluent Bit config — Terraform-managed log groups
  have retention policies; auto-created groups do not
- Assume a single log group contains all logs — always check which compute type your pod
  is running on to determine the correct log group

---

## Verification

```bash
# Fargate: confirm aws-observability namespace and ConfigMap exist
kubectl get ns aws-observability -o jsonpath='{.metadata.labels}'
# Expected: {aws-observability: enabled}

kubectl get configmap aws-logging -n aws-observability
# Expected: exists

# EC2: confirm CloudWatch Observability addon is ACTIVE
aws eks describe-addon \
  --cluster-name <cluster> \
  --addon-name amazon-cloudwatch-observability \
  --region <region> \
  --query 'addon.status'
# Expected: "ACTIVE"

# Both: confirm log groups exist with correct retention
aws logs describe-log-groups \
  --region <region> \
  --log-group-name-prefix "/aws/eks/<cluster>" \
  --query 'logGroups[*].{name:logGroupName,retention:retentionInDays}'
# Expected: two groups, both with retentionInDays: 30

# Query system logs (Karpenter controller)
aws logs filter-log-events \
  --region <region> \
  --log-group-name "/aws/eks/<cluster>/fargate" \
  --filter-pattern "karpenter" \
  --limit 10

# Query application logs
aws logs filter-log-events \
  --region <region> \
  --log-group-name "/aws/eks/<cluster>/application" \
  --limit 10
```

---

## References

- [AWS Fargate logging with Fluent Bit](https://docs.aws.amazon.com/eks/latest/userguide/fargate-logging.html)
- [Amazon CloudWatch Observability EKS Add-on](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Container-Insights-setup-EKS-addon.html)
- Related: [ADR-003 — Structured JSON Logging](./ADR-003-structured-json-logging.md)
- Related: [ADR-001 — IRSA](./ADR-001-irsa-over-node-instance-profiles.md)
