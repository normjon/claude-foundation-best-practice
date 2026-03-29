# ADR-018: MCP Gateway Input Validation

**Status:** Accepted
**Date:** 2026-03-29
**Scope:** Application + Infrastructure
**Tags:** security, mcp, validation, lambda, api-gateway, input-validation, json-schema

---

## Context

The MCP Gateway receives tool invocation requests from Claude agents and routes them to
Lambda handlers. Each request specifies a tool name and a payload of tool inputs. Without
input validation enforced at the Gateway layer, Lambda handlers receive arbitrary payloads —
creating a surface for injection attacks, type confusion errors, and unexpected Lambda
behaviour. The handler code cannot be assumed to perform its own schema validation uniformly.

A separate but related control is the ToolPolicy — a per-caller access control list that
governs which tools a caller may invoke. Input validation and ToolPolicy are complementary
controls: ToolPolicy governs *what* a caller may do; input validation governs *what inputs*
those tools will accept. Neither substitutes for the other.

Rate limiting at the API Gateway layer is a third distinct control that limits request volume.
Input validation does not overlap with rate limiting — both must be configured independently.

---

## Decision

Every tool invocation request received by the MCP Gateway must be validated against the
declared JSON schema for that tool before the Lambda handler executes any logic. The schema
for each tool is declared in the MCP tool catalogue and is the authoritative definition.
Requests that fail schema validation are rejected with HTTP 400 and logged to CloudWatch.
They are never forwarded to the Lambda handler.

---

## Options Considered

### Option 1: No Gateway-level validation — validate in each Lambda handler

**Description:** Each Lambda handler performs its own input validation. The Gateway passes
all requests through without inspection.

**Pros:**
- Handler authors have full control over validation logic
- No shared validation infrastructure to maintain

**Cons:**
- Validation coverage depends on every handler author implementing it consistently — gaps
  are not detected until a handler is called with invalid input
- Malformed requests reach Lambda invocations, consuming Lambda concurrency before rejection
- No centralised audit log of rejected inputs — each handler's logging may differ
- Adding a new tool requires the handler author to implement validation rather than inheriting it

---

### Option 2: JSON schema validation in the Gateway, schema defined per handler

**Description:** Each Lambda handler declares its own input schema (e.g., in a sidecar JSON
file or environment variable). The Gateway fetches and caches the schema at cold-start.

**Pros:**
- Schema is co-located with handler code — easier for handler authors to maintain
- Gateway enforces validation without handler code changes

**Cons:**
- Schema distribution is complex — Gateway must discover schemas from multiple handlers
- Schema versioning is decoupled from the tool catalogue, creating drift risk
- Handler updates that change the schema require coordinating two deployments

---

### Option 3: JSON schema declared in the MCP tool catalogue, enforced by Gateway ✓ *Selected*

**Description:** The MCP tool catalogue is the single source of truth for tool definitions,
including the JSON schema for each tool's inputs. The Gateway validates every incoming
request against the catalogue schema before invoking the Lambda handler. Requests that fail
validation are rejected with HTTP 400 and a structured error response; a CloudWatch log event
is emitted for every rejection.

**Pros:**
- Schema and tool definition are co-located in the catalogue — a single PR updates both
- The Gateway enforces validation uniformly across all tools without handler-specific code
- Rejected requests never consume Lambda concurrency
- All validation failures are logged centrally to CloudWatch for audit and alerting
- Catalogue-driven schema enables tooling: documentation generation, client SDK generation,
  integration testing against the schema

**Cons:**
- Catalogue updates require a deployment before new schema constraints take effect
- Handler authors must express validation requirements in JSON Schema (not code), which may
  be less expressive for complex conditional validation

---

## Consequences

### Positive
- No malformed request reaches any Lambda handler — handler code can assume a valid, typed
  payload conforming to the declared schema
- A single CloudWatch log stream captures all validation failures across all tools — can be
  alarmed centrally
- The tool catalogue schema is the contract between Claude agents (callers) and tool
  implementations (handlers) — both sides can rely on it

### Trade-offs
- The MCP tool catalogue becomes a deployment dependency — adding a new field to a tool's
  input schema requires a catalogue update and Gateway redeployment before clients can use
  the new field
- JSON Schema cannot express all business-level constraints (e.g., "field A is required when
  field B has value X with complex conditions") — such validation must still occur in the handler;
  the Gateway schema handles structural and type-level validation only

---

## Implementation Rules

> These rules apply to all Claude agents and engineers writing MCP tool handlers or catalogue entries.

**Tool catalogue schema declaration:**

```json
{
  "toolId": "create-agent",
  "description": "Provision a new agent in the registry",
  "inputSchema": {
    "$schema": "http://json-schema.org/draft-07/schema#",
    "type": "object",
    "required": ["agentName", "manifestVersion", "knowledgeBaseId"],
    "additionalProperties": false,
    "properties": {
      "agentName": {
        "type": "string",
        "minLength": 1,
        "maxLength": 64,
        "pattern": "^[a-z][a-z0-9-]*$"
      },
      "manifestVersion": {
        "type": "string",
        "pattern": "^[0-9a-f]{7,40}$"
      },
      "knowledgeBaseId": {
        "type": "string",
        "minLength": 10
      }
    }
  }
}
```

**Gateway validation response for rejected requests:**

```json
{
  "statusCode": 400,
  "body": {
    "error": "InputValidationError",
    "message": "Request payload failed schema validation",
    "details": [
      {
        "field": "/agentName",
        "constraint": "pattern",
        "value": "MyAgent",
        "message": "must match pattern ^[a-z][a-z0-9-]*$"
      }
    ]
  }
}
```

**CloudWatch log event for every rejected request:**

```json
{
  "level": "warn",
  "message": "Tool invocation rejected: schema validation failure",
  "timestamp": "2026-03-29T00:00:00.000Z",
  "toolId": "create-agent",
  "callerId": "agent-session-abc123",
  "validationErrors": ["/agentName: pattern mismatch"]
}
```

**DO:**
- Declare `"additionalProperties": false` in every tool schema — reject unknown fields
  rather than silently ignoring them
- Use `"required"` to enumerate all mandatory fields explicitly
- Log every validation failure to CloudWatch with `level: "warn"` and include `toolId`
  and `callerId` for correlation
- Add a CloudWatch alarm on validation failure rate — a spike indicates a client bug or
  an attempted injection
- Test the schema in the catalogue using `ajv validate` or equivalent before deploying

**DO NOT:**
- Allow Lambda handlers to be invoked with a payload that has not passed Gateway schema validation
- Conflate schema validation (structural) with ToolPolicy enforcement (access control) —
  both checks are required; ToolPolicy is checked before schema validation in the Gateway
  request pipeline
- Conflate schema validation with rate limiting — these are independent controls on separate
  layers; configuring one does not satisfy the other
- Include business logic in JSON Schema — use the handler for conditional validation
- Return 500 for schema validation failures — use 400 (client error)

---

## Verification

```bash
# Test valid request passes Gateway (expect handler response)
curl -s -X POST https://<gateway-url>/tools/create-agent \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"agentName":"my-agent","manifestVersion":"a1b2c3d","knowledgeBaseId":"kb-12345678901"}' \
  | jq .statusCode
# Expected: 200 (or handler-specific success code)

# Test invalid request rejected at Gateway (expect 400, never reaches Lambda)
curl -s -X POST https://<gateway-url>/tools/create-agent \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"agentName":"MyAgent"}' \
  | jq '{statusCode, error: .body.error}'
# Expected: statusCode 400, error "InputValidationError"

# Test additional properties rejected (expect 400)
curl -s -X POST https://<gateway-url>/tools/create-agent \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"agentName":"my-agent","manifestVersion":"a1b2c3d","knowledgeBaseId":"kb-12345678901","extra":"field"}' \
  | jq .statusCode
# Expected: 400

# Confirm validation failures appear in CloudWatch
aws logs filter-log-events \
  --region us-east-2 \
  --log-group-name "/aws/mcp-gateway/validation" \
  --filter-pattern "InputValidationError" \
  --limit 5
# Expected: log events for each failed request

# Confirm Lambda was not invoked for rejected requests
aws cloudwatch get-metric-statistics \
  --namespace AWS/Lambda \
  --metric-name Invocations \
  --dimensions Name=FunctionName,Value=<handler-function-name> \
  --start-time $(date -d '-5 minutes' -u +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 300 \
  --statistics Sum
# Expected: invocation count does not increase when 400 responses are returned
```

---

## References

- [JSON Schema specification](https://json-schema.org/specification)
- [AWS API Gateway request validation](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-method-request-validation.html)
- [OWASP API Security Top 10 — API3: Broken Object Property Level Authorization](https://owasp.org/www-project-api-security/)
- Related: [ADR-010 — API Input Allowlisting](./ADR-010-api-input-allowlisting.md)
- Related: [ADR-003 — Structured JSON Logging](../observability/ADR-003-structured-json-logging.md)
- Related: [ADR-001 — IRSA](./ADR-001-irsa-over-node-instance-profiles.md)
