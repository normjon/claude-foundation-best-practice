# ADR-015: README.md as Living Documentation

**Status:** Accepted
**Date:** 2026-03-01
**Scope:** Infrastructure + Application
**Tags:** documentation, process, maintainability, onboarding, claude-agents

---

## Context

Documentation that does not stay current with code becomes actively harmful: it misleads
engineers, causes failed deployments when following outdated procedures, and forces every
new contributor (human or AI agent) to reverse-engineer the correct state by reading the
code rather than trusting the documentation.

This problem is amplified when Claude Code agents use documentation as a primary input.
An agent that reads a stale README and follows its instructions may deploy incorrectly,
use the wrong command sequence, or make decisions based on an obsolete architecture.

Three files serve as the living documentation contract for each component:
- `README.md` — architecture, deployment procedures, configuration reference, troubleshooting
- `CLAUDE.md` — agent-specific guidance: commands, constraints, anti-patterns, verification
- `openapi.yaml` (for APIs) — machine-readable API specification

---

## Decision

Every pull request that changes architecture, APIs, deployment procedures, configuration
values, environment variables, or operational runbooks **must** update the corresponding
`README.md` (and `CLAUDE.md` if applicable) in the same PR. Documentation updates are not
optional or deferred — they are part of the definition of done for any change.

---

## Options Considered

### Option 1: No documentation requirement

**Description:** Write code; update docs when convenient or when someone complains they
are wrong.

**Pros:**
- Zero documentation overhead per PR

**Cons:**
- Documentation drifts within days of any significant change
- New contributors (and Claude agents) can't trust any written procedure
- Debugging requires reading code, not following runbooks
- Claude agents acting on stale documentation amplify the damage — incorrect automated
  actions are harder to reverse than incorrect human actions

---

### Option 2: Separate documentation repository or wiki

**Description:** Maintain documentation in a separate wiki (Confluence, Notion, GitHub Wiki)
or a dedicated docs repository.

**Pros:**
- Docs can be updated without a code PR
- Non-engineers can contribute without touching the codebase

**Cons:**
- Separation breaks atomicity — a code change and its documentation change can be in
  different places, different reviews, different commits
- Wikis drift faster than co-located docs because there is no CI check to enforce them
- Claude agents must access an external system to get current documentation

---

### Option 3: README.md co-located with code, updated in the same PR ✓ *Selected*

**Description:** Documentation lives alongside the code it describes (`README.md` in the
same directory). Every PR that changes the documented behavior must update the `README.md`
in the same commit. `CLAUDE.md` is updated whenever agent guidance changes. `openapi.yaml`
is updated whenever the API contract changes.

**Pros:**
- Documentation changes are reviewed alongside code changes in the same PR
- Git history shows documentation and code evolving together — `git log README.md` traces
  the documentation history
- `markdown-link-check.yml` CI workflow validates all Markdown links on every PR
- Claude agents always have access to current documentation via `Read` tool — no external
  system required
- `terraform-docs` (run by pre-commit) auto-generates the Terraform inputs/outputs
  section of `README.md` — reduces manual maintenance for the most error-prone section

**Cons:**
- Engineers must remember to update the README — this requires discipline and PR review
  enforcement
- Large architectural changes may require significant README rewrites alongside the code

---

## Consequences

### Positive
- Claude agents operating in a repository can trust that `README.md` reflects the current
  architecture and that `CLAUDE.md` reflects current commands and constraints
- Onboarding new engineers or agents requires only reading the README — no knowledge
  transfer session or wiki archaeology
- Documentation is versioned in git — checking out any previous commit gives the
  documentation that was accurate at that point in time

### Trade-offs
- PR review must include a documentation check — reviewers must ask "does the README need
  updating?" for every change
- Auto-generated sections (`terraform-docs`) require the pre-commit hook to run and the
  author to commit the generated output

---

## Implementation Rules

> These rules apply to all Claude agents and engineers making changes in any repository.

**Required README.md sections (minimum for every component):**

```markdown
# Component Name

## Architecture
[Diagram or description of the system and how components interact]

## Key Design Decisions
[Table of decisions with rationale — links to relevant ADRs]

## Project Structure
[Directory tree with purpose of each significant file]

## Prerequisites
[What must exist before deploying this component]

## Deploy / Quick Start
[Step-by-step deploy procedure — exact commands, not descriptions]

## Configuration Reference
[All environment variables, Helm values, Terraform locals]

## Verification
[How to confirm the deployment succeeded]

## Known Issues & Troubleshooting
[Documented failure modes with symptoms, causes, and fixes]

## Destroy / Cleanup
[How to safely tear down the component]
```

**Triggers that require a README.md update (as part of the same PR):**

| Change | Required documentation update |
|--------|-------------------------------|
| New environment variable | Add to Configuration Reference table |
| Changed deploy command or sequence | Update Deploy section |
| New AWS resource created | Update Architecture diagram/description |
| Changed IAM permission | Update relevant section; check ADR-001 |
| New API endpoint | Update `openapi.yaml` and API Reference |
| New troubleshooting scenario discovered | Add to Known Issues |
| Component renamed | Update all references in README.md and CLAUDE.md |
| Dependency version change | Update Tech Stack table |

**CLAUDE.md update triggers:**

| Change | Required CLAUDE.md update |
|--------|--------------------------|
| New verification command discovered | Add to Verification Protocol section |
| New anti-pattern discovered | Add to Architecture Anti-Patterns section |
| Command changed (CLI, Helm, kubectl) | Update Common Commands section |
| New constraint discovered | Add to relevant section |

**openapi.yaml update triggers (for API repositories):**

| Change | Required openapi.yaml update |
|--------|------------------------------|
| New endpoint added | New path entry |
| Request/response body changed | Update schema |
| New error response added | Add to responses |
| Parameter added, removed, or renamed | Update parameters section |

**DO:**
- Include documentation changes in the same commit as the code changes they describe
- Use `terraform-docs` (run by pre-commit hook) to auto-generate Terraform input/output
  tables — do not manually write or edit these sections
- Use concrete examples in the README (actual `curl` commands, actual `kubectl` output)
  not abstract descriptions
- Keep the Troubleshooting section current — add entries when debugging a production issue

**DO NOT:**
- Open a separate "docs PR" for documentation that should have been in a code PR
- Write documentation in future tense ("will support X") — document only current reality
- Reference external hostnames, ARNs, or account IDs that change between environments
  without noting that they are environment-specific
- Delete a Troubleshooting entry because the underlying issue was fixed — mark it as
  "resolved in version X" and explain the fix

---

## Verification

The following checks are automated in CI:

```bash
# Markdown link validation (runs on every PR via markdown-link-check.yml)
npx markdown-link-check README.md --config .mlc_config.json

# Terraform docs are current (runs via pre-commit terraform_docs hook)
terraform-docs markdown table . > /tmp/docs-output.md
diff /tmp/docs-output.md README.md
# Expected: no diff in the auto-generated section

# Spelling check (runs via pre-commit cspell hook)
cspell "**/*.md"
```

Manual review checklist for PR authors:

```
[ ] Architecture changes are reflected in README.md Architecture section
[ ] New or changed commands are in the Deploy / Quick Start section
[ ] New environment variables are in the Configuration Reference
[ ] New troubleshooting scenarios are in Known Issues
[ ] CLAUDE.md Verification Protocol is current
[ ] openapi.yaml is updated if any API endpoint changed
```

---

## References

- [Divio documentation system](https://documentation.divio.com/) — tutorials, how-to guides,
  explanation, reference (the four documentation types)
- [terraform-docs](https://terraform-docs.io/)
- [OpenAPI Specification](https://spec.openapis.org/oas/v3.1.0)
- Related: [ADR-016 — Claude Agent Integration](./ADR-016-claude-agent-integration.md)
- Related: [ADR-013 — GitFlow Branching Strategy](./ADR-013-gitflow-branching-strategy.md)
