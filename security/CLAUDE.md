# CLAUDE.md — Security

## Purpose
This folder governs IAM credential delivery, API input validation, and IAM policy management for all workloads on this platform.

## Key Rules

1. **Use IRSA for every workload — never node instance profiles or environment variable credentials.** Create one IAM role per Kubernetes ServiceAccount. Scope trust policies to the exact ServiceAccount name (singular form). (ADR-001)

2. **Never store `AWS_ACCESS_KEY_ID` or `AWS_SECRET_ACCESS_KEY` anywhere** — not in Kubernetes Secrets, ConfigMaps, Helm values, Dockerfiles, or environment variables. (ADR-001)

3. **Validate all API query parameters and request body fields against an explicit allowlist before any database call.** Return HTTP 400 for parameters outside the allowlist. Never pass `req.query` or `req.body` directly to a database operation. (ADR-010)

4. **All `Scan` operations must include a hard-coded `MAX_SCAN_LIMIT`.** Never omit `Limit` from `ScanCommand`. Default limit is 100. Increasing it requires explicit justification. (ADR-010)

5. **When an upstream IAM policy has a documented gap, apply a targeted inline policy patch — not a wildcard policy.** Document the missing permission, the controller version, and the expected upstream fix version in a code comment. (ADR-012)

6. **Scope all IAM policies to specific resource ARNs.** Never use `"Resource": "*"` for data-plane services (DynamoDB, S3, SQS). (ADR-001, ADR-012)

## ADR Index

| ADR | Title | One-line Summary |
|-----|-------|-----------------|
| [ADR-001](./ADR-001-irsa-over-node-instance-profiles.md) | IRSA over Node Instance Profiles | Use IRSA for all AWS credential delivery — no node profiles, no env vars |
| [ADR-010](./ADR-010-api-input-allowlisting.md) | API Input Allowlisting and Database Scan Limits | Allowlist API parameters; cap all DynamoDB Scans at MAX_SCAN_LIMIT |
| [ADR-012](./ADR-012-iam-policy-gap-remediation.md) | Upstream IAM Policy Gap Remediation Pattern | Patch upstream IAM gaps with minimal inline policies, never wildcards |

## When to Read These ADRs

- Before writing any code that calls AWS services from a pod → read ADR-001
- Before writing any API handler that queries a database → read ADR-010
- Before deploying or upgrading any third-party Kubernetes controller (LBC, ExternalDNS, etc.) → read ADR-012
- Before writing any IAM policy or trust policy → read ADR-001 and ADR-012
- When a controller is logging `AccessDenied` or permission errors → read ADR-012
