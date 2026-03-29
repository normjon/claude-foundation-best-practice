# CLAUDE.md — Process

## Purpose
This folder governs branching strategy, CI/CD pipeline configuration, and documentation standards for all repositories on this platform.

## Key Rules

1. **Never commit directly to `main` or `develop`.** All changes flow through short-lived branches and pull requests. This applies to infrastructure, application, and documentation changes alike. (ADR-013)

2. **Use GitFlow for multi-team or infrastructure repositories.** GitHub Flow (one `main` branch) is only acceptable for single-app repositories with ≤5 developers and GitHub Environment approval gates configured. See ADR-013 for the full conditions and compensating controls. (ADR-013)

3. **Pin all third-party GitHub Actions to a commit SHA, not a floating version tag.** Never store `AWS_ACCESS_KEY_ID` in GitHub Secrets — use OIDC for AWS authentication exclusively. Apply `step-security/harden-runner` on every workflow job. (ADR-014)

4. **Every PR that changes architecture, APIs, deployment procedures, environment variables, or configuration must update `README.md` (and `CLAUDE.md` if applicable) in the same PR.** Documentation updates are not optional or deferred. (ADR-015)

5. **Keep each CLAUDE.md under 150 lines.** Write instructions as imperative commands. Link to ADRs for detail — do not inline full ADR content. (ADR-016)

6. **Update CLAUDE.md in the same PR as any change that affects agent behaviour.** Never contradict guidance across CLAUDE.md levels — if a conflict exists, resolve it; the more specific level takes precedence. (ADR-016)

## ADR Index

| ADR | Title | One-line Summary |
|-----|-------|-----------------|
| [ADR-013](./ADR-013-gitflow-branching-strategy.md) | GitFlow Branching Strategy | GitFlow for all repos; GitHub Flow only with explicit compensating controls |
| [ADR-014](./ADR-014-github-actions-pipeline.md) | GitHub Actions Pipeline Automation | Three-tier CI/CD pipeline; OIDC auth; pin all Actions to commit SHAs |
| [ADR-015](./ADR-015-readme-living-documentation.md) | README.md as Living Documentation | README and CLAUDE.md must be updated in the same PR as the code they describe |
| [ADR-016](./ADR-016-claude-agent-integration.md) | Claude Agent Integration via CLAUDE.md Hierarchy | Three-level CLAUDE.md hierarchy: global → repository → pattern |

## When to Read These ADRs

- Before creating any branch or opening any PR → read ADR-013
- Before writing or modifying any GitHub Actions workflow → read ADR-014
- Before completing any PR that changes code behaviour → read ADR-015 (documentation checklist)
- Before writing any CLAUDE.md at any level → read ADR-016
- When setting up a new repository on this platform → read ADR-013, ADR-014, ADR-015, ADR-016
