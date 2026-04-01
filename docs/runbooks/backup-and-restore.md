# Backup and Restore Runbook

## Purpose

Use this runbook to protect Launchpad metadata and restore the control plane after data loss or environment failure.

## Prerequisites

- Access to the PostgreSQL primary and backup location
- Access to the object store or storage bucket used for backups
- Access to the Kubernetes cluster or hosting environment

## Backup Scope

Back up the following:

- PostgreSQL database
- Application secrets used for Launchpad itself
- Helm values and bootstrap manifests
- Monitoring and logging configuration

Do not rely on Kubernetes resources alone as the source of truth for deployment history or audit data.

## Backup Procedure

1. Stop or pause write traffic if you are performing a controlled backup window.
2. Take a PostgreSQL backup.
3. Verify the backup artifact integrity.
4. Store the backup in the approved retention location.
5. Record the backup timestamp and retention class.

## Exact Steps

Use your standard PostgreSQL backup mechanism. For a logical backup, a common pattern is:

```bash
pg_dump -h <db-host> -U <db-user> -d <db-name> -Fc -f launchpad-backup.dump
```

Verify the file exists and is non-empty:

```bash
ls -lh launchpad-backup.dump
```

## Restore Procedure

1. Provision an empty PostgreSQL target.
2. Restore the latest validated backup.
3. Apply the current Flyway migration set.
4. Restart Launchpad with the restored database.
5. Validate a few known deployment and audit records.

## Exact Steps

```bash
pg_restore -h <db-host> -U <db-user> -d <db-name> launchpad-backup.dump
```

Then restart the control plane and check health:

```bash
curl http://localhost:8080/actuator/health
```

## Verification

- Launchpad starts without schema errors
- Known teams, projects, and deployments are present
- Audit events are present
- One sample deployment can be queried successfully

## Recovery

If restore fails:

- Confirm the backup file is valid and complete.
- Confirm the target database version is compatible.
- Reapply migrations only after the restore data is in place.
- If partial restore occurred, restore again from the last known good backup.

## Troubleshooting

### Restore fails on migration mismatch

Likely causes:

- backup came from a different schema version
- migrations were changed without a corresponding restore test

### Data is missing after restore

Likely causes:

- wrong backup file
- incomplete backup window
- retention process selected an older snapshot

