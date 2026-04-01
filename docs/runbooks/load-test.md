# Load Test Runbook

## Purpose

Use this runbook to validate Launchpad behavior under light to moderate load before a demo or release.

## Prerequisites

- A working Launchpad deployment
- A sample project and environment
- At least one valid deploy token
- `k6` or an equivalent HTTP load tool

## Load-Test Targets

Test these endpoints first:

- `GET /actuator/health`
- `GET /api/v1/projects/{projectId}/runtime`
- `GET /api/v1/projects/{projectId}/logs`
- `POST /api/v1/deployments`

## Procedure

1. Start with read-heavy traffic against health and runtime endpoints.
2. Add a low rate of deployment creation requests with unique idempotency keys.
3. Repeat one deployment request with the same idempotency key to confirm deduplication.
4. Observe API latency, error rates, queue depth, and database pressure.
5. Stop the test if error rate or deployment duplication increases.

## Exact Steps

Run a lightweight test profile such as:

```bash
k6 run load-tests/launchpad-smoke.js
```

If you do not have a script yet, generate traffic manually with a few parallel shell loops or a simple HTTP benchmark tool.

## Verification

- Health endpoint remains `UP`
- Read APIs respond within acceptable latency
- Only one deployment is created per idempotency key
- Worker queue depth does not grow without bound
- No duplicate rollback or audit events are created

## Recovery

If latency grows too high:

- Reduce request concurrency.
- Check database CPU and connection pool usage.
- Check worker throughput and queue depth.
- Check ingress controller and application pod saturation.

If duplicate deployments appear:

- Confirm the idempotency key is stable per logical request.
- Confirm the client is not retrying with different payloads.
- Inspect deployment creation logs for retry behavior.

## Troubleshooting

### Read endpoints fail first

Likely causes:

- database pressure
- connection pool exhaustion
- application memory pressure

### Deploy creation becomes slow

Likely causes:

- too many concurrent writes
- database lock contention
- slow external registry or Kubernetes API calls

