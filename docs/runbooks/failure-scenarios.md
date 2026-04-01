# Failure Scenarios Runbook

## Purpose

Use this runbook to respond to common failure modes in Launchpad and the workloads it deploys.

## Common Scenarios

### 1. Deployment Stuck in `PENDING`

Symptoms:

- deployment exists but does not progress
- no worker activity is visible

Actions:

1. Check worker process health.
2. Check database connectivity.
3. Confirm the job queue contains pending work.
4. Inspect worker logs for lock or authentication errors.

Verification:

- job is claimed and deployment moves forward

### 2. Deployment Fails on Image Pull

Symptoms:

- rollout fails quickly after apply
- pod events mention image pull errors

Actions:

1. Confirm the image reference exists in the registry.
2. Confirm registry credentials are valid.
3. Confirm the registry host is allowlisted.
4. Retry the deployment after fixing the image or credentials.

Verification:

- pods pull the image successfully

### 3. Deployment Fails Readiness

Symptoms:

- pods start but do not become healthy
- deployment ends in `FAILED`

Actions:

1. Inspect application logs.
2. Check readiness and liveness probe definitions.
3. Confirm the service listens on the configured port.
4. Confirm environment variables and secrets are correct.

Verification:

- runtime snapshot shows healthy replicas

### 4. Rollback Does Not Recover

Symptoms:

- rollback request accepted but rollout still fails

Actions:

1. Confirm the previous healthy release still exists in the registry.
2. Check for secret drift between releases.
3. Check cluster capacity and scheduling pressure.
4. Restore manually if automated rollback cannot complete.

Verification:

- the restored release becomes healthy and serves traffic

### 5. Database Is Unavailable

Symptoms:

- control plane health degrades
- API requests fail
- jobs do not progress

Actions:

1. Restore database connectivity.
2. Check connection pool exhaustion.
3. Check migration or schema lock issues.
4. Restore from backup if needed.

Verification:

- health endpoint returns `UP`
- deployment history is readable again

## Escalation

Escalate to platform admin support if:

- the failure involves data loss
- multiple environments are affected
- the registry or cluster control plane is unavailable
- security compromise is suspected

