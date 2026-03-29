# ADR-017: Infrastructure as Code Governance

**Status:** Accepted
**Date:** 2026-03-29
**Scope:** Infrastructure
**Tags:** terraform, state, iac, governance, ci-cd, multi-account, security

---

## Context

The platform spans multiple AWS accounts (sandbox, staging, production) organised under AWS
Organizations. Each account contains independent infrastructure managed via Terraform. Without
explicit governance rules, several failure modes emerge: shared state files cause concurrent
apply conflicts; cross-account references are implemented by copying output values manually
rather than by reading remote state; engineers run `terraform apply` from local machines
against production, bypassing review and CI validation; and plan output is not attached to
PRs, so reviewers approve changes without knowing what will actually be applied.

---

## Decision

Each AWS account has exactly one Terraform state file, stored in S3 with DynamoDB locking.
Cross-account dependencies use remote state data sources, not manually copied values.
Production applies run exclusively via the CI/CD pipeline — never from a local machine.
Every PR that includes infrastructure changes must attach the `terraform plan` output before
review begins. The staged apply pattern from ADR-005 applies within each account.

---

## Options Considered

### Option 1: Single shared state file for all accounts

**Description:** One S3 bucket and one DynamoDB lock table, one state file containing all
accounts' infrastructure.

**Pros:**
- Single location to inspect all infrastructure state

**Cons:**
- A failed apply in one account can corrupt state for all accounts
- Concurrent applies across accounts require serialisation on a single lock
- Blast radius of a misconfigured plan extends to all accounts simultaneously
- No natural boundary for access control — every engineer with state access can see all accounts

---

### Option 2: One state file per component within an account

**Description:** Split state per Terraform module or per logical component (vpc, eks,
karpenter, etc.) within a single account.

**Pros:**
- Smaller blast radius per apply
- Faster plan and apply times per component

**Cons:**
- Multiplies the number of state backends to manage
- Cross-component references within an account require remote state data sources even for
  simple intra-account dependencies (e.g., VPC ID needed by EKS module)
- ADR-005's staged apply already handles intra-account sequencing without splitting state

---

### Option 3: One state file per AWS account ✓ *Selected*

**Description:** One S3 backend with DynamoDB locking per AWS account. The S3 bucket and
DynamoDB table live in the same account as the infrastructure they track. Cross-account
references read state via `terraform_remote_state` data sources with read-only IAM permissions.

**Pros:**
- Account isolation: a state corruption in one account does not affect others
- Access control follows account boundaries — production state is accessible only to the
  pipeline IAM role, not to individual engineers
- Clear ownership: each account team owns their state backend
- Cross-account dependencies are explicit in code and version-controlled

**Cons:**
- Multiple backends to provision (one per account); mitigated by using Terraform to provision
  the backends themselves (bootstrap pattern)
- `terraform_remote_state` data sources require the remote state to be applied before the
  dependent state can plan — ordering across accounts must be documented

---

## Consequences

### Positive
- State corruption is bounded to a single account
- Production state is never accessible from engineer workstations — only the pipeline role
  can read and write production state
- Cross-account dependencies are visible in code review
- The staged apply sequence from ADR-005 continues to apply within each account unchanged

### Trade-offs
- Cross-account dependencies require the upstream account's state to be available at plan time
- The state bootstrap (S3 bucket + DynamoDB table) must be applied manually once per account
  before any other Terraform can run — document this in each account's `README.md`

---

## Implementation Rules

> These rules apply to all Claude agents and engineers managing Terraform across accounts.

**State backend configuration (one per account):**

```hcl
terraform {
  backend "s3" {
    bucket         = "<account-name>-terraform-state"
    key            = "platform/terraform.tfstate"
    region         = "us-east-2"
    dynamodb_table = "<account-name>-terraform-locks"
    encrypt        = true
  }
}
```

**Cross-account remote state reference:**

```hcl
data "terraform_remote_state" "shared" {
  backend = "s3"
  config = {
    bucket = "shared-terraform-state"
    key    = "platform/terraform.tfstate"
    region = "us-east-2"
  }
}

# Reference an output from the remote state
locals {
  vpc_id = data.terraform_remote_state.shared.outputs.vpc_id
}
```

**Apply authorisation by environment:**

| Environment | Who may run `terraform apply` | How |
|-------------|------------------------------|-----|
| sandbox | Any engineer | Local machine or pipeline |
| staging | Pipeline only | `plan-examples.yml` workflow (manual dispatch) |
| production | Pipeline only | `e2e-parallel-full.yml` or dedicated deploy workflow (manual dispatch, requires approval) |

**DO:**
- Store Terraform state in S3 with server-side encryption (`encrypt = true`) and DynamoDB
  locking in the same account as the infrastructure
- Use `terraform_remote_state` data sources for cross-account dependencies — never copy
  output values manually
- Attach `terraform plan` output to every PR that changes infrastructure before requesting
  review — plan output must be for the same commit as the PR head
- Follow the staged apply sequence from ADR-005 for any account that contains EKS with
  `kubernetes_manifest` resources
- Run `terraform plan` in CI on every PR targeting `develop` or `main` (see ADR-014)

**DO NOT:**
- Run `terraform apply` against staging or production from a local machine
- Share state backend credentials between accounts — each account has its own IAM role
  for state access
- Commit `.terraform/` directories or `*.tfstate` files to the repository
- Use `terraform apply -auto-approve` in production — always require manual approval in CI
- Use a single workspace to manage multiple accounts — separate backends are required

**Plan attachment checklist (PR author):**

```bash
# Generate plan and save output for PR attachment
terraform plan -out=tfplan 2>&1 | tee plan-output.txt
# Attach plan-output.txt to the PR description before requesting review
```

---

## Verification

```bash
# Confirm state backend is S3 with DynamoDB locking
terraform init -backend-config=backend.hcl
terraform show -json | jq '.values.backend'
# Expected: S3 backend with DynamoDB table configured

# Confirm state is not stored locally
ls -la *.tfstate 2>/dev/null || echo "OK: no local state files"

# Confirm DynamoDB lock table exists
aws dynamodb describe-table \
  --table-name <account-name>-terraform-locks \
  --region us-east-2 \
  --query 'Table.{Name:TableName,Status:TableStatus}'
# Expected: TableStatus ACTIVE

# Confirm production pipeline role — local credentials must not have apply permissions
aws sts get-caller-identity --query 'Arn'
# In production: expected to be the pipeline role ARN, not an engineer's role

# Confirm plan is attached to the PR (manual check)
gh pr view <PR-number> --json body | jq -r '.body' | grep -i "terraform plan"
# Expected: plan output or link to plan artifact present in PR body
```

---

## References

- [Terraform S3 backend documentation](https://developer.hashicorp.com/terraform/language/settings/backends/s3)
- [Terraform remote state data source](https://developer.hashicorp.com/terraform/language/state/remote-state-data)
- [AWS Organizations multi-account best practices](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_best-practices.html)
- Related: [ADR-005 — Staged Terraform Apply](./ADR-005-staged-terraform-apply.md)
- Related: [ADR-014 — GitHub Actions Pipeline Automation](../process/ADR-014-github-actions-pipeline.md)
- Related: [ADR-001 — IRSA](../security/ADR-001-irsa-over-node-instance-profiles.md)
