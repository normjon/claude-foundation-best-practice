# CLAUDE.md — claude-foundation-best-practice

## Primary Audience
Claude Code agents. Human engineers are the secondary audience.
All decisions are written as imperative rules, not suggestions.

## Design Philosophy
ADRs in this repository encode decisions that override general knowledge. When an ADR
conflicts with a simpler or more obvious approach, the ADR wins — it exists precisely
because the simpler approach was considered and rejected. Read the full ADR to understand
why before proposing an alternative. If you cannot find guidance for a decision you need
to make, stop, surface the gap, and create an ADR before proceeding.

## Folder Routing

| Work type | Read first | Then read |
|-----------|-----------|-----------|
| Provisioning AWS resources or EKS clusters | `security/` | `infrastructure/` |
| Writing Terraform for infrastructure | `infrastructure/` | `security/` |
| Deploying or updating any container image | `application/` | `security/` (IRSA) |
| Writing application code (API handlers, routes) | `application/` | `observability/` |
| Configuring log collection or writing log output | `observability/` | — |
| Creating branches, PRs, or CI/CD workflows | `process/` | — |
| Building Claude agents or MCP integrations | `ai-platform/` | `security/` (IRSA for API keys) |
| Writing any CLAUDE.md file | `process/` (ADR-016) | `ai-platform/` (ADR-021 for agent repos) |

## External Reference Libraries

| Library | URL | Use for |
|---------|-----|---------|
| Claude Cookbooks | https://github.com/anthropics/claude-cookbooks | Tool use, RAG, sub-agents, prompt caching, evaluations |
| Anthropic on AWS | https://github.com/aws-samples/anthropic-on-aws | Claude on Bedrock, EKS-hosted agents, AWS-specific patterns |
| EKS Blueprints | https://github.com/aws-ia/terraform-aws-eks-blueprints | Reference Terraform patterns for EKS |

## Global Rules

These rules apply regardless of domain. Each cites the ADR that defines it.

1. **Never commit directly to `main` or `develop`.** All changes via feature branch + PR. (ADR-013)
2. **Never use `latest` image tags.** Tag with `git rev-parse --short HEAD`. (ADR-009)
3. **Never put AWS credentials in code, env vars, or files.** Use IRSA. (ADR-001)
4. **All logs must be structured JSON to stdout** with `level`, `message`, `timestamp`. (ADR-003)
5. **Use the four-step staged Terraform apply** for fresh cluster deployments — never bare `terraform apply`. (ADR-005)
6. **All API parameters must be validated against an explicit allowlist** before any database operation. (ADR-010)
7. **Update README.md and CLAUDE.md in the same PR** as the code change they document. (ADR-015)
8. **Production Terraform applies run via CI/CD pipeline only** — never from a local machine. (ADR-017)

## ADR Status Definitions

| Status | Meaning |
|--------|---------|
| **Proposed** | Under discussion — not yet in use |
| **Accepted** | Active — all new code must follow this decision |
| **Deprecated** | Superseded — existing code may still use the old approach, but new code must not |
| **Superseded** | Replaced by a newer ADR (link provided in the document) |
