# Launchpad Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Build a portfolio-grade deployment control plane in Spring Boot that can accept CI-driven deployment requests, release containerized applications to Kubernetes through Helm, surface runtime health and logs, and support auditable rollback.

**Architecture:** Implement Launchpad as a modular monolith in a single Spring Boot service named `control-plane`. Persist core state in PostgreSQL, use Redis only for transient concerns, and drive deployments through a database-backed job queue plus Kubernetes and Helm adapters. Keep source builds outside the platform; GitHub Actions will build images and call Launchpad's deployment API.

**Tech Stack:** Java 21, Spring Boot, Spring Security, Spring Data JPA, Flyway, PostgreSQL, Redis, Testcontainers, Docker Compose, Kubernetes, Helm, GitHub Actions, Micrometer, Prometheus, Grafana, Loki, OpenTelemetry.

---

## Assumptions

- Repository root will be `launchpad/`.
- Development happens on Linux or WSL2 with Docker, kubectl, helm, and a local Kubernetes cluster available.
- The initial release may be API-first; a minimal UI is optional and not on the critical path.
- `@superpowers:verification-before-completion` should be applied before claiming the implementation is complete.

## Target Repository Layout

```text
launchpad/
  .github/workflows/
  docs/
    plans/
    runbooks/
    architecture/
  infra/
    local/
    helm/launchpad-app/
    k8s/bootstrap/
    scripts/
  services/
    control-plane/
      pom.xml
      src/main/java/com/launchpad/controlplane/
      src/main/resources/
      src/test/java/com/launchpad/controlplane/
```

### Task 1: Bootstrap the repository and local runtime

**Files:**
- Create: `README.md`
- Create: `.gitignore`
- Create: `services/control-plane/pom.xml`
- Create: `services/control-plane/src/main/java/com/launchpad/controlplane/LaunchpadApplication.java`
- Create: `services/control-plane/src/main/resources/application.yml`
- Create: `services/control-plane/src/main/resources/application-local.yml`
- Create: `services/control-plane/src/test/java/com/launchpad/controlplane/LaunchpadApplicationTests.java`
- Create: `infra/local/docker-compose.yml`
- Create: `infra/scripts/dev-up.sh`
- Create: `infra/scripts/dev-down.sh`

**Step 1: Write the failing smoke test**

Create a minimal `@SpringBootTest` that asserts the application context loads.

**Step 2: Run the test before scaffolding**

Run: `./mvnw -f services/control-plane/pom.xml test`
Expected: fail because the project does not exist yet.

**Step 3: Scaffold the Spring Boot service**

Include these dependencies in `services/control-plane/pom.xml`:

- web
- validation
- actuator
- security
- data-jpa
- flyway
- postgresql
- testcontainers

Add a local Docker Compose file with:

- PostgreSQL
- Redis
- Grafana
- Loki

**Step 4: Verify local boot**

Run:

```bash
docker compose -f infra/local/docker-compose.yml up -d
./mvnw -f services/control-plane/pom.xml test
./mvnw -f services/control-plane/pom.xml spring-boot:run -Dspring-boot.run.profiles=local
```

Expected:

- tests pass
- application starts on `localhost:8080`
- `GET /actuator/health` returns `UP`

**Step 5: Commit**

```bash
git add .
git commit -m "chore: bootstrap launchpad control plane"
```

### Task 2: Establish persistence, migrations, and authentication foundations

**Files:**
- Create: `services/control-plane/src/main/resources/db/migration/V1__baseline.sql`
- Create: `services/control-plane/src/main/java/com/launchpad/controlplane/common/persistence/`
- Create: `services/control-plane/src/main/java/com/launchpad/controlplane/auth/`
- Create: `services/control-plane/src/main/java/com/launchpad/controlplane/team/`
- Create: `services/control-plane/src/test/java/com/launchpad/controlplane/support/PostgresIntegrationTest.java`
- Create: `services/control-plane/src/test/resources/application-test.yml`

**Step 1: Write failing persistence tests**

Add integration tests that assert baseline tables exist after startup:

- `users`
- `teams`
- `team_members`
- `projects`
- `environments`
- `releases`
- `deployments`
- `deployment_jobs`
- `audit_events`

**Step 2: Run the tests**

Run: `./mvnw -f services/control-plane/pom.xml -Dtest=PostgresIntegrationTest test`
Expected: fail because the schema is missing.

**Step 3: Implement the baseline schema**

Create `V1__baseline.sql` with:

- UUID primary keys
- explicit foreign keys
- unique constraints on identity and project slugs
- timestamps for auditable entities
- indexes on deployment and audit timelines

**Step 4: Implement auth and team foundations**

Create:

- `AuthController`
- `AuthService`
- `JwtService`
- `SecurityConfig`
- `TeamService`

Minimum API surface:

- `POST /api/v1/auth/register`
- `POST /api/v1/auth/login`
- `POST /api/v1/auth/refresh`
- `GET /api/v1/me`
- `POST /api/v1/teams`

Rules:

- password hashing with BCrypt or Argon2
- hashed refresh tokens only
- login success and failure generate audit events

**Step 5: Verify**

Run:

```bash
./mvnw -f services/control-plane/pom.xml test
curl -X POST http://localhost:8080/api/v1/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@launchpad.dev","password":"P@ssw0rd123!"}'
```

Expected:

- tests pass
- registration returns `201 Created`

**Step 6: Commit**

```bash
git add .
git commit -m "feat: add persistence baseline and authentication foundations"
```

### Task 3: Implement project configuration, environments, secrets, and deploy tokens

**Files:**
- Create: `services/control-plane/src/main/java/com/launchpad/controlplane/project/`
- Create: `services/control-plane/src/main/java/com/launchpad/controlplane/security/SecretEncryptionService.java`
- Create: `services/control-plane/src/main/java/com/launchpad/controlplane/security/DeployTokenService.java`
- Create: `services/control-plane/src/test/java/com/launchpad/controlplane/project/ProjectConfigurationTest.java`
- Modify: `services/control-plane/src/main/resources/db/migration/V1__baseline.sql`

**Step 1: Write failing configuration tests**

Cover:

- team admin can create project and environments
- developer cannot rotate a production deploy token
- secrets are stored encrypted, not in plaintext
- deploy tokens are only shown once at creation time

**Step 2: Run focused tests**

Run: `./mvnw -f services/control-plane/pom.xml -Dtest=ProjectConfigurationTest test`
Expected: fail.

**Step 3: Implement project and environment APIs**

Add:

- `POST /api/v1/projects`
- `POST /api/v1/projects/{projectId}/environments`
- `PUT /api/v1/environments/{environmentId}/secrets`
- `POST /api/v1/projects/{projectId}/deploy-tokens`

Required rules:

- environment names validated against an approved slug format
- each environment stores namespace, base domain, port, health path, and replica target
- secret updates in production require `TEAM_ADMIN`

**Step 4: Implement encryption and token hashing**

- encrypt secrets using AES-GCM
- store deploy tokens only as hashes
- never return full secret values after write

**Step 5: Verify**

Run: `./mvnw -f services/control-plane/pom.xml test`
Expected: tests pass and database inspection shows no plaintext secrets or raw tokens.

**Step 6: Commit**

```bash
git add .
git commit -m "feat: add project configuration secrets and deploy tokens"
```

### Task 4: Implement deployment creation, idempotency, and the job queue

**Files:**
- Create: `services/control-plane/src/main/java/com/launchpad/controlplane/deployment/domain/`
- Create: `services/control-plane/src/main/java/com/launchpad/controlplane/deployment/api/DeploymentController.java`
- Create: `services/control-plane/src/main/java/com/launchpad/controlplane/deployment/service/DeploymentService.java`
- Create: `services/control-plane/src/main/java/com/launchpad/controlplane/deployment/service/JobQueueService.java`
- Create: `services/control-plane/src/main/java/com/launchpad/controlplane/deployment/service/IdempotencyService.java`
- Create: `services/control-plane/src/test/java/com/launchpad/controlplane/deployment/DeploymentControllerTest.java`

**Step 1: Write failing deployment API tests**

Cover:

- missing `Idempotency-Key` returns `400`
- duplicate `Idempotency-Key` returns the same deployment record
- invalid image references return validation errors
- a valid request creates `release`, `deployment`, and `deployment_job`

**Step 2: Run focused tests**

Run: `./mvnw -f services/control-plane/pom.xml -Dtest=DeploymentControllerTest test`
Expected: fail.

**Step 3: Implement deployment contracts**

The request should include:

- project id
- environment id
- image reference
- git SHA
- trigger source

Deployment lifecycle states:

- `PENDING`
- `APPLYING`
- `HEALTH_CHECKING`
- `HEALTHY`
- `FAILED`
- `ROLLED_BACK`

**Step 4: Implement queue semantics**

- store jobs in PostgreSQL
- claim work with `FOR UPDATE SKIP LOCKED`
- support retries with backoff and lock expiry
- persist failure metadata

**Step 5: Verify**

Run:

```bash
./mvnw -f services/control-plane/pom.xml test
curl -X POST http://localhost:8080/api/v1/deployments \
  -H "Authorization: Bearer <token>" \
  -H "Idempotency-Key: demo-001" \
  -H "Content-Type: application/json" \
  -d '{"projectId":"<project-id>","environmentId":"<env-id>","image":"ghcr.io/acme/demo:sha-123abc","gitSha":"123abc","triggerSource":"manual"}'
```

Expected: `202 Accepted` with a deployment id and persisted queue state.

**Step 6: Commit**

```bash
git add .
git commit -m "feat: add deployment api and database-backed job queue"
```

### Task 5: Build the Kubernetes and Helm deployment adapters

**Files:**
- Create: `infra/helm/launchpad-app/Chart.yaml`
- Create: `infra/helm/launchpad-app/values.yaml`
- Create: `infra/helm/launchpad-app/templates/deployment.yaml`
- Create: `infra/helm/launchpad-app/templates/service.yaml`
- Create: `infra/helm/launchpad-app/templates/ingress.yaml`
- Create: `infra/helm/launchpad-app/templates/secret.yaml`
- Create: `services/control-plane/src/main/java/com/launchpad/controlplane/deployment/k8s/`
- Create: `services/control-plane/src/test/java/com/launchpad/controlplane/deployment/k8s/HelmAdapterTest.java`

**Step 1: Write failing adapter tests**

Cover:

- values renderer maps image, probes, replicas, and ingress host correctly
- namespace naming is deterministic
- executor fails with a clear reason when cluster credentials are missing

**Step 2: Run focused tests**

Run: `./mvnw -f services/control-plane/pom.xml -Dtest=HelmAdapterTest test`
Expected: fail.

**Step 3: Implement the chart**

The chart must support:

- image repository and tag
- container port
- environment variables
- secret environment variables
- resource requests and limits
- readiness, liveness, and startup probes
- ingress and TLS options

**Step 4: Implement the executor**

Recommended behavior:

- ensure the namespace exists
- ensure image pull secret state if registry credentials are configured
- execute `helm upgrade --install`
- capture command output for diagnostics

Keep command execution isolated in a single adapter rather than spreading shell calls across services.

**Step 5: Verify against a local cluster**

Run:

```bash
k3d cluster create launchpad
kubectl cluster-info
```

Then deploy a sample image such as `nginx:stable-alpine`.

Expected:

- namespace exists
- deployment becomes available
- service and ingress resources are created

**Step 6: Commit**

```bash
git add .
git commit -m "feat: add helm-based kubernetes deployment adapter"
```

### Task 6: Implement workers, reconciliation, runtime visibility, and rollback

**Files:**
- Create: `services/control-plane/src/main/java/com/launchpad/controlplane/deployment/service/DeploymentWorker.java`
- Create: `services/control-plane/src/main/java/com/launchpad/controlplane/deployment/service/DeploymentReconciler.java`
- Create: `services/control-plane/src/main/java/com/launchpad/controlplane/deployment/api/RuntimeController.java`
- Create: `services/control-plane/src/main/java/com/launchpad/controlplane/deployment/api/LogsController.java`
- Create: `services/control-plane/src/main/java/com/launchpad/controlplane/deployment/api/RollbackController.java`
- Create: `services/control-plane/src/main/java/com/launchpad/controlplane/deployment/service/LokiQueryService.java`
- Create: `services/control-plane/src/test/java/com/launchpad/controlplane/deployment/RolloutWorkflowTest.java`

**Step 1: Write failing workflow tests**

Cover:

- a worker locks one job and transitions deployment state correctly
- a healthy rollout writes a runtime snapshot
- a failed rollout persists failure details
- rollback creates a new deployment from the most recent healthy release

**Step 2: Run focused tests**

Run: `./mvnw -f services/control-plane/pom.xml -Dtest=RolloutWorkflowTest test`
Expected: fail.

**Step 3: Implement worker and reconciler behavior**

Required behavior:

- poll pending jobs on a schedule
- transition deployment from `PENDING` to `APPLYING`
- invoke the Helm adapter
- reconcile rollout status until healthy or timed out
- persist runtime metadata such as ready replica count and ingress URL

**Step 4: Implement runtime and log APIs**

Add:

- `GET /api/v1/projects/{projectId}/runtime`
- `GET /api/v1/projects/{projectId}/logs`
- `POST /api/v1/deployments/{deploymentId}/rollback`

Log queries must be constrained to safe server-generated Loki filters.

**Step 5: Verify**

Run an end-to-end flow:

1. deploy a healthy sample image
2. confirm status becomes `HEALTHY`
3. deploy a broken image or invalid port
4. confirm status becomes `FAILED`
5. call rollback
6. confirm a new deployment reaches `HEALTHY`

**Step 6: Commit**

```bash
git add .
git commit -m "feat: add rollout reconciliation runtime apis and rollback"
```

### Task 7: Add auditability, validation, rate limiting, and safety controls

**Files:**
- Create: `services/control-plane/src/main/java/com/launchpad/controlplane/audit/`
- Create: `services/control-plane/src/main/java/com/launchpad/controlplane/common/api/GlobalExceptionHandler.java`
- Create: `services/control-plane/src/main/java/com/launchpad/controlplane/common/ratelimit/RateLimitFilter.java`
- Create: `services/control-plane/src/test/java/com/launchpad/controlplane/audit/AuditContractTest.java`
- Create: `services/control-plane/src/test/java/com/launchpad/controlplane/common/ApiValidationTest.java`

**Step 1: Write failing safety tests**

Cover:

- deployment creation emits an audit event
- repeated failed logins are rate limited
- invalid image references return structured errors
- production deploys using `:latest` are rejected

**Step 2: Run focused tests**

Run: `./mvnw -f services/control-plane/pom.xml -Dtest=AuditContractTest,ApiValidationTest test`
Expected: fail.

**Step 3: Implement safety controls**

Add:

- structured error responses with request id and error code
- login and deployment endpoint rate limits
- registry host allowlist checks
- production image-tag policy validation
- audit events for login, secret updates, deploy, rollback, and token rotation

**Step 4: Verify**

Run: `./mvnw -f services/control-plane/pom.xml test`
Expected: pass.

Manual checks:

- audit table contains deploy and rollback entries
- invalid production deploy request returns `422`

**Step 5: Commit**

```bash
git add .
git commit -m "feat: add audit validation and deployment safety controls"
```

### Task 8: Add observability, environment bootstrap, and CI/CD

**Files:**
- Create: `infra/k8s/bootstrap/namespaces.yaml`
- Create: `infra/k8s/bootstrap/prometheus-values.yaml`
- Create: `infra/k8s/bootstrap/loki-values.yaml`
- Create: `infra/k8s/bootstrap/grafana-values.yaml`
- Create: `services/control-plane/src/main/resources/application-observability.yml`
- Create: `.github/workflows/control-plane-ci.yml`
- Create: `.github/workflows/control-plane-cd.yml`
- Create: `.github/workflows/sample-app-deploy.yml`
- Create: `docs/runbooks/local-cluster.md`
- Create: `docs/runbooks/staging-bootstrap.md`
- Create: `docs/runbooks/release.md`
- Create: `docs/runbooks/rollback.md`

**Step 1: Write failing operational checks**

Document success conditions:

- metrics endpoint is exposed
- dashboards can visualize deployment metrics
- CI runs tests on pull requests
- a sample app workflow can trigger Launchpad deployments

**Step 2: Bootstrap observability**

Install or define manifests for:

- ingress controller
- cert-manager
- Prometheus stack
- Loki stack

Commit the configuration as code rather than relying only on one-off commands.

**Step 3: Implement CI/CD**

`control-plane-ci.yml` should:

- install Java 21
- cache Maven dependencies
- run all tests

`control-plane-cd.yml` should:

- build the Launchpad image
- push to GHCR
- update staging

`sample-app-deploy.yml` should:

- build a sample application image
- push it to GHCR
- call Launchpad's deployment API with a deploy token

**Step 4: Verify**

Run:

```bash
./mvnw -f services/control-plane/pom.xml test
curl http://localhost:8080/actuator/prometheus
kubectl get pods -A
```

Expected:

- tests pass
- metrics are exposed
- cluster monitoring components are healthy
- sample app deployment appears in Launchpad

**Step 5: Commit**

```bash
git add .
git commit -m "feat: add observability bootstrap and ci cd pipelines"
```

### Task 9: Final hardening, demo preparation, and recruiter-facing documentation

**Files:**
- Create: `docs/architecture/system-context.md`
- Create: `docs/architecture/deployment-flow.md`
- Create: `docs/runbooks/demo-script.md`
- Create: `docs/runbooks/security-checklist.md`
- Create: `docs/runbooks/failure-scenarios.md`
- Create: `docs/runbooks/load-test.md`
- Modify: `README.md`

**Step 1: Prepare final documentation**

The `README.md` must explain:

- what Launchpad is
- the core deployment workflow
- local setup steps
- architecture overview
- operational stack
- screenshots or references for dashboards and deployment history

**Step 2: Rehearse the demo path**

The demo should show:

1. project creation
2. environment setup
3. CI-driven deployment
4. healthy rollout
5. failed rollout
6. rollback
7. logs and metrics
8. audit history

**Step 3: Run final verification**

Run:

```bash
./mvnw -f services/control-plane/pom.xml test
docker compose -f infra/local/docker-compose.yml up -d
kubectl get pods -A
```

Then manually verify:

- auth works
- secret writes remain encrypted
- deployment history is accurate
- rollback creates a new deployment record
- observability data is accessible

**Step 4: Run one light load or concurrency test**

Validate that:

- duplicate idempotency keys do not create duplicate deployments
- read APIs remain responsive under light load
- worker locking prevents double-processing

**Step 5: Commit**

```bash
git add .
git commit -m "docs: finalize launchpad runbooks and demo materials"
```

## Done When

- [ ] The control plane starts locally against PostgreSQL and Redis.
- [ ] Auth, roles, project configuration, encrypted secrets, and deploy tokens work.
- [ ] A CI pipeline can trigger a deployment with an immutable image reference.
- [ ] Launchpad can deploy at least two sample applications to Kubernetes.
- [ ] Rollout state, logs, and audit history are visible through the platform APIs.
- [ ] A failed deployment can be rolled back to a previously healthy release.
- [ ] Observability is configured as code and validated in a real cluster.
- [ ] The README and runbooks are strong enough for a reviewer with zero project context.

## Risks to Watch During Execution

- Do not expand scope into buildpacks or multi-cluster support before the MVP is stable.
- Do not allow controllers to call shell or Kubernetes adapters directly; keep the boundaries explicit.
- Do not store raw secrets or raw deploy tokens after creation.
- Do not trust client-reported deployment state; reconcile from Kubernetes.
- Do not skip operational documentation, because the platform story is weaker without runbooks.
