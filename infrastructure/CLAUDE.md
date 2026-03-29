# CLAUDE.md — Infrastructure

## Purpose
This folder governs EKS cluster provisioning, Terraform patterns, networking, compute architecture, and workload disruption management.

## Key Rules

1. **Start every new EKS pattern by forking the closest upstream blueprint from `aws-ia/terraform-aws-eks-blueprints`.** Document the upstream source and fork-point commit in the pattern header. Never consume blueprints as remote Terraform modules when customisation is expected. (ADR-002)

2. **All container images and Karpenter NodePools must target `linux/arm64` (Graviton).** Build with `--platform linux/arm64`. Include `kubernetes.io/arch: arm64` as the first NodePool requirement. Never add `amd64` to NodePool values. (ADR-004)

3. **Use the four-step staged Terraform apply sequence for fresh cluster deployments.** Never document or use bare `terraform apply` as the deployment procedure for a cluster that includes `kubernetes_manifest` CRD-dependent resources. Always wait for CRDs to be established between Step 3 and Step 4. (ADR-005)

4. **All internet-facing application ingresses must use IngressGroup annotation `group.name: platform`.** Never create a dedicated ALB unless there is a documented justification. Use `target-type: ip`, not `target-type: instance`. (ADR-006)

5. **Configure PDB, NodePool disruption budget, and rolling update strategy together.** For 3 replicas across 3 AZs: `minAvailable: 2`, `nodes: "1"`, `maxSurge: 0`, `maxUnavailable: 1`. Never set `maxSurge > 0` with strict pod anti-affinity. (ADR-008)

6. **Production Terraform applies must run via CI/CD pipeline only — never from a local machine.** Terraform plan output must be attached to the PR before apply is permitted. (ADR-017)

## ADR Index

| ADR | Title | One-line Summary |
|-----|-------|-----------------|
| [ADR-002](./ADR-002-upstream-reference-architecture.md) | Upstream Reference Architecture as Infrastructure Foundation | Fork EKS blueprints as the starting point; document all deviations |
| [ADR-004](./ADR-004-arm64-graviton-architecture.md) | arm64 (Graviton) as Required Container Architecture | All images and nodes must be arm64 — no amd64 mixing |
| [ADR-005](./ADR-005-staged-terraform-apply.md) | Staged Terraform Apply for CRD-Dependent Resources | Four-step apply sequence required for fresh clusters with kubernetes_manifest resources |
| [ADR-006](./ADR-006-shared-alb-ingressgroup.md) | Shared ALB via IngressGroup for Cluster-Wide Ingress | One ALB per cluster via `group.name: platform` annotation |
| [ADR-008](./ADR-008-pdb-nodepool-disruption-coordination.md) | Coordinated PDB, NodePool Budget, and Rolling Update Strategy | PDB + NodePool budget + rolling update strategy must be configured together |
| [ADR-017](./ADR-017-iac-governance.md) | Infrastructure as Code Governance | Terraform state isolation, cross-account references, and apply authorization |
| [ADR-020](./ADR-020-in-flight-session-protection.md) | In-Flight Session Protection During Infrastructure Updates | Drain active AgentCore sessions before updating dependent infrastructure |

## When to Read These ADRs

- Before writing any Terraform for an EKS cluster → read ADR-002, ADR-005
- Before writing any Dockerfile or CI build step → read ADR-004
- Before writing any Kubernetes Ingress resource → read ADR-006
- Before configuring a Deployment, PDB, or NodePool → read ADR-008
- Before running `terraform apply` in any environment → read ADR-017
- Before applying infrastructure changes that affect running sessions → read ADR-020
