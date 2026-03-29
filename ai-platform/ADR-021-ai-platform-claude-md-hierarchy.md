# ADR-021: AI Platform CLAUDE.md Hierarchy

**Status:** Accepted
**Date:** 2026-03-29
**Scope:** Infrastructure + Application
**Tags:** ai-agents, claude, documentation, claude-md, hierarchy, agent-repositories

---

## Context

ADR-016 established the three-level CLAUDE.md hierarchy (global → repository → pattern)
for all repositories on this platform. Agent repositories built on the AI platform have
additional context requirements beyond what ADR-016 specifies for general repositories:
each agent has a manifest, a target Knowledge Base, a set of MCP tools, a data classification
ceiling, and a promotion pipeline. Without standardising what belongs in each level of the
hierarchy for agent repositories, agents operating in those repositories either lack critical
context or receive it in inconsistent forms.

This ADR extends ADR-016 with the content requirements specific to agent repositories. It
does not replace ADR-016 — the three-level structure and all rules from ADR-016 continue to
apply. This ADR adds content requirements at each level.

---

## Decision

The three-level CLAUDE.md hierarchy from ADR-016 applies to all agent repositories.
This ADR extends it with AI platform-specific content requirements at each level:

- **Global CLAUDE.md** must include the platform knowledge base URL and the documentation
  gap resolution instruction
- **Repository CLAUDE.md** for an agent repository must include the agent's manifest
  location, target Knowledge Base ID, available MCP tools, data classification ceiling,
  and promotion pipeline configuration
- **Pattern CLAUDE.md** files apply to agent types — one per agent type documented in the
  platform knowledge base — covering the exact scaffolding sequence, quality gate thresholds,
  and known anti-patterns for that agent type

All three levels must reference the platform documentation repository URL as the authoritative
source for platform decisions.

---

## Options Considered

### Option 1: No additional content requirements beyond ADR-016

**Description:** Apply ADR-016's hierarchy to agent repositories exactly as written for
general repositories. Let each team decide what agent-specific content to include.

**Pros:**
- No additional standards to maintain
- Flexibility for agent teams to document what matters to them

**Cons:**
- An agent operating in a repository without its manifest location, Knowledge Base ID, or
  MCP tool list in CLAUDE.md will discover these by reading code — slower and error-prone
- Different agent teams encode the same types of information in different places — an agent
  working across multiple agent repositories cannot rely on consistent structure
- Documentation gaps in agent repositories are harder to detect without a required content checklist

---

### Option 2: Centralised agent configuration file (separate from CLAUDE.md)

**Description:** Define a standard `agent.config.json` file at the repository root containing
all platform-specific metadata (manifest location, Knowledge Base ID, MCP tools, etc.).
CLAUDE.md files remain general per ADR-016.

**Pros:**
- Machine-readable configuration — tooling can validate and parse it
- Separates agent metadata from agent guidance

**Cons:**
- Agents read CLAUDE.md, not `agent.config.json` — metadata not in CLAUDE.md must be
  explicitly fetched, adding steps to every agent session start
- Two files to maintain instead of one — content can diverge
- Agents without CLAUDE.md guidance still lack context on how to use the metadata

---

### Option 3: CLAUDE.md content requirements specific to agent repositories ✓ *Selected*

**Description:** Extend ADR-016's three levels with agent-repository-specific required content
at each level. Claude agents reading CLAUDE.md at any level get both the guidance (how to act)
and the configuration (what to act on) in the same file they already read.

**Pros:**
- CLAUDE.md is already in every agent's context window at session start — no additional fetches
- Consistent structure across all agent repositories — an agent knows where to find any piece
  of information regardless of which agent repository it is working in
- The content requirements double as a completeness checklist for repository setup

**Cons:**
- Three files to keep in sync (global + repository + pattern)
- Agent-specific content (Knowledge Base ID, manifest path) is environment-specific — must
  not be environment-specific values (use relative paths, not absolute ARNs per ADR-016)

---

## Consequences

### Positive
- Agents working in any agent repository immediately have the manifest location, Knowledge
  Base ID, and MCP tools in context — no searching required
- Documentation gaps are detectable: a repository CLAUDE.md missing required fields fails
  the completeness check
- Pattern CLAUDE.md files encode the scaffolding sequence and quality gates for each agent
  type — reducing the chance of agents following an incorrect or incomplete setup sequence

### Trade-offs
- Repository CLAUDE.md files require updates when manifest paths change, Knowledge Bases are
  rotated, or new MCP tools are added — these are structural changes that should trigger a PR
  with documentation updates per ADR-015
- The global CLAUDE.md must be distributed to every engineer's workstation — its update
  requires a mechanism outside normal repository PRs (onboarding script or standards repository)

---

## Implementation Rules

> These rules apply to all engineers and Claude agents setting up or maintaining agent repositories.

**Level 1 — Global CLAUDE.md additions (beyond ADR-016):**

Add the following section to `~/.claude/CLAUDE.md`:

```markdown
## AI Platform Knowledge Base
Platform documentation is at: https://github.com/normjon/claude-foundation-best-practice

Documentation gap resolution: If you cannot find complete guidance for a decision
you need to make, stop. Surface the gap explicitly. Resolve it by reading the ADR
index and consulting the relevant domain folder. If the gap is not covered by any
existing ADR, draft a new ADR, open a PR, and request review before proceeding.
Never guess at platform decisions.
```

**Level 2 — Repository CLAUDE.md required sections for agent repositories:**

```markdown
## Agent Configuration

- **Manifest location:** `manifests/<agent-name>/manifest.yaml`
- **Knowledge Base ID:** Retrieved from SSM Parameter Store at `/platform/<env>/kb-id`
  (do not hardcode — varies by environment)
- **MCP tools available to this agent:**
  - `create-session` — start a new AgentCore session
  - `invoke-tool` — call a registered MCP tool
  - `query-knowledge-base` — retrieve context from the Knowledge Base
  (full tool schemas in `mcp-catalogue/`)
- **Data classification ceiling:** CONFIDENTIAL — do not log or store data above this
  classification in this agent's session state
- **Promotion pipeline:** `dev` → `test` → `staging` → `prod` via GitHub Environment
  approval gates (see ADR-013 amendment)

## Platform Documentation
Authoritative source: https://github.com/normjon/claude-foundation-best-practice
Read the relevant domain folder CLAUDE.md before making decisions in that domain.
```

**Level 3 — Pattern CLAUDE.md for agent types (one file per agent type):**

```markdown
# CLAUDE.md — <Agent Type Name>

## Agent Type
[One sentence describing what this agent type does]

## Scaffolding Sequence
[Exact steps to scaffold a new agent of this type — runnable commands, not descriptions]

## Quality Gate Thresholds
- Unit test coverage: >= 80%
- Integration test required: yes/no
- Schema validation: [tool name] schema must be declared in mcp-catalogue/ before deploying
- Performance threshold: p99 latency < [X]ms for [operation]

## Known Anti-Patterns
- [Anti-pattern 1]: [what it looks like, why it is wrong, what to do instead]
- [Anti-pattern 2]: ...

## Platform Documentation
Authoritative source: https://github.com/normjon/claude-foundation-best-practice
```

**DO:**
- Include all required Level 2 sections in every agent repository CLAUDE.md before the
  repository is used in any environment beyond sandbox
- Write Knowledge Base IDs as SSM Parameter Store paths, not literal IDs — IDs vary by
  environment and change when Knowledge Bases are rebuilt
- Update repository CLAUDE.md in the same PR as any change to the manifest location, MCP
  tool list, or promotion pipeline (per ADR-015)
- Create a Level 3 pattern CLAUDE.md for each agent type when the agent type is first
  documented in the platform knowledge base

**DO NOT:**
- Hardcode environment-specific ARNs, account IDs, or Knowledge Base IDs in CLAUDE.md —
  use SSM Parameter Store paths or relative references
- Duplicate ADR content in CLAUDE.md — link to the ADR instead
- Omit the "Platform Documentation" reference at the bottom of every CLAUDE.md level —
  this is how Claude agents know where to resolve documentation gaps

---

## Verification

```bash
# Confirm repository CLAUDE.md contains all required agent configuration fields
grep -E "Manifest location|Knowledge Base ID|MCP tools|Data classification|Promotion pipeline" CLAUDE.md
# Expected: all five fields present

# Confirm Knowledge Base ID is a parameter path, not a literal ID
grep "Knowledge Base ID" CLAUDE.md | grep -v "kb-[a-zA-Z0-9]\{10,\}"
# Expected: output line exists (confirms it's a path, not a hardcoded ID)

# Confirm platform documentation URL is referenced at every CLAUDE.md level
grep "claude-foundation-best-practice" CLAUDE.md
# Expected: URL present

# Confirm pattern CLAUDE.md exists for each agent type in use
for agent_type in mcp-catalogue/agent-types/*/; do
  agent_name=$(basename "$agent_type")
  if [ ! -f "patterns/${agent_name}/CLAUDE.md" ]; then
    echo "MISSING pattern CLAUDE.md: patterns/${agent_name}/CLAUDE.md"
  fi
done
# Expected: no output

# Confirm global CLAUDE.md has the gap resolution instruction
grep "Documentation gap resolution" ~/.claude/CLAUDE.md
# Expected: line present
```

---

## References

- Related: [ADR-016 — Claude Agent Integration via CLAUDE.md Hierarchy](./ADR-016-claude-agent-integration.md) — this ADR extends ADR-016
- Related: [ADR-015 — README.md as Living Documentation](../process/ADR-015-readme-living-documentation.md)
- Related: [ADR-018 — MCP Gateway Input Validation](../security/ADR-018-mcp-gateway-input-validation.md)
- [Claude Code CLAUDE.md documentation](https://docs.anthropic.com/en/docs/claude-code)
