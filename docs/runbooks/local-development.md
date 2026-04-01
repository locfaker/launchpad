# Local Development Runbook

## Purpose

Use this runbook to start Launchpad on a developer machine and validate the core control plane locally.

## Prerequisites

- Java 21
- Maven Wrapper or local Maven
- Docker Desktop or Docker Engine
- `docker compose`
- `kubectl`
- `helm`
- A local Kubernetes cluster: `k3d` (recommended) or `kind`

## Local Services

Launchpad expects these dependencies in local development:

- PostgreSQL
- Redis
- Loki
- Grafana

## Procedure

1. Start the local dependency stack.
2. Start a local Kubernetes cluster if you need to validate deployment behavior.
3. Run database migrations through the application startup.
4. Start the Spring Boot control plane in the `local` profile.
5. Verify health, auth, and deployment APIs.

## Exact Steps

```bash
docker compose -f infra/local/docker-compose.yml up -d
```

```bash
./mvnw -f services/control-plane/pom.xml test
```

```bash
./mvnw -f services/control-plane/pom.xml spring-boot:run -Dspring-boot.run.profiles=local
```

If you need a local cluster:

```bash
k3d cluster create launchpad
kubectl cluster-info
```

## Verification

- `GET http://localhost:8080/actuator/health` returns `UP`
- `GET http://localhost:8080/actuator/prometheus` returns metrics text
- `POST /api/v1/auth/register` creates a user
- `POST /api/v1/projects` succeeds for an authenticated admin user

Example health check:

```bash
curl http://localhost:8080/actuator/health
```

Expected:

```json
{"status":"UP"}
```

## Recovery

If the application fails to start:

- Check PostgreSQL connectivity and credentials in `application-local.yml`.
- Check whether Flyway migration `V1__baseline.sql` applied cleanly.
- Confirm no other process is using port `8080`.
- Restart the dependency stack with `docker compose down` and `docker compose up -d`.

If local cluster resources are broken:

- Delete and recreate the cluster.
- Reapply bootstrap manifests from the `infra/k8s/bootstrap` directory after the control plane is healthy.

## Troubleshooting

### Application fails during startup

Common causes:

- invalid database URL
- missing encryption key
- schema migration failure
- Docker services not ready

### Actuator health is not `UP`

Common causes:

- database unreachable
- Redis unreachable
- application profile mismatch

### Tests pass but runtime behavior fails

Common causes:

- local environment variables do not match test configuration
- Docker Compose ports conflict with existing services
- the local cluster is not running when deployment tests expect it

