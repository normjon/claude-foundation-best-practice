# ADR-010: API Input Allowlisting and Database Scan Limits

**Status:** Accepted
**Date:** 2026-03-01
**Scope:** Application
**Tags:** security, api, dynamodb, validation, owasp, input-validation

---

## Context

REST APIs that accept query parameters or request bodies and pass them directly to a database
query are vulnerable to two related classes of problems:

**Attribute enumeration:** If the API accepts arbitrary query parameter names and maps them
to database filter expressions, an attacker can probe for hidden attributes
(e.g., `?isAdmin=true`, `?internalScore=100`) and infer the data model from error responses
or successful matches.

**Resource exhaustion via unbounded scans:** DynamoDB's `Scan` operation reads every item in
the table before applying filters. Without a result limit, a single API call can consume the
entire table's read capacity, causing throttling for all other operations. DynamoDB on-demand
billing (`PAY_PER_REQUEST`) does not cap scan costs — a scan against a large table incurs
full read costs for every item scanned.

---

## Decision

All API endpoints **must** maintain an explicit allowlist of permitted query parameters and
request body fields. No parameter outside the allowlist may be passed to a database operation.
All `Scan` operations **must** include a hard-coded maximum result limit. These constraints
must be validated in application code — not delegated to the database or a middleware layer.

---

## Options Considered

### Option 1: No input validation — pass parameters directly to DynamoDB

**Description:** Accept any query parameter and build a `FilterExpression` dynamically from
whatever the caller provides.

**Pros:**
- Zero boilerplate — flexible API surface with no code overhead

**Cons:**
- Exposes internal DynamoDB attribute names to callers
- Allows probing for undocumented attributes
- Unbounded scans consume unlimited read capacity
- Violates OWASP API Security Top 10: API3 (Broken Object Property Level Authorization)
  and API4 (Unrestricted Resource Consumption)

---

### Option 2: Schema validation library (Joi, Zod, JSON Schema)

**Description:** Define a schema for allowed inputs; validate incoming requests against the
schema before processing.

**Pros:**
- Expressive — type checking, format validation, and allowlisting in one place
- Standard approach in mature API frameworks

**Cons:**
- Adds a dependency that must be maintained and upgraded
- Schema drift: the schema and the allowlist can diverge if not kept in sync
- For simple CRUD APIs, the overhead of a full schema library is disproportionate

---

### Option 3: Explicit allowlist constants + hard-coded scan limit ✓ *Selected*

**Description:** Define the set of permitted filter attributes as a typed constant. Validate
every incoming query parameter against the constant before building the database query. Define
a separate constant for the maximum scan result count.

```typescript
const ALLOWED_FILTER_ATTRS = new Set(['userId', 'email', 'name']);
const MAX_SCAN_LIMIT = 100;

// Validate before touching DynamoDB
for (const key of Object.keys(queryParams)) {
  if (!ALLOWED_FILTER_ATTRS.has(key)) {
    return res.status(400).json({ error: `Unsupported query parameter: ${key}` });
  }
}
```

**Pros:**
- Zero additional dependencies
- The allowlist is the API contract — visible in code review, testable, auditable
- Changing the allowlist requires a code change (PR + review), not just a config change
- Error messages tell callers exactly what went wrong without revealing internal structure

**Cons:**
- Manual maintenance — adding a new filterable attribute requires updating the constant and
  adding a test
- Does not provide type coercion or format validation (email regex, UUID format) — those
  must be added separately for fields that need it

---

## Consequences

### Positive
- Attribute enumeration is structurally prevented — the allowlist is the only surface
- DynamoDB read costs are bounded — worst case is `MAX_SCAN_LIMIT` items per request
- Callers receive clear, actionable error messages for unsupported parameters
- The allowlist doubles as self-documenting API specification

### Trade-offs
- Adding a new filterable attribute requires a code change, not just a configuration change
  (this is a feature, not a bug — it forces a deliberate decision and PR review)
- `MAX_SCAN_LIMIT = 100` may be too low for some use cases — must be explicitly increased
  with justification, not implicitly removed

---

## Implementation Rules

> These rules apply to all Claude agents and engineers writing API handlers for this platform.

**DO:**
- Define filter attributes as a typed constant (Set or enum) at module scope:
  ```typescript
  const ALLOWED_FILTER_ATTRS = new Set<string>(['userId', 'email', 'name']);
  ```
- Define the scan limit as a named constant — never use a magic number inline:
  ```typescript
  const MAX_SCAN_LIMIT = 100;
  ```
- Validate all query parameters against the allowlist **before** any database call:
  ```typescript
  for (const [key, value] of Object.entries(req.query)) {
    if (!ALLOWED_FILTER_ATTRS.has(key)) {
      return res.status(400).json({ error: `Unsupported query parameter: ${key}` });
    }
    if (typeof value !== 'string') {
      return res.status(400).json({ error: `Query parameter "${key}" must be a single value` });
    }
  }
  ```
- Return HTTP 400 with a descriptive message for rejected parameters — do not return 403
  (which implies the parameter exists but is forbidden) or 500 (which hides validation errors)
- Add a test for each rejected parameter to confirm the 400 response

**DO NOT:**
- Pass `req.query` or `req.body` directly to `ScanCommand` `FilterExpression` builders
- Use `Limit: undefined` or omit `Limit` from `ScanCommand` — always pass `MAX_SCAN_LIMIT`
- Include internal field names in error messages (e.g., "field 'isAdmin' not found in schema")
  — generic "Unsupported query parameter: X" is sufficient
- Log the full query parameter set at INFO level — log parameter names but not values
  (values may contain PII)

**Tests to include for every filterable endpoint:**

```typescript
it('rejects unsupported query parameters with 400', async () => {
  const res = await request(app).get('/user-app/users?role=admin');
  expect(res.status).toBe(400);
  expect(res.body.error).toContain('Unsupported query parameter');
});

it('rejects duplicate query parameters with 400', async () => {
  const res = await request(app).get('/user-app/users?email=a@b.com&email=c@d.com');
  expect(res.status).toBe(400);
});

it('returns at most MAX_SCAN_LIMIT results', async () => {
  // mock DynamoDB to return 200 items; confirm response contains ≤ 100
});
```

---

## Verification

```bash
# Test allowlist enforcement (expect 400)
curl -s -o /dev/null -w "%{http_code}" \
  "http://${ALB}/<base-path>/users?role=admin"
# Expected: 400

# Test unsupported parameter returns error message (not 500)
curl -s "http://${ALB}/<base-path>/users?role=admin" | jq .error
# Expected: "Unsupported query parameter: role"

# Test scan limit (requires sufficient data in table)
curl -s "http://${ALB}/<base-path>/users" | jq '.users | length'
# Expected: ≤ 100

# Confirm no database calls occur before validation (check logs for ordering)
kubectl logs -l app.kubernetes.io/name=<app> --tail=20 | jq 'select(.message | contains("DynamoDB"))'
# Expected: no DynamoDB log entry when 400 is returned
```

---

## References

- [OWASP API Security Top 10](https://owasp.org/www-project-api-security/)
- [DynamoDB Scan pricing](https://aws.amazon.com/dynamodb/pricing/on-demand/)
- [DynamoDB best practices for queries](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/best-practices.html)
- Related: [ADR-003 — Structured JSON Logging](./ADR-003-structured-json-logging.md)
