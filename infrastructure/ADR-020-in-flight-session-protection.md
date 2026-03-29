# ADR-020: In-Flight Session Protection During Infrastructure Updates

**Status:** Accepted
**Date:** 2026-03-29
**Scope:** Infrastructure
**Tags:** agentcore, sessions, availability, infrastructure, lambda, api-gateway, dynamodb

---

## Context

AgentCore sessions can run for up to 8 hours. Platform infrastructure updates — Lambda
function deployments, API Gateway stage deployments, DynamoDB table changes — can silently
terminate in-flight sessions if applied while sessions are active. A Lambda deployment that
replaces a running function execution environment, a DynamoDB table update that causes a
brief unavailability window, or an API Gateway deployment that invalidates active connection
state can all interrupt a session that the agent or end user believes to be running.

Unlike a stateless web request, an interrupted AgentCore session cannot simply be retried —
the session state (conversation history, tool outputs, in-progress operations) may be
partially written or lost. Recovery requires the user to restart the session and re-establish
context.

---

## Decision

Infrastructure updates that affect components that active AgentCore sessions depend on must
not be applied while sessions are in flight. Before applying such updates, operators must
verify that the active session count is zero using the designated CloudWatch metric. If
sessions are in flight, the update must be deferred until they complete or are drained. A
documented emergency fast-path exists for changes that cannot wait.

---

## Options Considered

### Option 1: Apply infrastructure updates at any time — accept session disruption

**Description:** Schedule maintenance windows and communicate them to users. Apply updates
during low-traffic periods without checking for active sessions.

**Pros:**
- No operational overhead for pre-apply session checks
- Maintenance windows are a familiar operational pattern

**Cons:**
- AgentCore sessions of up to 8 hours make "low traffic" windows difficult to guarantee
- Users whose sessions are interrupted lose work without warning
- The blast radius of an infrastructure change extends to all in-flight sessions simultaneously
- Maintenance windows at 2am are not practical for a globally distributed platform

---

### Option 2: Rolling Lambda deployments with Lambda versioning and aliases

**Description:** Use Lambda function versions and weighted aliases to shift traffic gradually
from the old function version to the new one. In-flight invocations complete on the old version.

**Pros:**
- No downtime for in-flight requests at the Lambda invocation layer
- Rollback by adjusting alias weights

**Cons:**
- Lambda versioning handles individual invocations, not sessions spanning multiple invocations
- An AgentCore session that spans multiple Lambda invocations over 8 hours may have some
  invocations on the old version and some on the new — state schema compatibility must be
  guaranteed across versions
- Does not address API Gateway deployment changes or DynamoDB schema updates
- Adds version management overhead to every Lambda deployment

---

### Option 3: Session drain check before applying, with emergency fast-path ✓ *Selected*

**Description:** Before applying any infrastructure change to a component that active sessions
depend on, check the CloudWatch `ActiveSessionCount` metric. If the count is above zero,
defer the update until sessions drain or until the emergency fast-path is invoked. The
emergency fast-path is documented separately in the platform change control policy and
includes compensating controls (incident record, rollback plan, on-call notification).

**Pros:**
- No in-flight session is ever terminated by a routine infrastructure update
- The check is simple, automated, and can be integrated into the CI/CD pipeline as a
  pre-apply gate
- The emergency fast-path provides a documented escape valve for genuine emergencies without
  creating a loophole that bypasses the intent of the rule
- Applies uniformly to all dependent components (Lambda, API Gateway, DynamoDB)

**Cons:**
- A long-running session (8 hours) can block an urgent but non-emergency update for a
  significant period
- The `ActiveSessionCount` metric must be implemented and maintained — it is not an
  AWS-native metric; it must be emitted by the AgentCore session manager

---

## Consequences

### Positive
- Users' in-flight sessions are never interrupted by routine platform maintenance
- The pre-apply check is a forcing function for implementing the `ActiveSessionCount`
  CloudWatch metric — which also provides operational visibility into platform load
- Emergency changes have a documented path that satisfies audit requirements

### Trade-offs
- Platform teams must implement and maintain the `ActiveSessionCount` custom metric
- A session that is stuck (e.g., awaiting a user response indefinitely) can block updates
  until the session times out or is administratively terminated
- The emergency fast-path must not become the routine path — monitor its usage and alert if
  it is invoked more than once per quarter

---

## Implementation Rules

> These rules apply to all Claude agents and engineers applying infrastructure changes to
> components that AgentCore sessions depend on.

**Components that active sessions depend on (session-dependent infrastructure):**
- Lambda functions invoked by AgentCore (tool handlers, session manager)
- API Gateway stages that route agent traffic
- DynamoDB tables that store session state or agent registry data

**Pre-apply session check:**

```bash
# Retrieve the current active session count from CloudWatch
ACTIVE_SESSIONS=$(aws cloudwatch get-metric-statistics \
  --namespace "AgentCore/Sessions" \
  --metric-name "ActiveSessionCount" \
  --dimensions Name=Environment,Value=<env> \
  --start-time $(date -u -d '-2 minutes' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 120 \
  --statistics Maximum \
  --region us-east-2 \
  --query 'Datapoints[0].Maximum' \
  --output text)

if [ "$ACTIVE_SESSIONS" != "None" ] && [ "$(echo "$ACTIVE_SESSIONS > 0" | bc)" = "1" ]; then
  echo "ERROR: $ACTIVE_SESSIONS active session(s) in flight. Defer this apply until sessions drain."
  exit 1
fi
echo "OK: No active sessions. Proceeding with apply."
```

**CloudWatch alarm — gate for CI/CD pipeline:**

```hcl
resource "aws_cloudwatch_metric_alarm" "active_sessions_above_zero" {
  alarm_name          = "agentcore-sessions-active-${var.environment}"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "ActiveSessionCount"
  namespace           = "AgentCore/Sessions"
  period              = 60
  statistic           = "Maximum"
  threshold           = 0
  alarm_description   = "Active AgentCore sessions are in flight. Block infrastructure applies."
  dimensions = {
    Environment = var.environment
  }
}
```

**DO:**
- Run the pre-apply session check before every `terraform apply` to session-dependent infrastructure
- Integrate the `ActiveSessionCount` alarm check as a required pipeline step before `terraform apply`
  in staging and production
- Emit `ActiveSessionCount` as a CloudWatch metric from the AgentCore session manager on every
  session start and end
- Document the emergency fast-path invocation in the incident record when used

**DO NOT:**
- Apply changes to session-dependent Lambda functions, API Gateway stages, or DynamoDB tables
  while `ActiveSessionCount > 0` in a routine change
- Use the emergency fast-path for changes that can wait for session drain — it is for genuine
  emergencies only
- Delete or reduce the `ActiveSessionCount` metric emission — this metric is the only way to
  enforce this rule programmatically

**Emergency fast-path (when sessions cannot be drained):**

When an infrastructure change cannot wait for session drain (e.g., active security incident,
data corruption requiring immediate table update):

1. Open an incident record in the incident management system before applying
2. Notify the on-call engineer and the platform lead
3. Document the estimated impact: number of in-flight sessions that will be disrupted
4. Apply the change
5. Notify affected session owners if identifiable
6. Update the incident record with the actual impact and close

---

## Verification

```bash
# Confirm ActiveSessionCount metric is being emitted
aws cloudwatch list-metrics \
  --namespace "AgentCore/Sessions" \
  --metric-name "ActiveSessionCount" \
  --region us-east-2
# Expected: metric listed with at least one dimension set

# Confirm the CloudWatch alarm exists
aws cloudwatch describe-alarms \
  --alarm-names "agentcore-sessions-active-<env>" \
  --region us-east-2 \
  --query 'MetricAlarms[0].{Name:AlarmName,State:StateValue,Threshold:Threshold}'
# Expected: alarm exists, Threshold=0

# Confirm pre-apply check script is in the pipeline
grep -r "ActiveSessionCount" .github/workflows/
# Expected: at least one workflow references the session count check

# Verify no apply occurred while sessions were active (audit log check)
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=UpdateFunctionCode \
  --region us-east-2 \
  --start-time $(date -d '-7 days' -u +%Y-%m-%dT%H:%M:%SZ) \
  | jq '.Events[] | {time: .EventTime, user: .Username}'
# Cross-reference timestamps against CloudWatch ActiveSessionCount metric
# Expected: no UpdateFunctionCode events occur when ActiveSessionCount > 0
```

---

## References

- [Amazon CloudWatch custom metrics](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/publishingMetrics.html)
- [AWS Lambda function updates and in-flight invocations](https://docs.aws.amazon.com/lambda/latest/dg/lambda-traffic-shifting-using-aliases.html)
- Related: [ADR-005 — Staged Terraform Apply](./ADR-005-staged-terraform-apply.md)
- Related: [ADR-017 — Infrastructure as Code Governance](./ADR-017-iac-governance.md)
- Related: [ADR-014 — GitHub Actions Pipeline Automation](../process/ADR-014-github-actions-pipeline.md)
