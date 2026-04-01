# ADR-003: External CI for Image Builds

## Status

Accepted

## Context

Launchpad needs to support a Git-driven deployment workflow, but building source code inside the platform would require build sandboxing, source credential management, caching, and significantly more operational surface area.

## Decision

Keep image builds outside Launchpad. GitHub Actions will build and push immutable container images, then call Launchpad's deployment API with the resulting image reference.

## Consequences

### Positive

- Launchpad remains focused on deployment orchestration
- easier to secure and operate the MVP
- still demonstrates end-to-end CI/CD integration

### Negative

- Launchpad is not a full internal developer platform yet
- build pipeline behavior depends on external CI configuration

### Follow-Up

If future scope justifies it, source builds can be added behind a separate build subsystem rather than folded into the deployment control plane.
