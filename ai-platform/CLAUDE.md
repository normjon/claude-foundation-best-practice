# CLAUDE.md — AI Platform

## Purpose
This folder governs Claude agent integration, CLAUDE.md hierarchy standards, and MCP gateway configuration for AI workloads on this platform.

## Key Rules

1. **Organise Claude agent guidance in a three-level CLAUDE.md hierarchy: global → repository → pattern.** Each level must stay under 150 lines. Write instructions as imperative commands. Link to ADRs for detail. (ADR-016)

2. **Reference external pattern libraries by URL — never duplicate their content locally.** The authoritative libraries are: Claude Cookbooks (https://github.com/anthropics/claude-cookbooks), Anthropic on AWS (https://github.com/aws-samples/anthropic-on-aws), and EKS Blueprints (https://github.com/aws-ia/terraform-aws-eks-blueprints). (ADR-016, ADR-021)

3. **Every tool invocation request to the MCP Gateway must be validated against a declared JSON schema before any Lambda handler executes logic.** Requests that fail schema validation are rejected with a 400 response and logged to CloudWatch. Schema validation and ToolPolicy enforcement are both required — neither substitutes for the other. (ADR-018)

4. **Agent repositories must include in their repository-level CLAUDE.md: the agent manifest location, the target Knowledge Base ID, the MCP tools available to the agent, the data classification ceiling, and the promotion pipeline configuration.** (ADR-021)

5. **When Claude cannot find complete guidance in the documentation, stop, surface the gap, resolve it, and generate a PR before proceeding.** Never proceed by guessing at platform decisions that should be documented. (ADR-021)

6. **Apply ADR-001 (IRSA) for Claude API key storage.** Store API keys in Secrets Manager. Access via IRSA-scoped role. Never in environment variables or Helm values. (ADR-016)

7. **All agent layers must own four components inline: system prompt, guardrail, agent manifest, and Prompt Vault Lambda. IAM roles are created inline in the agent layer — never in a shared module or the platform layer.** (ADR-022)

8. **All AgentCore containers must use `us.anthropic.claude-sonnet-4-6` (not the bare model ID), start uvicorn with `--no-access-log --log-level warning`, cross-compile Python deps with `--python-platform aarch64-manylinux2014 --only-binary=:all:`, include `sessionId` in the payload body, and attach `BedrockAgentCoreFullAccess` to the runtime IAM role.** (ADR-023)

## ADR Index

| ADR | Title | One-line Summary |
|-----|-------|-----------------|
| [ADR-016](./ADR-016-claude-agent-integration.md) | Claude Agent Integration via CLAUDE.md Hierarchy | Three-level CLAUDE.md hierarchy encodes platform decisions for Claude agents |
| [ADR-018](../security/ADR-018-mcp-gateway-input-validation.md) | MCP Gateway Input Validation | JSON schema validation required for all MCP tool invocations before handler execution |
| [ADR-021](./ADR-021-ai-platform-claude-md-hierarchy.md) | AI Platform CLAUDE.md Hierarchy | AI-platform-specific content requirements at each level of the CLAUDE.md hierarchy |
| [ADR-022](./ADR-022-agent-layer-pattern.md) | Agent Layer Pattern | Four-component canonical structure for all agent layers with inline IAM ownership |
| [ADR-023](./ADR-023-agentcore-container-requirements.md) | AgentCore Container Requirements | Six non-obvious runtime rules derived from Phase 2 production failures |

## When to Read These ADRs

- Before writing any CLAUDE.md file for an agent repository → read ADR-016, ADR-021
- Before writing any MCP Gateway Lambda handler or tool schema → read ADR-018
- Before building any tool that handles MCP tool invocations → read ADR-018
- Before onboarding a new agent type to the platform → read ADR-021, ADR-022
- Before building or modifying an AgentCore container → read ADR-022, ADR-023
- When building Claude-powered features on this platform → read ADR-016 (external library references)

## Cross-References
These ADRs from other folders directly affect work in this domain:

| ADR | Folder | Rule Summary |
|-----|--------|--------------|
| ADR-015 | process/ | Update CLAUDE.md in the same PR as any change that affects agent behavior |
| ADR-001 | security/ | Use IRSA for all AWS credential delivery in agent infrastructure |
| ADR-003 | observability/ | Log all Claude API calls as structured JSON including model, token counts, and latency |
