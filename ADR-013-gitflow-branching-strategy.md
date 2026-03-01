# ADR-013: GitFlow Branching Strategy

**Status:** Accepted
**Date:** 2026-03-01
**Scope:** Infrastructure + Application
**Tags:** git, branching, source-control, process, ci-cd, collaboration

---

## Context

Without a defined branching strategy, teams accumulate several failure modes: direct commits
to production branches that bypass review, long-lived branches that diverge and cause painful
merges, unclear release processes, and no standardized hotfix path. These problems compound
in mixed teams where infrastructure engineers and application developers work in the same
repository — or in adjacent repositories with coordinated deployments.

A branching strategy also determines how CI/CD pipelines are triggered (see ADR-014) and how
rollbacks are performed. The strategy must be simple enough for Claude agents to follow
without ambiguity.

---

## Decision

This project uses **GitFlow** as its branching strategy. The two long-lived branches are
`main` (production-ready code) and `develop` (integration branch). All work flows through
short-lived branches — `feature/*`, `release/*`, and `hotfix/*` — and merges via pull
requests. No direct commits to `main` or `develop` are permitted.

---

## Options Considered

### Option 1: Trunk-Based Development

**Description:** All developers commit directly to `main` (or to very short-lived feature
branches lasting < 1 day). Releases are tagged from `main`.

**Pros:**
- Simplest model — no branch management overhead
- Forces continuous integration — code is always integrated

**Cons:**
- Requires a highly mature CI/CD pipeline with feature flags to hide incomplete work
- Difficult to maintain a stable `main` when infrastructure changes are staged over hours
- Does not provide a natural integration branch for coordinating multiple feature branches
  before release

---

### Option 2: GitHub Flow

**Description:** One long-lived branch (`main`). Feature branches are created from `main`,
reviewed via PR, and merged directly to `main`.

**Pros:**
- Simpler than GitFlow — no `develop` branch
- Well-suited for continuous deployment where every merge to `main` deploys automatically

**Cons:**
- No integration buffer between feature work and production — a broken feature branch
  merged to `main` immediately affects production
- No formal release branch for final hardening before a production release
- No structured hotfix path that bypasses pending feature work

---

### Option 3: GitFlow ✓ *Selected*

**Description:** Two long-lived branches (`main`, `develop`). Short-lived branches
(`feature/*`, `release/*`, `hotfix/*`) with a defined lifecycle.

```
main ──────────────────────────────────────────► production
  ↑              ↑                    ↑
  │         release/x.y.z         hotfix/x.y.z
  │              ↑                    ↑
develop ──────────────────────────────────────► integration
  ↑
feature/my-feature
```

**Branch lifecycle:**

| Branch | Created from | Merges into | Purpose |
|--------|-------------|-------------|---------|
| `main` | — | — | Production-ready code only. Every commit is tagged. |
| `develop` | `main` | — | Integration branch. CI runs on every push. |
| `feature/<name>` | `develop` | `develop` | New features, bug fixes, ADR additions |
| `release/<version>` | `develop` | `main` + `develop` | Release stabilization, version bumps |
| `hotfix/<name>` | `main` | `main` + `develop` | Emergency production fixes |

**Pros:**
- `main` is always deployable — it only receives code that has passed integration testing
  on `develop`
- Release branches provide a stabilization period before production deployment
- Hotfix branches allow emergency fixes without pulling in pending feature work
- Well-understood model — tooling support (`git flow` CLI), documentation, and team
  familiarity are widespread

**Cons:**
- More overhead than GitHub Flow for small teams or simple repositories
- `develop` and `main` can diverge if hotfixes are not merged back to `develop` promptly
- More branch management — engineers must remember to create branches from the correct base

---

## Consequences

### Positive
- Production (`main`) is protected from incomplete or untested feature work
- Clear process for all three change types: features, releases, emergencies
- CI/CD pipelines can target branches by pattern (see ADR-014)

### Trade-offs
- Hotfixes must be merged into both `main` and `develop` — forgetting the `develop` merge
  reintroduces the bug in the next release
- Release branches require discipline to limit to stabilization work only — feature additions
  to a release branch defeat the purpose

---

## Implementation Rules

> These rules apply to all Claude agents and engineers working in this repository.

**Branch naming:**
```
feature/<short-description>       # e.g., feature/add-user-search
release/<semver>                  # e.g., release/1.2.0
hotfix/<short-description>        # e.g., hotfix/irsa-trust-policy-fix
```

**Workflow — new feature or ADR:**
```bash
git checkout develop
git pull origin develop
git checkout -b feature/<name>
# ... make changes, commit, push ...
# Open PR: feature/<name> → develop
# Merge after review (squash merge preferred for clean history)
git branch -d feature/<name>
```

**Workflow — release:**
```bash
git checkout develop
git pull origin develop
git checkout -b release/<version>
# Bump version numbers, update CHANGELOG, update README if needed
git push origin release/<version>
# Open PR: release/<version> → main
# After merge to main, tag main:
git checkout main && git pull && git tag -a v<version> -m "Release v<version>"
git push origin v<version>
# Back-merge to develop:
git checkout develop && git merge main && git push origin develop
```

**Workflow — hotfix:**
```bash
git checkout main
git pull origin main
git checkout -b hotfix/<name>
# ... fix the issue, commit, push ...
# Open PR: hotfix/<name> → main
# After merge, also open PR: hotfix/<name> → develop (prevents regression)
```

**DO:**
- Always create branches from the correct base (`feature/*` and `release/*` from `develop`;
  `hotfix/*` from `main`)
- Open a PR for every merge — direct pushes to `main` or `develop` are prohibited
- Write a PR description explaining *what* changed and *why* (not just *how*)
- After merging a hotfix to `main`, immediately open a second PR to merge it into `develop`
- Tag `main` with a semver version tag after every release merge

**DO NOT:**
- Commit directly to `main` or `develop`
- Keep feature branches open longer than one sprint — long-lived branches diverge and cause
  painful merges
- Add new features to a `release/*` branch — it is for stabilization only
- Delete `develop` or merge `develop` into `main` manually — use a PR

---

## Verification

```bash
# Confirm branch protection rules are enabled
gh api repos/:owner/:repo/branches/main/protection \
  --jq '{required_reviews: .required_pull_request_reviews.required_approving_review_count, dismiss_stale: .required_pull_request_reviews.dismiss_stale_reviews}'
# Expected: required_reviews >= 1

gh api repos/:owner/:repo/branches/develop/protection \
  --jq '{required_reviews: .required_pull_request_reviews.required_approving_review_count}'
# Expected: required_reviews >= 1

# Confirm there are no direct commits to main (no commits without associated PRs)
gh pr list --state merged --base main --limit 10
# All merges to main should appear as closed PRs

# Confirm feature branches target develop, not main
gh pr list --state open | grep feature/
# Expected: all open feature PRs target develop
```

---

## References

- [GitFlow original blog post (Vincent Driessen)](https://nvie.com/posts/a-successful-git-branching-model/)
- [Atlassian GitFlow workflow](https://www.atlassian.com/git/tutorials/comparing-workflows/gitflow-workflow)
- [GitHub branch protection rules](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches)
- Related: [ADR-014 — GitHub Actions Pipeline Automation](./ADR-014-github-actions-pipeline.md)
- Related: [ADR-009 — Git SHA Image Tags](./ADR-009-git-sha-image-tags.md)
