# Rollback Runbook

## Purpose

Use this runbook to recover from a bad deployment by rolling back to the last healthy release.

## Prerequisites

- Access to Launchpad
- Permission to rollback the target environment
- At least one previous healthy release exists

## Rollback Policy

Rollback should be used when:

- deployment health fails after release
- customer impact is observed
- logs indicate a regression in startup, probes, or runtime behavior

Rollback should not be used when:

- the latest deployment is still under normal stabilization
- the problem is unrelated to the last release
- the environment has no verified healthy release to revert to

## Procedure

1. Identify the failed deployment and confirm the prior healthy release.
2. Inspect logs and runtime snapshot for the failure signature.
3. Trigger rollback from the API or UI.
4. Wait for the rollback deployment to become healthy.
5. Verify the restored version is serving traffic.

## Exact Steps

```bash
curl -X POST http://localhost:8080/api/v1/deployments/<deployment-id>/rollback \
  -H "Authorization: Bearer <token>"
```

Check the new deployment:

```bash
curl http://localhost:8080/api/v1/deployments/<new-deployment-id>
```

## Verification

- New deployment status becomes `HEALTHY`
- The service responds with the previous version
- Deployment history shows both the failed release and rollback release
- Audit history records the rollback actor and source deployment

## Recovery

If rollback fails:

- Check whether the previous release image is still available in the registry.
- Confirm the old release configuration still matches the environment.
- Confirm the cluster has enough resources to schedule the rollback.
- If rollback cannot be automated, deploy the last healthy image manually through the standard release path.

## Troubleshooting

### Rollback request is rejected

Likely causes:

- caller lacks permission
- deployment has no previous healthy release
- environment is misconfigured

### Rollback starts but never becomes healthy

Likely causes:

- old image is now broken or deleted
- environment secret drift
- cluster resource pressure

