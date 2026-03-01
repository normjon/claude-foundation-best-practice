# ADR-009: Git SHA Image Tags for Container Images

**Status:** Accepted
**Date:** 2026-03-01
**Scope:** Application
**Tags:** deployment, docker, ecr, image, versioning, automation, karpenter

---

## Context

Container image tags are mutable labels — the same tag can point to a different image digest
after a push. When Karpenter provisions a new EC2 node, that node's kubelet pulls the
container image fresh from ECR. If the running pod and the newly-pulled image have the same
tag but different digests, pods on different nodes run different code. This silent divergence
is particularly dangerous with mutable tags like `latest` because it is invisible in
`kubectl get pods` and only becomes apparent during debugging or when behavior differs between
instances.

In EKS with Karpenter, nodes are created and destroyed dynamically. A re-push to ECR under
the same tag affects every future pod start, including:
- New pods during rolling updates
- Pods rescheduled after spot instance interruption
- Pods on newly consolidated nodes

---

## Decision

All container images deployed to this platform **must** be tagged with the short git commit
SHA (`git rev-parse --short HEAD`). Tags must never be reused. `imagePullPolicy` must be set
to `Always` to guarantee the declared tag is always fetched from ECR rather than served from
a node-local cache.

---

## Options Considered

### Option 1: `latest` tag

**Description:** Always build and push to the `latest` tag. Pods pull `latest` on each
restart.

**Pros:**
- Zero tag management — no need to track versions

**Cons:**
- Non-deterministic: pods on different nodes may run different code if `latest` was
  re-pushed between node starts
- `kubectl rollout undo` cannot work — there is no previous tag to roll back to
- Impossible to correlate a running pod with a specific code commit
- Cannot audit which commit is running in production from cluster state alone

---

### Option 2: Semantic versioning (e.g., `v1.2.3`)

**Description:** Tag images with semantic version numbers following a release process.

**Pros:**
- Human-readable version numbers
- Standard convention for published software
- Rollback to a specific version is explicit

**Cons:**
- Requires a manual or semi-automated release process to bump versions
- In active development, engineers frequently push without cutting a release — gaps between
  code changes and version tags lead to `v1.2.3-dirty` or multiple pushes to the same tag
- Does not map 1:1 to commits — a version may include multiple commits with no way to
  distinguish between them in a running cluster

---

### Option 3: Date-based tags (e.g., `2026-03-01-1430`)

**Description:** Tag with a timestamp at build time.

**Pros:**
- Unique per build
- Human-readable approximate age

**Cons:**
- No direct mapping to source code — cannot `git checkout` the code running in production
  without a separate mapping table
- Collisions possible if two builds run within the same minute
- Timezone ambiguity

---

### Option 4: Git SHA tags ✓ *Selected*

**Description:** Tag each image with `$(git rev-parse --short HEAD)`. Since every commit has
a unique SHA, every image push has a unique, non-reusable tag.

```bash
TAG=$(git rev-parse --short HEAD)
docker build --platform linux/arm64 -t ${IMAGE}:${TAG} .
docker push ${IMAGE}:${TAG}
helm upgrade --install user-app ./charts/user-app \
  --set image.tag=${TAG} --wait
```

**Pros:**
- Every image tag maps 1:1 to a commit — `git show <tag>` shows exactly what code is running
- Tags are immutable by convention — a SHA cannot be reused unless a commit is amended
  (which is prohibited by ADR-013)
- Rollback is a Helm operation: `helm rollout undo` restores the previous SHA tag
- CI pipelines derive the tag automatically — no manual version management

**Cons:**
- Tags are not human-readable version numbers (addressed by maintaining a changelog)
- Short SHAs have a theoretical collision probability for very large repositories (use
  `--short=8` for lower collision risk if the repo grows significantly)

---

## Consequences

### Positive
- Every pod in the cluster can be traced to an exact commit: `kubectl get pod -o jsonpath='{.spec.containers[0].image}'`
- No image cache staleness across Karpenter-provisioned nodes
- Rollbacks are unambiguous — the previous Helm release recorded the previous SHA

### Trade-offs
- `imagePullPolicy: Always` adds a registry round-trip on every pod start — mitigated by
  ECR's fast pull performance within the same region
- The build pipeline must always run from a clean working directory — dirty (uncommitted)
  changes are not reflected in the tag

---

## Implementation Rules

> These rules apply to all Claude agents and engineers building and deploying application images.

**DO:**
- Derive the image tag from the current git commit in every build script:
  ```bash
  TAG=$(git rev-parse --short HEAD)
  ```
- Set `imagePullPolicy: Always` in the Helm chart to ensure every pod start fetches the
  declared tag from ECR:
  ```yaml
  image:
    pullPolicy: Always
  ```
- Verify the image tag before deploying — the running pod's image must match the pushed SHA:
  ```bash
  kubectl get pods -l app.kubernetes.io/name=<app> \
    -o jsonpath='{.items[*].spec.containers[0].image}'
  ```
- Verify image architecture after build (see ADR-004):
  ```bash
  docker inspect ${IMAGE}:${TAG} --format '{{.Architecture}}'
  # Must be: arm64
  ```

**DO NOT:**
- Push to `latest`, `stable`, or any other mutable tag
- Reuse a tag — if a build fails and you rebuild, use a new commit or amend (on a non-pushed
  branch only, per ADR-013)
- Use `imagePullPolicy: IfNotPresent` — this defeats the guarantee on Karpenter-provisioned
  nodes that do not have a local image cache

---

## Verification

```bash
# Confirm all running pods use a SHA-tagged image (not 'latest' or semver)
kubectl get pods -l app.kubernetes.io/name=<app> \
  -o jsonpath='{range .items[*]}{.spec.containers[0].image}{"\n"}{end}'
# Expected: all lines contain a short hex SHA (e.g., repo-url:a1b2c3d)

# Confirm the tag resolves to a real commit
IMAGE_TAG=$(kubectl get pods -l app.kubernetes.io/name=<app> \
  -o jsonpath='{.items[0].spec.containers[0].image}' | cut -d: -f2)
git log --oneline | grep "^${IMAGE_TAG}"
# Expected: shows the commit that was deployed

# Confirm imagePullPolicy is Always
kubectl get deployment <name> \
  -o jsonpath='{.spec.template.spec.containers[0].imagePullPolicy}'
# Expected: Always
```

---

## References

- [Docker image tagging best practices](https://docs.docker.com/build/building/best-practices/#use-multi-stage-builds)
- [ECR image tagging](https://docs.aws.amazon.com/AmazonECR/latest/userguide/image-retag.html)
- Related: [ADR-004 — arm64 Architecture](./ADR-004-arm64-graviton-architecture.md)
- Related: [ADR-013 — GitFlow Branching Strategy](./ADR-013-gitflow-branching-strategy.md)
