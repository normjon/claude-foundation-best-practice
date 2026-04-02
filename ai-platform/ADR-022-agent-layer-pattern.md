# ADR-022: Agent Layer Pattern

**Status:** Accepted
**Date:** 2026-04-01
**Scope:** AI Platform + Infrastructure
**Tags:** ai-agents, agentcore, terraform, iam, agent-layer, module-boundaries, remote-state

---

## Context

The HR Assistant is the first production agent on the Enterprise AI Platform. Building it
established a canonical four-component structure for all agent layers: a system prompt stored
in Bedrock Prompt Management, a guardrail, an agent manifest registered in DynamoDB, and a
Prompt Vault Lambda. Without documenting this pattern, future agent teams will make
inconsistent choices about component boundaries, IAM ownership, and remote state wiring.
This inconsistency increases the cost of reviewing and supporting agent layers over time.

---

## Decision

All agent layers follow the four-component structure defined below. All four components are
required. IAM roles are owned inline by the agent layer (Option B). Platform dependencies
are consumed via `terraform_remote_state` from the platform layer — never from the foundation
layer directly.

---

## Options Considered

### Option 1: Platform layer owns shared agent infrastructure

**Description:** Define a generic agent template in the platform layer and parameterise it
per agent. Agent teams submit variable files; the platform team applies them.

**Pros:**
- Centralised control of agent infrastructure
- Consistent policy application across all agents

**Cons:**
- Platform team becomes a bottleneck for every agent deploy and update
- Agent teams cannot iterate independently — blast radius of platform changes extends to all
  agents simultaneously
- Agent-specific configuration (system prompt, guardrail topics, IAM scope) does not belong
  in the platform layer

---

### Option 2: Fully autonomous agent layers with no shared conventions ✓ *Rejected*

**Description:** Each agent team builds their layer however they see fit, using the platform
outputs as available.

**Pros:**
- Maximum team autonomy

**Cons:**
- No consistency across agent layers — cross-cutting changes (e.g., KMS key rotation,
  Prompt Vault schema updates) require per-team negotiation
- IAM roles may be defined in the wrong layer or in reusable modules, violating ADR-017
- Remote state wiring may bypass the platform layer and read foundation state directly,
  creating hidden cross-layer dependencies

---

### Option 3: Four-component agent layer pattern with inline IAM (Option B) ✓ *Selected*

**Description:** Each agent layer owns its four components and its IAM roles inline. Platform
outputs are consumed via remote state. Component boundaries are standardised; component
content is agent-specific.

**Pros:**
- Agent teams can deploy and iterate independently without gating the platform team
- IAM scope is visible in the agent layer that owns the protected resource — not buried in
  a shared module
- Consistent structure makes cross-cutting changes predictable: the same sections exist in
  every agent layer

**Cons:**
- Each new agent team must learn the four-component pattern — mitigated by this ADR and the
  agent CLAUDE.md template
- IAM role definitions are duplicated across agent layers rather than centralised — accepted
  because scope differences between agents make true reuse impractical

---

## Consequences

### Positive
- Agent teams have clear ownership boundaries: everything in `agents/<name>/` belongs to
  that team and can be applied without touching platform or foundation
- The four-component checklist is verifiable: a layer missing any component is incomplete
- Remote state wiring is unidirectional: agent layers read platform outputs; they never write
  to platform state or read foundation state directly

### Trade-offs
- The agent manifest DynamoDB item is not cleaned up by `terraform destroy` (see Known Gap
  below). Manual cleanup is required after destroy.
- A new agent type (e.g., one without a Knowledge Base) may not need all four components;
  document the exception in the agent CLAUDE.md rather than omitting without explanation.

---

## Implementation Rules

> These rules apply to all Claude agents and engineers building agent layers on this platform.

### The Four Required Components

**Component 1 — System Prompt**

Store the system prompt in Bedrock Prompt Management via `aws_bedrockagent_prompt`. Load the
prompt text at plan time using `file()` from a `prompts/` directory in the agent layer:

```hcl
resource "aws_bedrockagent_prompt" "system" {
  name        = "${var.project_name}-${var.agent_name}-system-prompt-${var.environment}"
  description = "System prompt for the ${var.agent_name} agent"

  variants = [{
    name          = "default"
    template_type = "TEXT"
    template_configuration = {
      text = {
        text = file("${path.module}/prompts/system-prompt.txt")
      }
    }
    model_id = var.model_id
  }]

  tags = var.tags
}
```

No variable substitution at plan time. Placeholder strings (e.g., `[COMPANY_NAME]`) are
replaced as part of environment promotion, not Terraform interpolation.

**Component 2 — Guardrail**

Define an `aws_bedrock_guardrail` with topic policies (DENY list), content filters, and PII
anonymization. Set the contextual grounding threshold per agent requirements. All guardrail
configuration lives in the agent layer — never in the platform layer:

```hcl
resource "aws_bedrock_guardrail" "agent" {
  name                      = "${var.project_name}-${var.agent_name}-guardrail-${var.environment}"
  blocked_input_messaging   = "I cannot respond to that request."
  blocked_outputs_messaging = "I cannot provide that information."

  topic_policy_config {
    topics_config {
      name       = "ExampleDeniedTopic"
      definition = "Describe the topic to deny here."
      examples   = ["Example prompt that triggers this topic"]
      type       = "DENY"
    }
  }

  content_policy_config {
    filters_config {
      type            = "HATE"
      input_strength  = "HIGH"
      output_strength = "HIGH"
    }
  }

  sensitive_information_policy_config {
    pii_entities_config {
      type   = "EMAIL"
      action = "ANONYMIZE"
    }
  }

  tags = var.tags
}
```

**Component 3 — Agent Manifest**

Register the agent manifest in the platform DynamoDB agent registry via
`terraform_data + local-exec` using `aws dynamodb put-item`. This is a workaround until a
native Terraform resource exists for AgentCore agent configuration:

```hcl
resource "terraform_data" "agent_manifest" {
  triggers_replace = [
    var.model_id,
    aws_bedrockagent_prompt.system.id,
    aws_bedrock_guardrail.agent.guardrail_id,
  ]

  provisioner "local-exec" {
    command = <<-EOT
      aws dynamodb put-item \
        --table-name "${local.agent_registry_table}" \
        --region "${var.aws_region}" \
        --item '{
          "agentId":       {"S": "${var.agent_name}"},
          "modelId":       {"S": "${var.model_id}"},
          "promptId":      {"S": "${aws_bedrockagent_prompt.system.id}"},
          "guardrailId":   {"S": "${aws_bedrock_guardrail.agent.guardrail_id}"},
          "environment":   {"S": "${var.environment}"},
          "updatedAt":     {"S": "${timestamp()}"}
        }'
    EOT
  }
}
```

**Known gap:** The `terraform_data` block has no `when = destroy` provisioner — the manifest
item persists in DynamoDB after `terraform destroy` and must be deleted manually. The
`put-item` call is idempotent so a stale item does not block re-apply. Document this in the
agent CLAUDE.md teardown section.

When `aws_bedrockagentcore_agent_configuration` or an equivalent native Terraform resource
becomes available in the AWS provider, replace the `local-exec` block.

**Component 4 — Prompt Vault Lambda**

Define an `aws_lambda_function` (arm64, python3.12) with an inline IAM execution role scoped
to the agent's S3 prefix only. Include a Lambda permission allowing AgentCore to invoke it:

```hcl
resource "aws_iam_role" "prompt_vault_lambda" {
  name = "${var.project_name}-${var.agent_name}-prompt-vault-lambda-${var.environment}"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "lambda.amazonaws.com" }
      Action    = "sts:AssumeRole"
    }]
  })

  tags = var.tags
}

resource "aws_iam_role_policy" "prompt_vault_lambda" {
  name = "prompt-vault-access"
  role = aws_iam_role.prompt_vault_lambda.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect   = "Allow"
        Action   = ["s3:PutObject", "s3:GetObject"]
        Resource = "arn:aws:s3:::${local.prompt_vault_bucket}/${var.agent_name}/*"
      },
      {
        Effect   = "Allow"
        Action   = ["kms:Decrypt", "kms:GenerateDataKey"]
        Resource = local.kms_key_arn
      },
      {
        Effect   = "Allow"
        Action   = ["logs:CreateLogGroup", "logs:CreateLogStream", "logs:PutLogEvents"]
        Resource = "arn:aws:logs:${var.aws_region}:${data.aws_caller_identity.current.account_id}:log-group:/aws/lambda/*"
      }
    ]
  })
}

resource "aws_lambda_function" "prompt_vault" {
  function_name = "${var.project_name}-${var.agent_name}-prompt-vault-${var.environment}"
  role          = aws_iam_role.prompt_vault_lambda.arn
  handler       = "handler.lambda_handler"
  runtime       = "python3.12"
  architectures = ["arm64"]
  filename      = data.archive_file.prompt_vault.output_path

  environment {
    variables = {
      PROMPT_VAULT_BUCKET = local.prompt_vault_bucket
      AGENT_PREFIX        = var.agent_name
    }
  }

  tags = var.tags
}

resource "aws_lambda_permission" "agentcore_invoke" {
  statement_id  = "AllowAgentCoreInvoke"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.prompt_vault.function_name
  principal     = "bedrock-agentcore.amazonaws.com"
  source_arn    = "arn:aws:bedrock-agentcore:${var.aws_region}:${data.aws_caller_identity.current.account_id}:runtime/${local.agentcore_endpoint_id}"
}
```

### IAM Ownership (Option B)

Each agent layer creates its own IAM roles inline in `main.tf`. Never define agent IAM roles
in the platform layer or in reusable modules. Required inline IAM roles per agent layer:

- **Prompt Vault Lambda execution role** — scoped to the agent's S3 prefix, KMS decrypt and
  GenerateDataKey, CloudWatch Logs
- **Knowledge Base service role** (if the agent has a KB) — scoped to S3 read, OpenSearch
  write, Bedrock embedding invocation, KMS decrypt

### Remote State Wiring

All platform dependencies are consumed via `data "terraform_remote_state" "platform"`.
The agent layer reads platform state but never writes to it.

Required platform outputs consumed by every agent layer:

```hcl
data "terraform_remote_state" "platform" {
  backend = "s3"
  config = {
    bucket = var.state_bucket
    key    = "dev/platform/terraform.tfstate"
    region = var.aws_region
  }
}

locals {
  agentcore_endpoint_id  = data.terraform_remote_state.platform.outputs.agentcore_endpoint_id
  agentcore_gateway_id   = data.terraform_remote_state.platform.outputs.agentcore_gateway_id
  agent_registry_table   = data.terraform_remote_state.platform.outputs.agent_registry_table
  prompt_vault_bucket    = data.terraform_remote_state.platform.outputs.prompt_vault_bucket
  kms_key_arn            = data.terraform_remote_state.platform.outputs.kms_key_arn
}
```

Never read foundation state directly from an agent layer — always go through platform.

### CLAUDE.md Hierarchy (ADR-021)

Each agent layer must have its own `CLAUDE.md` following the three-level hierarchy from
ADR-021. The agent CLAUDE.md must document:

- Pre-flight checklist (platform layer applied, container image built and pushed to ECR,
  DynamoDB manifest cleanup if re-applying after destroy)
- Container build rules (arm64 targeting, Python dependency cross-compilation per ADR-004,
  uvicorn startup flags per ADR-023)
- Known pitfalls specific to this agent (cross-reference platform README, not duplicate it)
- Invocation patterns (session ID handling, payload structure, CloudWatch log path)
- Teardown notes including the manual DynamoDB manifest item deletion step

The agent CLAUDE.md must NOT duplicate platform-layer configuration (VPC endpoints, IAM
permission lists, security group rules). Cross-reference `terraform/dev/platform/README.md`
instead.

**DO:**
- Own all four components in the agent layer
- Create IAM roles inline in `main.tf` (Option B)
- Consume platform outputs via `terraform_remote_state`; never read foundation state directly
- Document the local-exec manifest gap in the agent CLAUDE.md teardown section
- Manually delete the DynamoDB manifest item after `terraform destroy`:
  ```bash
  aws dynamodb delete-item \
    --table-name <agent_registry_table> \
    --key '{"agentId": {"S": "<agent_name>"}}' \
    --region us-east-2
  ```
- Set `architectures = ["arm64"]` on every `aws_lambda_function`

**DO NOT:**
- Define agent IAM roles in the platform layer or any reusable module
- Read foundation state directly from an agent layer
- Use `when = destroy` on `terraform_data` blocks without testing — provisioner reliability
  for cleanup is not guaranteed
- Duplicate platform-layer VPC or IAM documentation in the agent CLAUDE.md
- Interpolate environment-specific values into the system prompt at plan time — use
  placeholder strings replaced during environment promotion

---

## Verification

```bash
# Confirm all four components are present in the agent layer main.tf
grep -E "aws_bedrockagent_prompt|aws_bedrock_guardrail|terraform_data.*manifest|aws_lambda_function.*prompt_vault" main.tf
# Expected: all four resource types present

# Confirm IAM roles are defined inline in the agent layer, not imported from a module
grep -E "aws_iam_role\." main.tf | grep -v "module\."
# Expected: IAM role resources defined directly in main.tf

# Confirm platform remote state is the only remote state dependency
grep "terraform_remote_state" main.tf | grep -v "platform"
# Expected: no output (agent layer reads only platform state)

# Confirm Lambda architectures are arm64
grep -A5 "aws_lambda_function" main.tf | grep "arm64"
# Expected: arm64 present

# Confirm Lambda permission for AgentCore invocation exists
grep "aws_lambda_permission" main.tf | grep -i "agentcore"
# Expected: permission resource present

# Confirm no agent IAM roles defined in platform layer
grep "aws_iam_role" ../platform/main.tf | grep -i "agent"
# Expected: no output (agent roles must not be in platform layer)
```

---

## References

- Related: [ADR-021 — AI Platform CLAUDE.md Hierarchy](./ADR-021-ai-platform-claude-md-hierarchy.md)
- Related: [ADR-023 — AgentCore Container Requirements](./ADR-023-agentcore-container-requirements.md)
- Related: [ADR-017 — Infrastructure as Code Governance](../infrastructure/ADR-017-iac-governance.md)
- Related: [ADR-004 — arm64 (Graviton) as the Required Container Architecture](../infrastructure/ADR-004-arm64-graviton-architecture.md)
- Related: [ADR-001 — IRSA over Node Instance Profiles](../security/ADR-001-irsa-over-node-instance-profiles.md)
- Related: [ADR-015 — README.md as Living Documentation](../process/ADR-015-readme-living-documentation.md)
- [Amazon Bedrock Prompt Management documentation](https://docs.aws.amazon.com/bedrock/latest/userguide/prompt-management.html)
- [Amazon Bedrock Guardrails documentation](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails.html)
