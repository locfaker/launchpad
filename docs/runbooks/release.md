# Release Runbook

## Purpose

Use this runbook to release Launchpad or a sample workload through the standard CI-driven deployment path.

## Prerequisites

- A built container image pushed to the registry
- A valid deploy token or authenticated user session
- Target project and environment already created in Launchpad
- A healthy staging cluster

## Release Inputs

- Project ID
- Environment ID
- Immutable image reference
- Git SHA
- Idempotency key

## Procedure

1. Confirm the image tag is immutable and points to the intended commit.
2. Confirm the target environment has the correct namespace, base domain, and health check settings.
3. Trigger the deployment through the API or GitHub Actions workflow.
4. Wait for the deployment to transition from `PENDING` to `HEALTHY`.
5. Verify logs, runtime snapshot, and ingress access.

## Exact Steps

```bash
curl -X POST http://localhost:8080/api/v1/deployments \
  -H "Authorization: Bearer <token>" \
  -H "Idempotency-Key: release-001" \
  -H "Content-Type: application/json" \
  -d '{
    "projectId":"<project-id>",
    "environmentId":"<environment-id>",
    "image":"ghcr.io/<org>/<app>:sha-<git-sha>",
    "gitSha":"<git-sha>",
    "triggerSource":"github_actions"
  }'
```

Poll the deployment until it becomes healthy:

```bash
curl http://localhost:8080/api/v1/deployments/<deployment-id>
```

## Verification

- Deployment status becomes `HEALTHY`
- The app responds through the configured ingress host
- `GET /api/v1/projects/{projectId}/runtime` shows a recent snapshot
- `GET /api/v1/projects/{projectId}/logs` returns recent logs
- An audit event exists for the release

## Recovery

If deployment fails:

- Read the deployment failure reason in the deployment detail response.
- Inspect the runtime snapshot for probe or replica failures.
- Inspect logs for image pull, crash loop, or port mismatch errors.
- Roll back to the last healthy release if the issue is production-impacting.

If the API request is rejected:

- Confirm the `Idempotency-Key` header is present.
- Confirm the caller has permission for the target project and environment.
- Confirm the image registry host is allowlisted.
- Confirm the production image policy does not reject the tag.

## Troubleshooting

### Deployment stays in `PENDING`

Likely causes:

- worker is not running
- queue job is locked or stuck
- database is unavailable

### Deployment moves to `FAILED`

Likely causes:

- invalid image tag
- readiness probe never became healthy
- application crashed on startup
- ingress or service port mismatch

### Idempotent retry returns unexpected data

Likely causes:

- different payload with the same idempotency key
- stale client cache
- duplicate CI workflow execution

