# ADR-014: GitHub Actions Pipeline Automation

**Status:** Accepted
**Date:** 2026-03-01
**Scope:** Infrastructure + Application
**Tags:** ci-cd, automation, github-actions, security, testing, terraform, quality-gates

---

## Context

Manual validation of Terraform, Helm charts, and application code is error-prone and does not
scale across multiple contributors or Claude agents making frequent changes. A consistent,
automated quality gate pipeline ensures every change is validated identically regardless of
who authored it — human engineer, pair programmer, or AI agent.

The pipeline must cover three distinct validation tiers:
1. **Syntax and style** — immediate feedback on formatting, linting, and documentation
2. **Plan-level validation** — Terraform can successfully plan against real AWS without apply
3. **End-to-end testing** — full deploy and destroy cycle against real infrastructure

Security of the pipeline itself is also a consideration: workflows that access AWS or merge
into protected branches are attack surfaces if not hardened.

---

## Decision

GitHub Actions is the CI/CD platform for this repository. Workflows are organized into three
tiers corresponding to their cost and risk. All workflows use OIDC for AWS authentication
(no stored long-lived AWS credentials), pin all third-party Action versions to commit SHAs,
and apply `step-security/harden-runner` on every job.

---

## Options Considered

### Option 1: Jenkins (self-hosted)

**Description:** Operate a self-hosted Jenkins instance for pipeline execution.

**Pros:**
- Full control over the execution environment
- No per-minute cost for compute

**Cons:**
- Significant operational overhead: EC2 instance management, plugin updates, security patching
- No native GitHub integration — requires webhook configuration and credential management
- Not practical for a team-sized or solo operation

---

### Option 2: CircleCI or GitLab CI

**Description:** Use a third-party CI platform with native GitHub integration.

**Pros:**
- Managed compute — no self-hosting overhead
- Rich feature sets

**Cons:**
- Additional vendor relationship and billing
- GitHub Actions is already integrated into the GitHub repository — adding a second platform
  adds complexity and split visibility (PR checks appear in two places)

---

### Option 3: GitHub Actions (native) ✓ *Selected*

**Description:** Use GitHub Actions workflows stored in `.github/workflows/`. Three workflow
tiers map to the three validation levels.

**Tier 1 — PR Gate (runs on every PR to `main` or `develop`):**
- `pre-commit.yml` — `terraform_fmt`, `terraform_docs`, `tflint`, `terraform_validate`,
  `cspell` spelling check
- `pr-title.yml` — conventional commit title format
- `markdown-link-check.yml` — validate all Markdown links
- `dependency-review.yml` — flag new dependencies with known vulnerabilities

**Tier 2 — Plan Validation (manual dispatch):**
- `plan-examples.yml` — `terraform plan` across all `patterns/` directories
  using a matrix strategy; authenticates to AWS via OIDC

**Tier 3 — E2E Testing (manual dispatch):**
- `e2e-parallel-full.yml` — full deploy and destroy cycle for selected patterns;
  runs `iamlive` alongside apply to capture the actual IAM permissions used and
  upload them to S3 for policy auditing

**Pros:**
- Zero additional vendor: GitHub Actions is included with the repository
- PR checks surface directly in the pull request interface
- OIDC AWS authentication eliminates stored credentials
- `iamlive` in E2E tests captures actual AWS API calls — identifies IAM gaps proactively

**Cons:**
- GitHub Actions compute is billed per minute — E2E tests (full EKS deploy) are expensive
  and therefore manual-dispatch only
- Workflow YAML syntax can be complex; changes require testing in a branch

---

## Consequences

### Positive
- Every PR receives automated feedback on formatting, documentation, and linting within
  minutes of opening
- Terraform plan validation catches resource errors before they reach apply
- E2E pipeline produces a verified IAM policy for each pattern (via `iamlive`) that can
  be audited and compared against the declared policies
- No long-lived AWS credentials in GitHub Secrets — OIDC tokens are ephemeral

### Trade-offs
- E2E tests cost real AWS money (EKS cluster, Fargate nodes, NAT Gateway per run) —
  must be used deliberately, not on every PR
- `plan-examples.yml` requires AWS credentials (OIDC) — skipped on forks that don't have
  access to repository secrets

---

## Implementation Rules

> These rules apply to all Claude agents and engineers authoring GitHub Actions workflows.

**Security hardening (required on every workflow job):**

```yaml
steps:
  - name: Harden Runner
    uses: step-security/harden-runner@v2
    with:
      egress-policy: audit   # Use 'block' for higher security; 'audit' to discover needed egress

  - name: Checkout
    uses: actions/checkout@8e8c483db84b4bee98b60c0593521ed34d9990e8  # pin to commit SHA, not tag
```

**AWS authentication (OIDC — no stored credentials):**

```yaml
permissions:
  id-token: write   # required for OIDC
  contents: read

steps:
  - name: Auth AWS
    uses: aws-actions/configure-aws-credentials@v5.1.1
    with:
      role-to-assume: ${{ secrets.ROLE_TO_ASSUME }}
      aws-region: us-east-2
      role-duration-seconds: 3600
      role-session-name: GithubActions-Session
```

**Concurrency control (cancel redundant runs):**

```yaml
concurrency:
  group: '${{ github.workflow }} @ ${{ github.event.pull_request.head.label || github.head_ref || github.ref }}'
  cancel-in-progress: true
```

**Path filtering (only run on relevant changes):**

```yaml
- uses: dorny/paths-filter@v3
  id: changes
  with:
    filters: |
      src:
        - '**.tf'
        - '**.yml'
```

**DO:**
- Pin all third-party Actions to a commit SHA (`uses: actions/checkout@<sha>`), not a
  floating tag — tags are mutable and can be overwritten
- Use OIDC for AWS authentication — never store `AWS_ACCESS_KEY_ID` in repository Secrets
- Use `step-security/harden-runner` on every job — it enforces egress restrictions and
  detects supply chain attacks
- Use `fail-fast: false` in matrix strategies — allows other patterns to complete even if
  one fails, providing more complete test results
- Set `concurrency: cancel-in-progress: true` on PR-triggered workflows — avoids wasting
  compute on superseded commits
- Restrict expensive workflows (E2E) to `workflow_dispatch` — prevents accidental runs
- Add `if: github.repository == '<org>/<repo>'` to plan/E2E jobs — skips on forks without
  access to secrets, preventing confusing failures

**DO NOT:**
- Store long-lived AWS credentials (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`) in
  GitHub Secrets — use OIDC exclusively
- Use floating version tags for third-party Actions (e.g., `@v3`) — pin to a commit SHA
- Run full E2E tests on every PR — the per-run AWS cost is significant
- Skip `harden-runner` — it is the primary defense against workflow injection attacks

**Tier assignment for new workflows:**

| Trigger | Cost | Use for |
|---------|------|---------|
| PR to `main`/`develop` | Low | Linting, formatting, static analysis, unit tests |
| Manual dispatch | Medium | Terraform plan, integration tests |
| Manual dispatch (controlled) | High | Full E2E deploy/destroy |

---

## Verification

```bash
# List all workflows and their triggers
gh workflow list

# Confirm PR gate workflows are enabled and run on PRs
gh run list --workflow=pre-commit.yml --limit=5

# Confirm OIDC is used (no AccessKeyId in workflow env)
grep -r 'AWS_ACCESS_KEY_ID' .github/workflows/
# Expected: no matches (OIDC only)

# Confirm all Action versions are pinned to SHAs (not floating tags)
grep -r 'uses:' .github/workflows/ | grep -v '@[0-9a-f]\{40\}\|@v[0-9]'
# Expected: every 'uses:' line pins to a SHA or a stable version tag

# View recent E2E run results
gh run list --workflow=e2e-parallel-full.yml --limit=5
```

---

## References

- [GitHub Actions security hardening](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions)
- [step-security/harden-runner](https://github.com/step-security/harden-runner)
- [AWS OIDC for GitHub Actions](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_create_oidc.html)
- [iamlive — live IAM policy generation](https://github.com/iann0036/iamlive)
- [OpenSSF Scorecard](https://securityscorecards.dev/) (enforced by `scorecards.yml`)
- Related: [ADR-013 — GitFlow Branching Strategy](./ADR-013-gitflow-branching-strategy.md)
- Related: [ADR-005 — Staged Terraform Apply](./ADR-005-staged-terraform-apply.md)
