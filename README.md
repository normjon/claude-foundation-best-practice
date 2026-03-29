# Architecture Decision Records — claude-foundation-best-practice

## Primary Audience

Claude Code agents. Human engineers are the secondary audience. All ADRs are written as
imperative rules — when an ADR conflicts with a simpler approach, the ADR wins because the
simpler approach was considered and rejected. Read the full ADR for the rationale.

## Folder Structure

ADRs are organised into domain folders. Each folder contains a `CLAUDE.md` that summarises
the key rules for that domain and lists which ADRs to read first.

```
security/        IAM credential delivery, API input validation, IAM policy governance
infrastructure/  EKS provisioning, Terraform patterns, networking, compute, disruption
application/     Container image tagging, path routing, container traceability
observability/   Structured logging, log collection infrastructure
process/         Branching strategy, CI/CD pipeline, documentation standards
ai-platform/     Claude agent integration, CLAUDE.md hierarchy, MCP gateway
```

**Placement rule for new ADRs:** An ADR lives in the folder of its primary concern. If it
touches multiple domains, add a cross-reference in the affected folders' `CLAUDE.md` files
under the ADR Index section.

## How to Add a New ADR

1. Choose the folder whose domain best matches the decision.
2. Copy the template at the bottom of this file into `<folder>/ADR-NNN-short-title.md`.
3. Fill in all sections — especially **Options Considered** and **Implementation Rules**.
4. Set status to `Proposed`; open a PR targeting `develop` (per ADR-013).
5. Update the index table in this README once the ADR is accepted.
6. Add a row to the affected folder's `CLAUDE.md` ADR Index table.
7. Add a cross-reference in other folders' `CLAUDE.md` files if the ADR affects multiple domains.

---

## ADR Index

### Security

| ADR | Title | Status |
|-----|-------|--------|
| [ADR-001](./security/ADR-001-irsa-over-node-instance-profiles.md) | IRSA over Node Instance Profiles for AWS Credential Delivery | Accepted |
| [ADR-010](./security/ADR-010-api-input-allowlisting.md) | API Input Allowlisting and Database Scan Limits | Accepted |
| [ADR-012](./security/ADR-012-iam-policy-gap-remediation.md) | Upstream IAM Policy Gap Remediation Pattern | Accepted |
| [ADR-018](./security/ADR-018-mcp-gateway-input-validation.md) | MCP Gateway Input Validation | Accepted |

### Infrastructure

| ADR | Title | Status |
|-----|-------|--------|
| [ADR-002](./infrastructure/ADR-002-upstream-reference-architecture.md) | Upstream Reference Architecture as the Infrastructure Foundation | Accepted |
| [ADR-004](./infrastructure/ADR-004-arm64-graviton-architecture.md) | arm64 (Graviton) as the Required Container Architecture | Accepted |
| [ADR-005](./infrastructure/ADR-005-staged-terraform-apply.md) | Staged Terraform Apply for CRD-Dependent Resources | Accepted |
| [ADR-006](./infrastructure/ADR-006-shared-alb-ingressgroup.md) | Shared ALB via IngressGroup for Cluster-Wide Ingress | Accepted |
| [ADR-008](./infrastructure/ADR-008-pdb-nodepool-disruption-coordination.md) | Coordinated PDB, NodePool Budget, and Rolling Update Strategy | Accepted |
| [ADR-017](./infrastructure/ADR-017-iac-governance.md) | Infrastructure as Code Governance | Accepted |
| [ADR-020](./infrastructure/ADR-020-in-flight-session-protection.md) | In-Flight Session Protection During Infrastructure Updates | Accepted |

### Application

| ADR | Title | Status |
|-----|-------|--------|
| [ADR-007](./application/ADR-007-app-base-path-ssot.md) | APP_BASE_PATH as Single Source of Truth for Path Routing | Accepted |
| [ADR-009](./application/ADR-009-git-sha-image-tags.md) | Git SHA Image Tags for Container Images | Accepted |
| [ADR-019](./application/ADR-019-container-image-traceability.md) | Container Image Traceability | Accepted |

### Observability

| ADR | Title | Status |
|-----|-------|--------|
| [ADR-003](./observability/ADR-003-structured-json-logging.md) | Structured JSON Logging to stdout | Accepted |
| [ADR-011](./observability/ADR-011-dual-observability-paths.md) | Dual Observability Paths for Fargate and EC2 Node Types | Accepted |

### Process

| ADR | Title | Status |
|-----|-------|--------|
| [ADR-013](./process/ADR-013-gitflow-branching-strategy.md) | GitFlow Branching Strategy | Accepted |
| [ADR-014](./process/ADR-014-github-actions-pipeline.md) | GitHub Actions Pipeline Automation | Accepted |
| [ADR-015](./process/ADR-015-readme-living-documentation.md) | README.md as Living Documentation | Accepted |

### AI Platform

| ADR | Title | Status |
|-----|-------|--------|
| [ADR-016](./ai-platform/ADR-016-claude-agent-integration.md) | Claude Agent Integration via CLAUDE.md Hierarchy | Accepted |
| [ADR-021](./ai-platform/ADR-021-ai-platform-claude-md-hierarchy.md) | AI Platform CLAUDE.md Hierarchy | Accepted |

---

## ADR Status Definitions

| Status | Meaning |
|--------|---------|
| **Proposed** | Under discussion — not yet in use |
| **Accepted** | Active — all new code must follow this decision |
| **Deprecated** | Superseded — existing code may still use the old approach, but new code must not |
| **Superseded** | Replaced by a newer ADR (link provided in the document) |

---

## ADR Template

```markdown
# ADR-NNN: [Title]

**Status:** Proposed
**Date:** YYYY-MM-DD
**Scope:** Security | Infrastructure | Application | Observability | Process | AI Platform | Multiple
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

[Runnable commands to confirm the pattern is correctly implemented]

## References

- ...
```
