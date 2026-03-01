# ADR-002: Upstream Reference Architecture as the Infrastructure Foundation

**Status:** Accepted
**Date:** 2026-03-01
**Scope:** Infrastructure
**Tags:** modularity, iac, terraform, eks, community, maintainability

---

## Context

Building production EKS infrastructure involves dozens of interdependent decisions: Fargate
vs. managed node groups, Karpenter vs. cluster-autoscaler, IRSA setup, subnet tagging conventions,
Helm ordering constraints, webhook race conditions, and more. Each of these can be discovered
independently — or inherited from a maintained reference architecture.

AWS publishes `aws-ia/terraform-aws-eks-blueprints` as an open-source library of battle-tested
EKS patterns. The question is whether to consume it as a Terraform module, fork-and-modify it,
or build independently using the same community modules it wraps.

---

## Decision

Use the `aws-ia/terraform-aws-eks-blueprints` patterns as the **starting point** for new EKS
infrastructure. Fork the relevant pattern directory into the project, then modify it to meet
project-specific requirements. Do not consume patterns as remote Terraform modules when
customization is expected. Document every deviation from upstream in code comments or ADRs.

---

## Options Considered

### Option 1: Build from scratch using raw community modules

**Description:** Write all Terraform directly using `terraform-aws-modules/eks/aws`,
`terraform-aws-modules/vpc/aws`, etc., without referencing any blueprint pattern.

**Pros:**
- Maximum control over every resource
- No upstream coupling

**Cons:**
- Re-discovers known failure modes (webhook race conditions, staged apply requirements,
  missing IAM permissions in official policies)
- Higher initial cost; lower confidence in production readiness
- No community-reviewed baseline to diff against when debugging

---

### Option 2: Consume blueprint patterns as remote Terraform modules

**Description:** Reference `aws-ia/terraform-aws-eks-blueprints` patterns as Terraform module
sources with pinned versions. Accept all upstream defaults.

**Pros:**
- Minimal code surface
- Automatically benefits from upstream patches

**Cons:**
- Severely limits customization — adding resources requires overrides or wrappers
- Upstream changes can break the deployment without warning (even with version pins, module
  APIs change between major versions)
- Pattern-level decisions (logging config, subnet tags, Karpenter settings) are not exposed
  as module inputs — they are hardcoded in the upstream source

---

### Option 3: Fork and modify upstream patterns ✓ *Selected*

**Description:** Copy the relevant upstream pattern directory into the project repository.
Treat it as owned code from that point forward. Make targeted modifications for project
requirements. Periodically check upstream for relevant fixes when component versions are upgraded.

**Pros:**
- Inherits proven configuration for known failure modes (see Implementation Notes)
- Full control over every resource and configuration value
- Easy to diff against upstream when debugging ("did we introduce this, or was it always here?")
- Patterns are self-contained and deployable without external module dependencies beyond
  community modules (`terraform-aws-modules/eks`, `terraform-aws-modules/vpc`, etc.)
- Each pattern directory is independently versioned via git — no module release cycle dependency

**Cons:**
- Does not automatically receive upstream bug fixes — must monitor upstream changelog
- Risk of drift: project-specific changes make future upstream merges harder
- Requires discipline to document deviations so future maintainers understand what changed and why

---

## Consequences

### Positive
- New patterns start from a production-validated baseline, not a blank slate
- Known failure modes are handled from day one (webhook ordering, staged apply, IRSA trust
  policy format, arm64 constraints, LBC IAM policy gaps)
- The project's patterns repository becomes an internal library of reviewed, working infrastructure
- Engineers and Claude agents can reference the `patterns/` directory as canonical examples

### Trade-offs
- Upstream maintenance requires monitoring: when a component version is upgraded (e.g., LBC
  `v2.8.1` → `v2.9.0`), the maintainer must check whether the upstream IAM policy has changed
- The fork-point should be documented (git tag or commit SHA) so future maintainers know
  the baseline and can assess drift

---

## Implementation Rules

> These rules apply to all Claude agents and engineers creating or modifying infrastructure patterns.

**DO:**
- Start every new EKS pattern by copying the closest upstream blueprint pattern from
  `aws-ia/terraform-aws-eks-blueprints` as the baseline
- Document the upstream source and fork-point in the pattern's `README.md` or `main.tf` header:
  ```hcl
  # Forked from: aws-ia/terraform-aws-eks-blueprints patterns/karpenter
  # Upstream ref: <git-tag-or-commit>
  # Modifications: <summary of changes>
  ```
- Use `locals` blocks rather than `variables.tf` for pattern-level configuration — patterns
  are meant for copy-paste adaptation, not module reuse with variable inputs
- Keep each pattern self-contained: `main.tf`, `vpc.tf`, `eks.tf`, component-specific `.tf`
  files, `outputs.tf`, `versions.tf`, `README.md`
- Check the upstream changelog for relevant fixes when upgrading component versions (Karpenter,
  LBC, EKS cluster version)

**DO NOT:**
- Modify upstream patterns in-place in a shared fork — copy the pattern into the project repo
  and own it
- Assume upstream defaults are correct for the project — verify IAM policies, subnet tags,
  and Helm values against project requirements before deploying
- Use a single monolithic `main.tf` for all infrastructure — split by component (vpc, eks,
  karpenter, lbc, observability) to enable targeted applies and reduce blast radius

**Known upstream gaps that must be addressed on fork:**
- LBC official IAM policy (v2.8.1) is missing `ec2:GetSecurityGroupsForVpc` — add an inline
  policy patch (see `lbc.tf` in `patterns/karpenter` for the implementation)
- `kubernetes_manifest` resources require a staged apply sequence — a bare `terraform apply`
  fails because CRDs don't exist until the Helm release runs (see ADR-003 and the pattern
  README for the exact apply sequence)

---

## Verification

```bash
# Confirm the pattern has a clear upstream reference in its header or README
head -10 patterns/<pattern-name>/main.tf
# Expected: comment documenting upstream source and fork-point

# Confirm the pattern is self-contained (no remote module sources other than community modules)
grep -r 'source\s*=' patterns/<pattern-name>/*.tf | grep -v 'terraform-aws-modules\|aws-ia'
# Expected: empty (only community modules as sources)

# Confirm locals are used for configuration, not variables
ls patterns/<pattern-name>/variables.tf 2>/dev/null && echo "WARN: variables.tf present" || echo "OK: no variables.tf"

# Confirm component separation
ls patterns/<pattern-name>/*.tf
# Expected: vpc.tf, eks.tf, karpenter.tf, lbc.tf, observability.tf (or similar component files)
```

---

## References

- [aws-ia/terraform-aws-eks-blueprints on GitHub](https://github.com/aws-ia/terraform-aws-eks-blueprints)
- [Terraform Recommended Module Structure](https://developer.hashicorp.com/terraform/language/modules/develop/structure)
- Related: [ADR-001 — IRSA over Node Instance Profiles](./ADR-001-irsa-over-node-instance-profiles.md)
