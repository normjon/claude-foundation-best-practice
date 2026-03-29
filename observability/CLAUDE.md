# CLAUDE.md — Observability

## Purpose
This folder governs log format standards and log collection infrastructure for all workloads on this platform.

## Key Rules

1. **All application services must emit logs as newline-delimited JSON objects to stdout.** Every log line must include `level`, `message`, and `timestamp`. No file-based logging. No `console.log` raw strings in production code. (ADR-003)

2. **Never log sensitive data: tokens, passwords, full JWT payloads, or PII (email, name, phone).** Log the error message string in the `error` field — never full stack traces. (ADR-003)

3. **The cluster uses two separate log collection paths — one for Fargate, one for EC2.** Fargate pods use the built-in Fluent Bit sidecar via the `aws-observability` namespace. EC2 (Karpenter) pods use the `amazon-cloudwatch-observability` EKS addon DaemonSet. Do not assume a single log group contains all logs. (ADR-011)

4. **Pre-create both CloudWatch log groups in Terraform with `retention_in_days = 30`.** Set `auto_create_group false` in the Fargate Fluent Bit ConfigMap to prevent retention policy bypass. (ADR-011)

5. **When debugging, determine which compute type your pod is running on before querying logs.** System pods (Karpenter controller, LBC) → `/aws/eks/<cluster>/fargate`. Application pods → `/aws/eks/<cluster>/application`. (ADR-011)

6. **Treat log output as an API: maintain backward-compatible field names when refactoring.** CloudWatch Logs Insights queries depend on consistent field names across releases. (ADR-003)

## ADR Index

| ADR | Title | One-line Summary |
|-----|-------|-----------------|
| [ADR-003](./ADR-003-structured-json-logging.md) | Structured JSON Logging to stdout | All logs must be JSON with level, message, timestamp — no raw strings |
| [ADR-011](./ADR-011-dual-observability-paths.md) | Dual Observability Paths for Fargate and EC2 Node Types | Fargate uses Fluent Bit sidecar; EC2 uses CloudWatch DaemonSet — two log groups |

## When to Read These ADRs

- Before writing any logging code in any language → read ADR-003
- Before writing any `console.log`, `fmt.Println`, or equivalent → read ADR-003
- Before configuring the `aws-observability` namespace or `aws-logging` ConfigMap → read ADR-011
- Before attaching IAM policies to Fargate execution roles or Karpenter node roles → read ADR-011
- When debugging missing logs or querying CloudWatch Logs Insights → read ADR-011
