# Launchpad Technical Design

## Document Control

| Field | Value |
| --- | --- |
| Status | Draft |
| Last Updated | 2026-04-01 |
| Intended Audience | Backend engineers, DevOps engineers, technical interviewers |
| Primary Stack | Spring Boot, PostgreSQL, Redis, Kubernetes, Helm, GitHub Actions |

## 1. Executive Summary

Launchpad is a self-service deployment control plane for small engineering teams. It standardizes how containerized applications are released to Kubernetes, manages application configuration and secrets, records an auditable deployment history, and exposes enough operational data to support troubleshooting and rollback.

The system is intentionally scoped as a portfolio-grade production project rather than a full platform-as-a-service. The first release will not attempt to build source code inside the platform, manage multiple clusters, or support arbitrary infrastructure primitives. Instead, it will focus on a disciplined deployment workflow:

1. A GitHub Actions pipeline builds and pushes a container image.
2. Launchpad receives a deploy request with an immutable image reference.
3. Launchpad renders a standardized Helm release and applies it to Kubernetes.
4. Launchpad reconciles rollout state and exposes runtime status, logs, and rollback actions.

This scope is large enough to demonstrate real backend and DevOps capability while remaining implementable by a single engineer.

## 2. Problem Statement

Small teams often deploy software through a mix of shell scripts, manual `kubectl` commands, or unmanaged Docker Compose files. Those approaches work early, but they create operational weaknesses:

- No central record of who deployed what, where, and when.
- No standard contract for environment variables, health checks, or rollout rules.
- No simple rollback workflow tied to a release record.
- No shared operational surface for logs, deployment history, and runtime state.
- High dependency on one team member who knows the scripts.

Launchpad addresses those weaknesses by introducing a single control plane with standardized deployment contracts and operational visibility.

## 3. Goals

### 3.1 Product Goals

- Provide a repeatable deployment workflow for containerized applications.
- Make deployments auditable and safe enough for team use.
- Expose essential operational signals without requiring direct cluster access.
- Keep the first release small enough to build, test, and demo convincingly.

### 3.2 Portfolio Goals

- Demonstrate Spring Boot architecture beyond CRUD.
- Demonstrate production-oriented DevOps practices: containerization, CI/CD, Kubernetes, ingress, metrics, and logs.
- Show sound engineering judgement through explicit trade-offs and scoped decisions.

## 4. Non-Goals

The first release explicitly does not include:

- Source builds inside the platform using buildpacks, Kaniko, or Tekton.
- Multi-cluster scheduling or region-aware placement.
- Stateful workloads such as databases or object stores.
- Billing, quotas with hard enforcement, or tenant invoicing.
- Shell access into running pods from the UI.
- Arbitrary Kubernetes manifest upload by end users.

These exclusions are intentional. They reduce implementation risk and keep the project centered on deployment orchestration, not generalized infrastructure management.

## 5. Users and Roles

| Role | Responsibilities | Sensitive Actions |
| --- | --- | --- |
| Platform Admin | Operates Launchpad itself, manages global policy, registry allowlists, cluster configuration | Can manage all teams and production settings |
| Team Admin | Owns project configuration within one team | Can edit production secrets, rotate deploy tokens, trigger rollback |
| Developer | Ships application releases and reads operational data | Can deploy and inspect logs but cannot change high-risk settings |
| Viewer | Read-only access to runtime and deployment history | No write actions |

Every write operation affecting production configuration or runtime state must generate an audit event.

## 6. Scope by Release

### 6.1 MVP

- Authentication and role-based authorization
- Team, project, and environment management
- Encrypted environment secret storage
- Deploy token issuance for CI/CD pipelines
- Deployment API with idempotency support
- Helm-based deployment to Kubernetes
- Rollout reconciliation and health tracking
- Runtime status endpoint
- Log retrieval for recent application logs
- Rollback to the last healthy release
- Audit history for high-risk actions
- Baseline observability for Launchpad itself

### 6.2 Post-MVP

- Preview environments per pull request
- Chat notifications for deploy success and failure
- Custom domains per environment
- Progressive delivery patterns such as canary or blue/green
- Policy-based quotas and concurrency controls

## 7. Architectural Approach

Launchpad will start as a modular monolith implemented in a single Spring Boot service named `control-plane`.

This service will own:

- API request handling
- authentication and authorization
- domain persistence
- deployment job creation
- background deployment execution
- rollout reconciliation
- audit event emission
- runtime and log query APIs

This decision is deliberate. A microservice split would increase deployment complexity, test overhead, and coordination cost without improving the MVP. The chosen architecture still allows strong module boundaries and future extraction if scale or team size requires it.

## 8. High-Level System Components

| Component | Responsibility |
| --- | --- |
| Control Plane API | Accepts user and CI requests, validates policy, persists release and deployment state |
| Deployment Worker | Pulls pending deployment jobs, renders Helm values, applies releases to Kubernetes |
| Reconciler | Polls Kubernetes rollout status and updates deployment lifecycle state |
| PostgreSQL | Source of truth for identity, projects, environments, releases, deployments, and audit events |
| Redis | Rate limiting, short-lived cache, and transient idempotency helpers |
| Helm Chart | Standard deployment contract for user workloads |
| Kubernetes Cluster | Hosts Launchpad and deployed workloads |
| GitHub Actions | Builds container images and triggers deployment through Launchpad |
| Prometheus / Grafana / Loki | Metrics, dashboards, and centralized logs |

## 9. Core Deployment Workflow

### 9.1 Release Creation

1. A repository workflow builds an image and pushes it to a trusted registry.
2. The workflow calls `POST /api/v1/deployments` with:
   - target project
   - target environment
   - image reference
   - git SHA
   - idempotency key
3. Launchpad validates:
   - caller identity or deploy token
   - allowed registry host
   - required environment configuration
   - image tag policy for the selected environment
4. Launchpad creates:
   - a `release` record
   - a `deployment` record in `PENDING`
   - a `deployment_job` record ready for worker pickup

### 9.2 Deployment Execution

1. The worker locks one job using database row locking.
2. The worker renders Helm values from project and environment configuration.
3. The worker ensures namespace and registry pull secret state.
4. The worker applies `helm upgrade --install`.
5. The deployment enters `APPLYING`.

### 9.3 Rollout Reconciliation

1. The reconciler checks Kubernetes rollout status on a schedule.
2. If pods become healthy, the deployment moves to `HEALTHY`.
3. If rollout exceeds timeout or fails probes, the deployment moves to `FAILED`.
4. Runtime metadata is persisted for the latest observed state.

## 10. Domain Model

| Entity | Purpose |
| --- | --- |
| `users` | Platform identities |
| `teams` | Multi-user ownership boundary |
| `team_members` | User membership and role assignment |
| `projects` | Logical application definition |
| `environments` | Deployment target such as `staging` or `production` |
| `project_secrets` | Encrypted environment variables |
| `registry_credentials` | Registry pull credentials where required |
| `deploy_tokens` | CI-only credentials used to trigger deployments |
| `releases` | Immutable version records tied to image and source revision |
| `deployments` | Runtime attempts to roll out a release into an environment |
| `deployment_jobs` | Worker queue state for pending and retried deployments |
| `runtime_snapshots` | Latest observed runtime health and endpoint metadata |
| `audit_events` | Append-only operational audit trail |

## 11. API Boundaries

Representative API surface:

- `POST /api/v1/auth/register`
- `POST /api/v1/auth/login`
- `POST /api/v1/auth/refresh`
- `GET /api/v1/me`
- `POST /api/v1/teams`
- `POST /api/v1/projects`
- `POST /api/v1/projects/{projectId}/environments`
- `PUT /api/v1/environments/{environmentId}/secrets`
- `POST /api/v1/projects/{projectId}/deploy-tokens`
- `POST /api/v1/deployments`
- `GET /api/v1/deployments/{deploymentId}`
- `POST /api/v1/deployments/{deploymentId}/rollback`
- `GET /api/v1/projects/{projectId}/runtime`
- `GET /api/v1/projects/{projectId}/logs`
- `GET /api/v1/projects/{projectId}/audit-events`

API design constraints:

- All responses include a request identifier.
- Deploy requests require an `Idempotency-Key` header.
- Validation errors use a structured format with machine-readable error codes.
- Log queries are constrained server-side; raw log backend queries are never passed through from clients.

## 12. Persistence and Queueing Strategy

PostgreSQL is the system of record. Schema changes are managed through Flyway migrations. Entities that are vulnerable to concurrent modification should use optimistic locking.

The initial deployment queue will also live in PostgreSQL rather than an external broker. Jobs will be claimed with `SELECT ... FOR UPDATE SKIP LOCKED`.

Reasons for this decision:

- Deployment throughput is low in the MVP.
- Consistency and operational simplicity matter more than raw queue throughput.
- It removes one infrastructure dependency from the initial release.
- It demonstrates disciplined scoping rather than premature distribution.

Redis remains useful for:

- rate limiting
- transient cache entries
- short-lived idempotency or request deduplication helpers

## 13. Kubernetes and Release Model

Each environment is deployed as a Helm release in a dedicated namespace using a deterministic naming convention:

- Namespace: `lp-<team>-<project>-<environment>`
- Release name: `app`
- Service name: `app`
- Hostname: `<project>-<environment>.<base-domain>`

The Helm chart must support:

- image repository and tag
- container port
- environment variables
- secret references
- replica count
- resource requests and limits
- readiness, liveness, and startup probes
- ingress configuration

Rollback will not call `kubectl rollout undo` directly. Instead, Launchpad will create a new deployment that points to the most recent healthy release. This keeps deployment history explicit and auditable.

## 14. Security Model

### 14.1 Authentication and Session Management

- Spring Security with access and refresh tokens
- password hashing with BCrypt or Argon2
- refresh tokens stored only as hashes

### 14.2 Secrets

- secret values encrypted at rest with AES-GCM
- master encryption key injected through environment or secret store
- secret values never returned in full after creation

### 14.3 Cluster Permissions

- Launchpad uses a dedicated service account
- RBAC is restricted to approved namespaces and resource types
- production writes require elevated team role checks

### 14.4 Deployment Safety Controls

- registry host allowlist
- mutable tags such as `:latest` rejected for production
- rollout blocked if mandatory health probe configuration is missing

## 15. Reliability and Failure Handling

The design must explicitly handle:

- Kubernetes API outages
- invalid or incomplete deployment configuration
- image pull failures
- health probe failures after rollout
- worker restarts while jobs are in progress
- repeated CI retries against the same deployment request

Required safeguards:

- idempotency key on deployment creation
- lock timeout and retry backoff for worker jobs
- rollout timeout with a clear failure state
- persisted failure reason and timestamp metadata
- append-only audit trail for critical actions

## 16. Observability

Launchpad itself must expose:

- health endpoint
- Prometheus metrics
- structured application logs
- distributed traces for deployment workflows

Minimum dashboards:

- deployment success rate
- deployment duration
- queue depth
- API latency
- pod restart count by project

Application workload logs should be centralized into Loki and exposed through Launchpad's log API using safe label-based filtering.

## 17. Environment Strategy

| Environment | Purpose | Minimum Shape |
| --- | --- | --- |
| Local | Fast developer feedback | Docker Compose + k3d or kind |
| Staging | End-to-end validation | Small k3s cluster, ingress, TLS, monitoring |
| Portfolio Production Demo | Recruiter-facing proof | Real domain, backups, dashboards, rollback runbook |

## 18. Alternatives Considered

### Alternative A: Microservices from the start

Rejected for MVP. It increases deployment and test complexity without adding meaningful value for a solo project.

### Alternative B: RabbitMQ or Kafka for deployment jobs

Deferred. Useful later, but unnecessary for the initial workload profile.

### Alternative C: Build source code inside Launchpad

Deferred. External CI keeps the first release smaller and still demonstrates a complete Git-driven deployment flow.

## 19. Acceptance Criteria

The design is considered successfully implemented when:

- a user can create a project and environment
- a CI pipeline can trigger a deployment with an immutable image reference
- Launchpad can deploy at least two sample applications to Kubernetes
- rollout state is visible through the API
- recent application logs are queryable
- a failed rollout can be rolled back to a previous healthy release
- high-risk actions are visible in audit history
- metrics and dashboards exist for the control plane

## 20. Interview and Demo Value

This project is strong for interviews because it lets the candidate discuss:

- backend architecture and domain modeling
- auth, RBAC, and secret handling
- queueing, idempotency, and failure recovery
- Kubernetes release automation
- observability and operational readiness
- practical trade-offs instead of over-engineering

That combination is much closer to real platform or backend work than a typical student CRUD portfolio.
