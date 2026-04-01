# Security Checklist

## Purpose

Use this checklist before staging or production release to reduce the chance of credential leakage, unauthorized deploys, or insecure runtime configuration.

## Checklist

### Identity and Access

- Confirm all users have the minimum required role.
- Confirm production write access is limited to team admins or platform admins.
- Confirm deploy tokens are scoped to the correct project or environment.
- Confirm refresh tokens and session secrets are rotated on schedule.

### Secrets

- Confirm environment secrets are encrypted at rest.
- Confirm no secret values are logged.
- Confirm no secret values are returned in API responses after creation.
- Confirm encryption keys are stored outside source control.

### Deployment Safety

- Confirm production rejects mutable tags such as `latest`.
- Confirm registry host allowlists are current.
- Confirm health check configuration is present for every production environment.
- Confirm rollback has a known healthy target.

### Kubernetes and Network

- Confirm Launchpad uses a dedicated service account.
- Confirm namespace scoping is restricted.
- Confirm ingress hosts are correct and TLS is enabled.
- Confirm network policies are in place where the cluster supports them.

### Observability and Audit

- Confirm audit events are emitted for login, deploy, secret changes, token rotation, and rollback.
- Confirm metrics are exposed only on the intended control plane endpoint.
- Confirm logs do not contain raw tokens or secrets.

## Verification

Run a quick pre-release pass:

```bash
curl http://localhost:8080/actuator/health
curl http://localhost:8080/actuator/prometheus
```

Inspect a recent deployment and confirm audit history is present.

## Recovery

If a security issue is detected:

- Revoke the affected deploy token immediately.
- Rotate the affected secret or key.
- Roll back the problematic deployment if needed.
- Preserve logs and audit data for investigation.

## Troubleshooting

### A secret appears in logs

Likely causes:

- verbose exception logging
- debug mode enabled in the wrong environment
- application code wrote raw request bodies to logs

### A deploy token is rejected unexpectedly

Likely causes:

- token expired or revoked
- wrong environment or project scope
- signature or hashing mismatch

