# ADR-003: Structured JSON Logging to stdout

**Status:** Accepted
**Date:** 2026-03-01
**Scope:** Application + Infrastructure
**Tags:** observability, logging, cloudwatch, automation, containerization

---

## Context

EKS collects logs from container stdout/stderr and routes them to CloudWatch via one of two
mechanisms: a Fluent Bit sidecar (Fargate pods) or a CloudWatch Observability DaemonSet (EC2
pods). In both cases, log lines are captured verbatim and stored as CloudWatch log events.

CloudWatch Logs Insights can query log events using a SQL-like syntax — but only if the log
events contain structured fields. Unstructured text requires full-text search, which is slower,
less precise, and cannot aggregate or filter by severity.

Containerized applications also run in ephemeral environments where file-based logs are lost
on pod restart. The twelve-factor app methodology and Kubernetes best practices both mandate
that logs go to stdout/stderr.

---

## Decision

All application services running on this platform **must** emit logs as newline-delimited JSON
objects to stdout. Each log line must include at minimum: `level`, `message`, and `timestamp`.
No file-based logging. No `console.log` raw strings in production code.

---

## Options Considered

### Option 1: Unstructured text to stdout (`console.log`)

**Description:** Use `console.log`, `fmt.Println`, or equivalent for all log output. Log lines
are human-readable strings.

**Pros:**
- Zero setup — available in every language by default
- Easy to read during local development

**Cons:**
- CloudWatch Logs Insights cannot filter by severity — searching for errors requires full-text
  scan across all log levels
- Cannot aggregate error counts, group by endpoint, or correlate by request ID
- Log parsing regex patterns are brittle and language-specific
- Inconsistent format across services makes cross-service debugging harder

---

### Option 2: Structured logging via a framework (Winston, Pino, Zap, structlog)

**Description:** Use a structured logging library that outputs JSON by default and provides
level-based filtering, field injection, and transport configuration.

**Pros:**
- Full-featured: log levels, field injection, redaction, sampling, transports
- Some frameworks (Pino) offer significant performance benefits via async I/O

**Cons:**
- Adds a dependency with its own API, configuration surface, and upgrade cycle
- Teams must agree on framework choice per language/runtime
- Adds complexity for simple services that don't need the full feature set

---

### Option 3: Minimal structured JSON to stdout ✓ *Selected*

**Description:** Emit JSON log lines directly to stdout using the language's built-in
serialization. Every log entry is a JSON object with a required field schema. No external
logging framework required for simple services.

**Pros:**
- CloudWatch Logs Insights natively parses JSON log events — no log parsing config required
- Consistent field schema (`level`, `message`, `timestamp`) across all services regardless
  of language or framework
- Zero external dependencies for the logging mechanism itself
- Works identically in local development and EKS (stdout in both cases)
- Compatible with all log collectors (Fluent Bit sidecar, CloudWatch DaemonSet, local terminal)

**Cons:**
- Less ergonomic than a dedicated logging library for complex use cases (request tracing,
  distributed correlation IDs, log sampling)
- Stack traces must be serialized explicitly — they are not automatically included
- No built-in log level filtering at the application layer (all levels always emitted)

---

## Consequences

### Positive
- CloudWatch Logs Insights queries work immediately with no parser configuration:
  ```sql
  fields @timestamp, level, message
  | filter level = "error"
  | sort @timestamp desc
  | limit 20
  ```
- Consistent log shape across all services reduces cognitive overhead when debugging
- Logs are visible in local development without any infrastructure (just run the process)
- Fluent Bit and CloudWatch Observability agent require zero configuration to collect JSON logs

### Trade-offs
- Logs are less immediately human-readable without a JSON pretty-printer
  (mitigated: use `npm run dev | jq` or `kubectl logs <pod> | jq` locally)
- Error field should contain the error message string — do not include full stack traces
  in production logs to avoid leaking internal implementation details

---

## Implementation Rules

> These rules apply to all Claude agents and engineers writing application code for this platform.

**Required log fields:**

| Field | Type | Required | Notes |
|---|---|---|---|
| `level` | string | Always | `"info"`, `"warn"`, `"error"`, `"debug"` |
| `message` | string | Always | Human-readable description of the event |
| `timestamp` | string | Always | ISO 8601 format: `new Date().toISOString()` |
| `error` | string | On errors | Error message only — no stack traces |

**Example compliant log lines:**
```json
{"level":"info","message":"Server started on port 3000","timestamp":"2026-03-01T00:00:00.000Z"}
{"level":"error","message":"Failed to create user","error":"ConditionalCheckFailedException","timestamp":"2026-03-01T00:00:01.000Z"}
{"level":"warn","message":"Scan limit reached","count":100,"timestamp":"2026-03-01T00:00:02.000Z"}
```

**DO:**
- Use `JSON.stringify({level, message, timestamp, ...fields})` or equivalent for every log line
- Include the `error` field (message string) on all catch blocks that log
- Add context fields as additional JSON properties on the same object (`count`, `userId`, `path`)
- Emit to stdout (`console.error` is acceptable for error-level if the framework routes it
  to stderr — both are collected by Fluent Bit)
- Treat log output as an API: maintain backward-compatible field names when refactoring

**DO NOT:**
- Use `console.log("User created:", user)` — string concatenation produces unparseable output
- Log sensitive data: tokens, passwords, full JWT payloads, PII (email, name, phone)
- Include full stack traces in the `error` field — log the error message only
- Write to files or use file-based logging transports
- Use logging frameworks that require sidecar agents or additional infrastructure

---

## Infrastructure: Log Collection by Node Type

Log collection is automatic — no application code changes required — but the collection
mechanism differs by node type:

| Node Type | Collection Mechanism | Destination |
|---|---|---|
| Fargate | Built-in Fluent Bit sidecar (activated by `aws-observability` namespace + `aws-logging` ConfigMap) | `/aws/eks/<cluster>/fargate` |
| EC2 (Karpenter) | CloudWatch Observability EKS addon (DaemonSet) | `/aws/eks/<cluster>/application` |

Both destinations support CloudWatch Logs Insights queries. Both require the relevant IAM
permissions on the execution role or node role respectively.

**Pre-requisites (managed by Terraform, not by the application):**
- CloudWatch log groups pre-created with 30-day retention
- Fargate execution roles with `logs:CreateLogStream`, `logs:PutLogEvents` scoped to the log group
- Karpenter node IAM role with `CloudWatchAgentServerPolicy` attached

---

## Verification

```bash
# Local: confirm logs are valid JSON (run app then check output)
npm run dev 2>&1 | head -5 | jq .
# Expected: parsed JSON objects, no parse errors

# EKS: tail pod logs and validate structure
kubectl logs -l app.kubernetes.io/name=<app> --tail=10 | jq '{level, message, timestamp}'
# Expected: all three fields present on every line

# CloudWatch: confirm log group exists and has log events
aws logs describe-log-groups \
  --region <region> \
  --log-group-name-prefix "/aws/eks/<cluster>" \
  --query 'logGroups[*].{name: logGroupName, retention: retentionInDays}'

# CloudWatch Logs Insights: query for errors in the last hour
aws logs start-query \
  --region <region> \
  --log-group-name "/aws/eks/<cluster>/application" \
  --start-time $(date -d '-1 hour' +%s) \
  --end-time $(date +%s) \
  --query-string 'fields @timestamp, level, message | filter level = "error" | limit 20'
```

---

## References

- [CloudWatch Logs Insights query syntax](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CWL_QuerySyntax.html)
- [Twelve-Factor App: Logs](https://12factor.net/logs)
- [AWS Fargate logging with Fluent Bit](https://docs.aws.amazon.com/eks/latest/userguide/fargate-logging.html)
- [Amazon CloudWatch Observability EKS Add-on](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Container-Insights-setup-EKS-addon.html)
- Related: [ADR-001 — IRSA](./ADR-001-irsa-over-node-instance-profiles.md)
