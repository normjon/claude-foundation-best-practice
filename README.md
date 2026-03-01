# Architecture Decision Records

This directory contains Architecture Decision Records (ADRs) for the EKS Blueprints platform.
ADRs capture significant decisions made about the platform's architecture, security posture,
and operational practices — including the context, alternatives considered, and consequences.

## Purpose

These ADRs serve as the authoritative reference for:
- **Developers** building applications on the platform
- **Engineers** extending the infrastructure patterns
- **Claude agents** generating code, configuration, or documentation for this platform

When a decision documented here conflicts with a simpler or more obvious approach, the ADR
explains why the simpler approach was rejected. Follow the decision as stated unless an ADR
has been superseded.

---

## Index

### Security

| ADR | Title | Status |
|-----|-------|--------|
| [ADR-001](./ADR-001-irsa-over-node-instance-profiles.md) | IRSA over Node Instance Profiles for AWS Credential Delivery | Accepted |
| [ADR-010](./ADR-010-api-input-allowlisting.md) | API Input Allowlisting and Database Scan Limits | Accepted |
| [ADR-012](./ADR-012-iam-policy-gap-remediation.md) | Upstream IAM Policy Gap Remediation Pattern | Accepted |

### Infrastructure & Architecture

| ADR | Title | Status |
|-----|-------|--------|
| [ADR-002](./ADR-002-upstream-reference-architecture.md) | Upstream Reference Architecture as the Infrastructure Foundation | Accepted |
| [ADR-004](./ADR-004-arm64-graviton-architecture.md) | arm64 (Graviton) as the Required Container Architecture | Accepted |
| [ADR-005](./ADR-005-staged-terraform-apply.md) | Staged Terraform Apply for CRD-Dependent Resources | Accepted |
| [ADR-006](./ADR-006-shared-alb-ingressgroup.md) | Shared ALB via IngressGroup for Cluster-Wide Ingress | Accepted |
| [ADR-008](./ADR-008-pdb-nodepool-disruption-coordination.md) | Coordinated PDB, NodePool Budget, and Rolling Update Strategy | Accepted |

### Application

| ADR | Title | Status |
|-----|-------|--------|
| [ADR-007](./ADR-007-app-base-path-ssot.md) | APP_BASE_PATH as Single Source of Truth for Path Routing | Accepted |
| [ADR-009](./ADR-009-git-sha-image-tags.md) | Git SHA Image Tags for Container Images | Accepted |

### Observability

| ADR | Title | Status |
|-----|-------|--------|
| [ADR-003](./ADR-003-structured-json-logging.md) | Structured JSON Logging to stdout | Accepted |
| [ADR-011](./ADR-011-dual-observability-paths.md) | Dual Observability Paths for Fargate and EC2 Node Types | Accepted |

### Process & Automation

| ADR | Title | Status |
|-----|-------|--------|
| [ADR-013](./ADR-013-gitflow-branching-strategy.md) | GitFlow Branching Strategy | Accepted |
| [ADR-014](./ADR-014-github-actions-pipeline.md) | GitHub Actions Pipeline Automation | Accepted |
| [ADR-015](./ADR-015-readme-living-documentation.md) | README.md as Living Documentation | Accepted |
| [ADR-016](./ADR-016-claude-agent-integration.md) | Claude Agent Integration via CLAUDE.md Hierarchy | Accepted |

---

## ADR Status Definitions

| Status | Meaning |
|--------|---------|
| **Proposed** | Under discussion — not yet in use |
| **Accepted** | Active — all new code must follow this decision |
| **Deprecated** | Superseded — existing code may still use the old approach, but new code must not |
| **Superseded** | Replaced by a newer ADR (link provided in the document) |

---

## Adding a New ADR

1. Copy the template below into a new file: `ADR-NNN-short-descriptive-title.md`
2. Fill in all sections — especially **Options Considered** and **Implementation Rules**
3. Set status to `Proposed` and open a PR targeting `develop` (per ADR-013)
4. Update this index table once the ADR is accepted
5. Add a row to the ADR index table in the repository root `CLAUDE.md`

```markdown
# ADR-NNN: [Title]

**Status:** Proposed
**Date:** YYYY-MM-DD
**Scope:** Infrastructure | Application | Both
**Tags:** tag1, tag2

---

## Context

[2-4 sentences describing the situation and why a decision is needed]

## Decision

[1-2 sentences stating what was decided, in clear imperative terms]

## Options Considered

### Option 1: [Name]
**Pros:** ...
**Cons:** ...

### Option 2: [Name] ✓ *Selected*
**Pros:** ...
**Cons:** ...

## Consequences

### Positive
- ...

### Trade-offs
- ...

## Implementation Rules

**DO:**
- ...

**DO NOT:**
- ...

## Verification

[Commands to confirm the pattern is correctly implemented]

## References

- ...
```
