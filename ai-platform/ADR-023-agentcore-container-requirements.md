# ADR-023: AgentCore Container Requirements

**Status:** Accepted
**Date:** 2026-04-01
**Scope:** AI Platform + Infrastructure
**Tags:** agentcore, container, arm64, graviton, bedrock, claude, uvicorn, session, iam, debugging

---

## Context

Building and debugging the HR Assistant AgentCore container revealed a set of non-obvious
runtime requirements. Each rule in this ADR corresponds to a production-blocking failure
during Phase 2 that required significant debugging time. None of these requirements are
clearly documented in AWS documentation as of October 2025. Without capturing them in an
ADR, future agent teams will rediscover the same failures independently.

---

## Decision

All AgentCore containers on this platform must follow the six rules defined below without
exception. The rules address: Claude 4.x model ID format, uvicorn startup flags, Python
dependency cross-compilation, session ID payload placement, required IAM managed policy,
and CloudWatch log group path for debugging.

---

## Options Considered

### Option 1: Document in each agent layer CLAUDE.md only

**Description:** Each agent team documents the runtime requirements in their own agent
CLAUDE.md. No platform-level ADR.

**Pros:**
- Keeps agent-specific context close to the agent code

**Cons:**
- Requirements are not platform-wide — future teams may miss them when creating a new agent
- No single authoritative source; documentation diverges across agent layers over time
- The failures this ADR prevents are platform-level failures (IAM policy, VPC routing,
  model ID format) not agent-level failures — they belong at the platform ADR level

---

### Option 2: Enforce via Terraform module validation only

**Description:** Encode the requirements as Terraform `precondition` blocks or variable
validation in the agentcore module.

**Pros:**
- Machine-enforced at plan time — violations fail early

**Cons:**
- Many of the requirements (uvicorn flags, Python cross-compilation, session ID payload
  structure) are container and application concerns, not Terraform concerns — they cannot
  be validated at plan time
- Terraform validation does not help when debugging a container that starts but returns 502

---

### Option 3: Platform ADR covering all AgentCore container requirements ✓ *Selected*

**Description:** Document all six requirements in a single ADR. Reference this ADR from
the project CLAUDE.md, the platform README, and every agent layer CLAUDE.md.

**Pros:**
- Single authoritative source — future agent teams read one document to understand all
  non-obvious requirements
- Requirements are traceable: each rule links to the failure it prevents
- The ADR can be updated as AWS documentation improves or new failure modes are discovered

**Cons:**
- Requires each agent team to read the ADR — mitigated by referencing it in the project
  CLAUDE.md and agent layer template

---

## Consequences

### Positive
- Agent teams building on this platform start with the complete set of known non-obvious
  requirements rather than discovering them through production failures
- Debugging is faster: the CloudWatch log path and session ID rules eliminate two common
  investigation dead ends
- IAM policy requirement is documented: the `BedrockAgentCoreFullAccess` policy absence is
  silent and its requirement is not mentioned in AWS AgentCore documentation

### Trade-offs
- This ADR will need updates as AgentCore matures and AWS addresses documentation gaps.
  Mark rules as superseded rather than deleting them so the history of the failure is preserved.

---

## Implementation Rules

> These rules apply to all Claude agents and engineers building AgentCore containers on this platform.

### Rule 1 — Cross-region inference profile prefix for Claude 4.x

Claude 4.x models (`claude-sonnet-4-6`, `claude-opus-4-6`) require the `us.*` cross-region
inference profile prefix for on-demand throughput. The bare model ID fails with:

```
Invocation of model ID anthropic.claude-sonnet-4-6 with on-demand throughput isn't supported
```

Use `us.anthropic.claude-sonnet-4-6` everywhere: `variables.tf` defaults, DynamoDB manifest
items, and application code:

```hcl
# variables.tf
variable "model_id" {
  description = "Bedrock model ID for the agent"
  type        = string
  default     = "us.anthropic.claude-sonnet-4-6"
}
```

```python
# application code
MODEL_ID = os.environ.get("MODEL_ID", "us.anthropic.claude-sonnet-4-6")
```

The IAM `bedrock:InvokeModel` resource list must include BOTH ARN forms. Without
`inference-profile/*`, the runtime receives `AccessDeniedException` even when `InvokeModel`
is otherwise allowed:

```hcl
resource "aws_iam_role_policy" "bedrock_invoke" {
  policy = jsonencode({
    Statement = [{
      Effect = "Allow"
      Action = "bedrock:InvokeModel"
      Resource = [
        "arn:aws:bedrock:${var.aws_region}:${data.aws_caller_identity.current.account_id}:inference-profile/*",
        "arn:aws:bedrock:*::foundation-model/*"
      ]
    }]
  })
}
```

### Rule 2 — uvicorn startup flags

Do NOT pass `--log-config /dev/null` to uvicorn. uvicorn 0.32.1 calls
`logging.config.fileConfig()` on the provided path. `/dev/null` is an empty file;
`fileConfig()` raises `RuntimeError: /dev/null is an empty file`. The container crashes at
startup with no useful log output.

Use instead:

```dockerfile
CMD ["uvicorn", "app.main:app", \
     "--host", "0.0.0.0", \
     "--port", "8080", \
     "--no-access-log", \
     "--log-level", "warning"]
```

### Rule 3 — Python dependency cross-compilation

All Python dependencies must be built with explicit arm64 targeting. Without
`--only-binary=:all:`, packages with C extensions build for the host architecture (x86) and
silently crash at runtime on Graviton. This failure is invisible during `docker build` and
only surfaces on invocation:

```bash
uv pip install \
  --python-platform aarch64-manylinux2014 \
  --python-version "3.12" \
  --target="${BUILD_DIR}" \
  --only-binary=:all: \
  -r requirements.txt
```

Never omit `--only-binary=:all:`. Never build Python dependencies on an x86 machine for
arm64 runtimes without this flag. See ADR-004 for the full ARM64/Graviton requirement.

### Rule 4 — Session ID must be in the payload body

`--runtime-session-id` (the AgentCore CLI flag) is used by the control plane for routing and
billing tracking. It is NOT forwarded to the container as the
`X-Amzn-Bedrock-AgentCore-Session-Id` header. The container never sees it.

Always include `sessionId` in the JSON payload body on every invocation:

```python
import json, base64

session_id = str(uuid.uuid4())
payload = json.dumps({
    "prompt": user_prompt,
    "sessionId": session_id
}).encode()

response = agentcore_client.invoke_agent_runtime(
    agentRuntimeId=runtime_id,
    runtimeSessionId=session_id,   # control-plane routing only
    payload=base64.b64encode(payload).decode()
)
```

Reusing a session ID with corrupt history (incomplete `tool_use`/`tool_result` pairs from a
failed invocation) causes:

```
Expected toolResult blocks at messages.X.content
```

Use a fresh session ID (`uuid.uuid4()`) when this error occurs. Do not attempt to resume
the corrupt session.

### Rule 5 — BedrockAgentCoreFullAccess managed policy is required

The `agentcore_runtime` IAM role must have the `BedrockAgentCoreFullAccess` managed policy
attached. Without it, the container cannot obtain a workload identity token. The failure is
silent: the invocation returns HTTP 502 with no entries in CloudWatch logs:

```hcl
resource "aws_iam_role_policy_attachment" "agentcore_full_access" {
  role       = aws_iam_role.agentcore_runtime.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonBedrockAgentCoreFullAccess"
}
```

This policy is required in addition to any inline policies scoped to specific Bedrock
resources. It is not a substitute for IRSA-scoped policies (ADR-001).

### Rule 6 — CloudWatch log group path

The correct log group path for AgentCore runtime containers is:

```
/aws/bedrock-agentcore/runtimes/<runtime-id>-DEFAULT
```

NOT `/aws/agentcore/<name>` — that path does not exist. Always check this log group first
when debugging 502 errors:

```bash
# Retrieve recent log events for the runtime
LOG_GROUP="/aws/bedrock-agentcore/runtimes/${RUNTIME_ID}-DEFAULT"

aws logs describe-log-streams \
  --log-group-name "${LOG_GROUP}" \
  --order-by LastEventTime \
  --descending \
  --max-items 1 \
  --region us-east-2 \
  --query 'logStreams[0].logStreamName' \
  --output text | xargs -I{} \
aws logs get-log-events \
  --log-group-name "${LOG_GROUP}" \
  --log-stream-name "{}" \
  --region us-east-2 \
  --query 'events[*].message' \
  --output text
```

If no logs appear after an invocation, the container failed to start. Check VPC endpoints
and security group rules in the platform layer — see the AgentCore Runtime Operational
Requirements section in the project `CLAUDE.md`.

---

**DO:**
- Use `us.anthropic.claude-sonnet-4-6` (not the bare model ID) for all Claude 4.x references
  in Terraform variables, DynamoDB manifest items, and application code
- Include both `inference-profile/*` and `foundation-model/*` in the `bedrock:InvokeModel`
  IAM resource list
- Start uvicorn with `--no-access-log --log-level warning` (not `--log-config /dev/null`)
- Cross-compile Python deps with `--python-platform aarch64-manylinux2014 --only-binary=:all:`
- Include `sessionId` in the payload JSON body on every invocation
- Use a fresh `uuid.uuid4()` session ID for every independent conversation
- Attach `BedrockAgentCoreFullAccess` to the `agentcore_runtime` IAM role
- Check `/aws/bedrock-agentcore/runtimes/<runtime-id>-DEFAULT` first when debugging 502 errors

**DO NOT:**
- Use bare model IDs for Claude 4.x (e.g., `anthropic.claude-sonnet-4-6`)
- Pass `--log-config /dev/null` to uvicorn
- Build Python dependencies on x86 without `--python-platform aarch64-manylinux2014 --only-binary=:all:`
- Rely on `--runtime-session-id` for session isolation inside the container — it is not
  forwarded to the container
- Attempt to resume a session that returned `Expected toolResult blocks` — use a fresh session ID
- Omit `BedrockAgentCoreFullAccess` from the runtime IAM role and expect useful error output

---

## Verification

```bash
# Rule 1: Confirm us.* prefix is used in variables.tf default
grep "model_id" variables.tf | grep "us\."
# Expected: us.anthropic.claude-sonnet-4-6

# Rule 1: Confirm inference-profile/* is in the bedrock:InvokeModel resource list
grep "inference-profile" main.tf
# Expected: arn:aws:bedrock:...:inference-profile/* present

# Rule 2: Confirm uvicorn is not started with --log-config /dev/null
grep "log-config" Dockerfile 2>/dev/null || echo "OK: --log-config not present"
# Expected: OK: --log-config not present

# Rule 3: Confirm cross-compilation flags in build script
grep "aarch64-manylinux2014" scripts/build.sh
# Expected: flag present in dependency install command

grep "only-binary" scripts/build.sh
# Expected: --only-binary=:all: present

# Rule 4: Confirm sessionId is included in payload construction
grep "sessionId" src/*.py
# Expected: sessionId key present in payload dict

# Rule 5: Confirm BedrockAgentCoreFullAccess is attached to runtime role
grep "BedrockAgentCoreFullAccess" main.tf
# Expected: policy ARN referenced in aws_iam_role_policy_attachment

# Rule 6: Confirm CloudWatch log group name uses correct path
aws logs describe-log-groups \
  --log-group-name-prefix "/aws/bedrock-agentcore/runtimes/" \
  --region us-east-2 \
  --query 'logGroups[*].logGroupName'
# Expected: one or more log groups with runtime ID suffix -DEFAULT
```

---

## References

- Related: [ADR-022 — Agent Layer Pattern](./ADR-022-agent-layer-pattern.md)
- Related: [ADR-004 — arm64 (Graviton) as the Required Container Architecture](../infrastructure/ADR-004-arm64-graviton-architecture.md)
- Related: [ADR-001 — IRSA over Node Instance Profiles](../security/ADR-001-irsa-over-node-instance-profiles.md)
- Related: [ADR-003 — Structured JSON Logging to stdout](../observability/ADR-003-structured-json-logging.md)
- [Amazon Bedrock AgentCore documentation](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/what-is-agentcore.html)
- [AgentCore Terraform samples — basic runtime](https://github.com/awslabs/amazon-bedrock-agentcore-samples/tree/main/04-infrastructure-as-code/terraform/basic-runtime)
- [Anthropic Claude cross-region inference profiles](https://docs.aws.amazon.com/bedrock/latest/userguide/inference-profiles-support.html)
