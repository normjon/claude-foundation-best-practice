# ADR-016: Claude Agent Integration via CLAUDE.md Hierarchy

**Status:** Accepted
**Date:** 2026-03-01
**Scope:** Infrastructure + Application
**Tags:** ai-agents, claude, automation, documentation, developer-experience, anthropic

---

## Context

Claude Code (Anthropic's AI coding assistant) is used as a development partner by engineers
on this platform — writing Terraform, application code, Helm charts, and operational runbooks.
Without structured guidance, agents make decisions based on general knowledge rather than
project-specific patterns, producing code that:
- Uses node instance profiles instead of IRSA
- Builds amd64 images instead of arm64
- Skips the staged Terraform apply sequence
- Writes unstructured logs instead of structured JSON
- Creates new ALBs instead of using the shared IngressGroup

Claude Code reads a special file named `CLAUDE.md` at multiple levels of the filesystem
hierarchy. This file is loaded automatically into the agent's context at the start of every
session — no explicit instruction from the user is required.

External reference collections from Anthropic and AWS document patterns for building
Claude-powered systems on AWS infrastructure:
- **[claude-cookbooks](https://github.com/anthropics/claude-cookbooks)** — Anthropic's official
  collection of Claude API patterns: tool use, RAG, sub-agents, prompt caching, evaluations
- **[anthropic-on-aws](https://github.com/aws-samples/anthropic-on-aws)** — AWS samples for
  running Claude workloads on AWS: agent SDK implementations, Bedrock integration, EKS-hosted
  Claude services, workshop materials

---

## Decision

Claude agent guidance is organized in a **three-level CLAUDE.md hierarchy**. Each level adds
context without duplicating information from higher levels. All project-specific decisions
reference the ADR library in `docs/decisions/`. External pattern libraries are referenced by
URL but are not duplicated locally.

```
~/.claude/CLAUDE.md              ← Global: org-wide standards, credentials, ADR index
  ↓
<repo>/CLAUDE.md                 ← Repository: architecture, commands, CI/CD, key constraints
  ↓
<repo>/patterns/<name>/CLAUDE.md ← Pattern: deploy/destroy sequences, verification, known issues
```

---

## Options Considered

### Option 1: No CLAUDE.md files — rely on general Claude knowledge

**Description:** Do not add any agent-specific guidance. Claude Code uses its training data
and reads the codebase on demand.

**Pros:**
- Zero maintenance overhead
- Claude's general knowledge covers standard practices

**Cons:**
- Every session rediscovers the same project-specific constraints (staged apply, IRSA, arm64)
- Agents produce code that follows general best practices, not project-specific ones
- No way to encode hard rules that override general knowledge (e.g., "never use latest tags")

---

### Option 2: Single CLAUDE.md at repository root

**Description:** One comprehensive CLAUDE.md at the repository root covering everything.

**Pros:**
- Single file to maintain
- Always loaded regardless of working directory

**Cons:**
- Grows too large — a 2000+ line CLAUDE.md overflows the context window
  (Claude Code truncates files longer than 200 lines in `MEMORY.md`; long CLAUDE.md files
  are similarly unwieldy)
- Pattern-specific sequences (deploy/destroy) are too long to include in a single file
  without displacing other important context

---

### Option 3: Three-level CLAUDE.md hierarchy + ADR reference ✓ *Selected*

**Description:**

**Level 1 — Global (`~/.claude/CLAUDE.md`):**
- Organization-wide standards (credential management, AWS SSO pattern)
- ADR index with one-line summaries for the most critical decisions
- Pointers to external reference libraries

**Level 2 — Repository (`<repo>/CLAUDE.md`):**
- Repository overview and purpose
- Common commands (fmt, validate, lint)
- Key style and safety rules
- Brief ADR index table (links to `docs/decisions/`)
- CI/CD overview

**Level 3 — Pattern (`<repo>/patterns/<name>/CLAUDE.md` or co-located `CLAUDE.md`):**
- The exact deploy sequence for this specific pattern
- The exact destroy sequence
- Verification commands
- Known issues and workarounds specific to this pattern

**Pros:**
- Each level stays concise — agents get exactly the context relevant to their working directory
- ADRs in `docs/decisions/` provide full detail on demand without polluting the context window
- External reference libraries are cited but not duplicated — agents can fetch them when building
  Claude-powered features
- The hierarchy is self-documenting: adding a new pattern adds a new Level 3 file

**Cons:**
- Three files to maintain instead of one
- Changes to project-wide rules must be reflected at the correct level

---

## Consequences

### Positive
- Claude agents working at the pattern level get pattern-specific deploy/destroy sequences
  automatically, without requiring the user to provide them
- The ADR index in CLAUDE.md gives every agent the one-line rule for each decision, with a
  link to the full rationale — agents that need more context can read the full ADR
- External library references enable agents building AI-powered features to follow established
  AWS + Anthropic patterns rather than inventing their own

### Trade-offs
- CLAUDE.md files must be updated when decisions change (see ADR-015)
- The global `~/.claude/CLAUDE.md` is not version-controlled in the project repository —
  it must be documented and distributed separately (e.g., via a team onboarding script or
  a separate standards repository)

---

## Implementation Rules

> These rules apply to all engineers setting up development environments and Claude agent workflows.

**Global CLAUDE.md (`~/.claude/CLAUDE.md`) — recommended content:**

```markdown
# Organization Standards

## AWS Credentials
Run `aws sso login --profile <your-sso-profile>` to refresh SSO session when credentials expire.
Never store long-lived credentials in environment variables or files.

## Architecture Decision Records
Before designing infrastructure or application code, read the
relevant domain folder CLAUDE.md first, then consult the full ADR:
- security/        — credential delivery, input validation, IAM policy
- infrastructure/  — Terraform, compute, networking, session protection
- application/     — image tagging, path routing, container traceability
- observability/   — structured logging, log collection
- process/         — branching, CI/CD, documentation standards
- ai-platform/     — Claude agent integration, CLAUDE.md hierarchy

Key rules (read the full ADR for rationale and verification):
- ADR-001 (security/):       Use IRSA — never node instance profiles or env vars
- ADR-003 (observability/):  All logs must be structured JSON to stdout
- ADR-009 (application/):    Image tags must be git SHA — never 'latest'
- ADR-013 (process/):        GitFlow branching — never commit directly to main or develop
- ADR-015 (process/):        README.md must be updated in the same PR as the code it documents
- ADR-017 (infrastructure/): Each AWS account has one state file — never share state across accounts
- ADR-018 (security/):       Validate all MCP Gateway inputs against declared schema before execution
- ADR-021 (ai-platform/):    Agent repositories must follow the three-level CLAUDE.md hierarchy

## External Reference Libraries
- Claude patterns: https://github.com/anthropics/claude-cookbooks
- Claude on AWS: https://github.com/aws-samples/anthropic-on-aws
- EKS blueprints: https://github.com/aws-ia/terraform-aws-eks-blueprints
```

**Repository CLAUDE.md — ADR index section:**
Already implemented. Maintain the table in `CLAUDE.md` as new ADRs are added.

**Pattern-level CLAUDE.md (`patterns/<name>/CLAUDE.md`) — required sections:**
```markdown
# CLAUDE.md — <Pattern Name>

## Working Directory
Always run terraform commands from within `patterns/<name>/`.

## Deploy Sequence
[exact staged apply sequence — see ADR-005]

## Destroy Sequence
[exact staged destroy sequence]

## Verification
[verification commands from the pattern README]

## Known Issues
[pattern-specific gotchas not in the main CLAUDE.md]
```

**DO:**
- Keep each CLAUDE.md level under 150 lines — link to ADRs for detail, don't inline them
- Write CLAUDE.md instructions as imperative commands, not suggestions
  ("Run `aws sso login --profile <your-sso-profile>` to refresh credentials" not "You may want to refresh credentials")
- Update CLAUDE.md in the same PR as any change that affects agent behavior (see ADR-015)
- Reference external pattern libraries by URL — do not copy their content locally

**Referencing external libraries in agent workflows:**

When building Claude-powered features (API integrations, agent pipelines, evaluation systems):

```
1. Start with claude-cookbooks for the API pattern (tool use, RAG, sub-agents, etc.)
   → https://github.com/anthropics/claude-cookbooks

2. Adapt to AWS infrastructure using anthropic-on-aws patterns (Bedrock, EKS hosting, IAM)
   → https://github.com/aws-samples/anthropic-on-aws

3. Follow ADR-001 (IRSA) for Claude API key management — store in Secrets Manager,
   access via IRSA-scoped role, never in environment variables

4. Follow ADR-003 (structured JSON logging) for Claude API call logging
   — log model, input_tokens, output_tokens, latency_ms at INFO level
```

**DO NOT:**
- Put environment-specific values (account IDs, ARNs, hostnames) in CLAUDE.md — they change
  per environment and will cause agents to use stale values
- Write contradictory guidance across CLAUDE.md levels — if Level 2 and Level 3 conflict,
  Level 3 (more specific) takes precedence, but the conflict should be resolved
- Use CLAUDE.md as a substitute for proper documentation in README.md — CLAUDE.md is for
  agent guidance; README.md is for human documentation (they overlap but serve different readers)

---

## Verification

```bash
# Confirm CLAUDE.md exists at the repository level
ls -la CLAUDE.md
# Expected: exists, recently modified

# Confirm CLAUDE.md contains ADR index
grep "ADR-0" CLAUDE.md | wc -l
# Expected: > 5 ADR entries

# Confirm global CLAUDE.md exists
ls -la ~/.claude/CLAUDE.md
# Expected: exists

# Confirm each pattern has a CLAUDE.md or the root README covers the pattern
for dir in patterns/*/; do
  if [ ! -f "${dir}CLAUDE.md" ] && [ ! -f "${dir}README.md" ]; then
    echo "MISSING docs: $dir"
  fi
done
# Expected: no output (all patterns have README.md at minimum)

# Confirm external library URLs are reachable
curl -s -o /dev/null -w "%{http_code}" https://github.com/anthropics/claude-cookbooks
# Expected: 200

curl -s -o /dev/null -w "%{http_code}" https://github.com/aws-samples/anthropic-on-aws
# Expected: 200
```

---

## References

- [Anthropic Claude Cookbooks](https://github.com/anthropics/claude-cookbooks) — Official
  Claude API patterns: tool use, RAG, sub-agents, prompt caching, evaluations, multimodal
- [Anthropic on AWS](https://github.com/aws-samples/anthropic-on-aws) — Claude workloads on
  AWS: Agent SDK on EKS, Bedrock integration, workshop materials, IDP, computer vision
- [Claude Code documentation](https://docs.anthropic.com/en/docs/claude-code)
- [CLAUDE.md format reference](https://docs.anthropic.com/en/docs/claude-code#claudemd-files)
- Related: [ADR-015 — README.md as Living Documentation](../process/ADR-015-readme-living-documentation.md)
- Related: [ADR-001 — IRSA](../security/ADR-001-irsa-over-node-instance-profiles.md)
