# ADR-019: Container Image Traceability

**Status:** Accepted
**Date:** 2026-03-29
**Scope:** Application
**Tags:** deployment, docker, ecr, image, versioning, traceability, lambda, ecs, agents

---

## Context

The AI platform runs multiple categories of containerised components: Lambda container images
for MCP tool handlers, ECS services for long-running platform services, and containerised
federated agents. ADR-009 established git SHA tagging for application images deployed to EKS.
This ADR extends that decision to all platform components and adds requirements specific to
the platform: deployment record traceability and ECR lifecycle policy minimums.

A deployment record in the Agent Registry associates a running agent instance with the
manifest version that configured it. Without also recording the container image commit, an
incident in production cannot be traced to the exact code that ran — the manifest version
alone does not identify the Lambda or container code. Deployment records must carry both the
manifest commit and the container image commit to be traceable end to end.

---

## Decision

All container images built for platform components must be tagged with the short git SHA of
the commit that produced them. Images tagged `latest` must never be deployed to staging or
production. The container image tag must be recorded in the Agent Registry deployment record
alongside the manifest version. ECR lifecycle policies must retain a minimum of 30 image
versions per repository to support rollback within the standard rollback window.

---

## Options Considered

### Option 1: Semantic version tags for platform images

**Description:** Tag platform images with semantic version numbers (e.g., `v1.2.3`) on each
release. Lambda functions and ECS services reference the semver tag.

**Pros:**
- Human-readable version numbers in deployment records
- Consistent with how third-party software is versioned

**Cons:**
- Requires a manual release process to bump versions — CI pipelines that push on every commit
  must tag as `v1.2.3-<commit>`, which loses the simplicity of semver
- Does not map 1:1 to commits — a release may include many commits with no per-commit rollback
- Reusing a tag for a hotfix patch requires incrementing the version, adding ceremony

---

### Option 2: Git SHA tags, no deployment record requirement

**Description:** Tag images with git SHA per ADR-009. Record only the manifest version in
deployment records.

**Pros:**
- Implements ADR-009 uniformly across all image types
- No additional deployment record changes required

**Cons:**
- Deployment records do not identify which Lambda code ran — only which manifest was active
- An incident investigation that reveals a code bug cannot be cross-referenced to the
  deployment record without a separate manual lookup from the Lambda function version

---

### Option 3: Git SHA tags + deployment record traceability + ECR retention minimum ✓ *Selected*

**Description:** Extend ADR-009's git SHA tagging to all platform container images. Add a
requirement that the Agent Registry deployment record stores both the manifest commit SHA and
the container image commit SHA. Define an ECR lifecycle policy minimum of 30 retained image
versions per repository.

**Pros:**
- Any deployment record uniquely identifies the exact code that ran — both infrastructure
  configuration (manifest) and execution code (container image)
- 30-version ECR retention covers the standard rollback window (30 deployments) without
  manual intervention
- Consistent with ADR-009 — no new tagging mechanism to learn

**Cons:**
- Deployment record schema change required — must add `imageCommit` field to existing records
- ECR lifecycle policy creation is a Terraform change for each repository

---

## Consequences

### Positive
- Any production incident can be traced to the exact container image commit within the
  deployment record — no separate lookup required
- Rollback to any of the last 30 deployments is fully supported by ECR retention
- Uniform tagging policy across Lambda, ECS, and agent containers reduces cognitive overhead

### Trade-offs
- Deployment records written before this ADR was accepted do not have `imageCommit` fields —
  treat them as incomplete but do not backfill (the historical container versions are
  unverifiable)
- 30-version minimum may need to increase for repositories with high deployment frequency
  (e.g., multiple deploys per day) — review per-repository

---

## Implementation Rules

> These rules apply to all Claude agents and engineers building and deploying platform container images.

**Image tagging (extends ADR-009 to all platform image types):**

```bash
TAG=$(git rev-parse --short HEAD)

# Lambda container image
docker build --platform linux/arm64 \
  -t ${AWS_ACCOUNT}.dkr.ecr.us-east-2.amazonaws.com/<repo>:${TAG} .
docker push ${AWS_ACCOUNT}.dkr.ecr.us-east-2.amazonaws.com/<repo>:${TAG}

# Verify architecture before push
docker inspect ${IMAGE}:${TAG} --format '{{.Architecture}}'
# Must output: arm64
```

**Deployment record schema (Agent Registry):**

```json
{
  "deploymentId": "deploy-abc123",
  "agentId": "agent-xyz",
  "manifestVersion": "a1b2c3d",
  "imageTag": "f4e5d6c",
  "imageRepository": "<account>.dkr.ecr.us-east-2.amazonaws.com/my-agent",
  "environment": "production",
  "deployedAt": "2026-03-29T00:00:00.000Z",
  "deployedBy": "pipeline/deploy-workflow"
}
```

Both `manifestVersion` and `imageTag` are required fields. A deployment record with either
field absent is considered incomplete and must trigger a validation error.

**ECR lifecycle policy (Terraform):**

```hcl
resource "aws_ecr_lifecycle_policy" "platform_image" {
  repository = aws_ecr_repository.platform_image.name

  policy = jsonencode({
    rules = [
      {
        rulePriority = 1
        description  = "Retain last 30 images for rollback support"
        selection = {
          tagStatus   = "any"
          countType   = "imageCountMoreThan"
          countNumber = 30
        }
        action = { type = "expire" }
      }
    ]
  })
}
```

**DO:**
- Tag every container image with `$(git rev-parse --short HEAD)` in CI before push
- Record both `manifestVersion` and `imageTag` in every Agent Registry deployment record
- Apply an ECR lifecycle policy retaining at least 30 image versions to every platform ECR repository
- Verify `imagePullPolicy: Always` in Lambda function container configurations and ECS task definitions
- Build all platform images with `--platform linux/arm64` (per ADR-004)

**DO NOT:**
- Push to `latest`, `stable`, or any mutable tag — not to any platform repository
- Deploy to staging or production from a local `docker push` — all image pushes go through CI
- Create an ECR repository without an attached lifecycle policy
- Reduce ECR retention below 30 images without a documented justification
- Write deployment records that omit `imageTag`

---

## Verification

```bash
# Confirm all platform ECR repositories have lifecycle policies
aws ecr describe-lifecycle-policy \
  --repository-name <repo-name> \
  --region us-east-2 \
  --query 'lifecyclePolicyText' | jq '.rules[0].selection.countNumber'
# Expected: >= 30

# Confirm no 'latest' tagged images are deployed in production Lambda functions
aws lambda list-functions --region us-east-2 \
  --query 'Functions[?PackageType==`Image`].{Name:FunctionName,Image:Code.ImageUri}' \
  | jq '.[] | select(.Image | contains(":latest"))'
# Expected: no output (no latest-tagged images)

# Confirm deployed Lambda image tag resolves to a git commit
LAMBDA_IMAGE=$(aws lambda get-function \
  --function-name <function-name> \
  --region us-east-2 \
  --query 'Code.ImageUri' --output text)
IMAGE_TAG=$(echo $LAMBDA_IMAGE | cut -d: -f2)
git log --oneline | grep "^${IMAGE_TAG}"
# Expected: commit line for the deployed image

# Confirm deployment record contains both manifest version and image tag
aws dynamodb get-item \
  --table-name agent-registry-deployments \
  --key '{"deploymentId": {"S": "<deploy-id>"}}' \
  --region us-east-2 \
  | jq '.Item | {manifestVersion, imageTag}'
# Expected: both fields present and non-null
```

---

## References

- Related: [ADR-009 — Git SHA Image Tags](./ADR-009-git-sha-image-tags.md) — this ADR extends ADR-009 to all platform components
- Related: [ADR-004 — arm64 Architecture](../infrastructure/ADR-004-arm64-graviton-architecture.md)
- [Amazon ECR lifecycle policies](https://docs.aws.amazon.com/AmazonECR/latest/userguide/LifecyclePolicies.html)
- [AWS Lambda container images](https://docs.aws.amazon.com/lambda/latest/dg/images-create.html)
