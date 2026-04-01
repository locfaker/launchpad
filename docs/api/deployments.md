# Deployments API

Deployment endpoints accept immutable image references and create auditable rollout records. Deployments are asynchronous: a request creates a release and a job, then the worker applies the release to Kubernetes.

## POST `/api/v1/deployments`

Create a deployment request.

**Headers**

```http
Authorization: Bearer <access_token>
Idempotency-Key: 8e8f6c7c-1f07-4c5a-92d5-67a8d61f8d74
```

For CI-driven deployment flows, a deploy token may be used instead of a user access token:

```http
X-Launchpad-Deploy-Token: lp_dt_eyJhbGciOiJIUzI1NiIs...
Idempotency-Key: 8e8f6c7c-1f07-4c5a-92d5-67a8d61f8d74
```

**Request**

```json
{
  "projectId": "ae1d2d39-5bb5-4f8f-8a45-9d3d1f1d0f87",
  "environmentId": "d2d1d1f7-aef4-45b1-8f48-2ac4c3a9d0d2",
  "image": "ghcr.io/acme/billing-service:sha-123abc",
  "gitSha": "123abc",
  "triggerSource": "github_actions"
}
```

**Responses**

- `202 Accepted`: Deployment accepted and queued.
- `400 Bad Request`: Missing idempotency key or invalid payload shape.
- `401 Unauthorized`: Missing or invalid credentials.
- `403 Forbidden`: Caller cannot deploy to the target environment.
- `409 Conflict`: Duplicate request or incompatible deployment in progress.
- `422 Unprocessable Entity`: Image policy violation or validation failure.

**Response**

```json
{
  "deployment": {
    "id": "2a743d8c-47a4-4e5d-90f9-93dca04a1a10",
    "releaseId": "3b7b0d63-4f5a-40f4-93d0-0e1b3c3b1a13",
    "projectId": "ae1d2d39-5bb5-4f8f-8a45-9d3d1f1d0f87",
    "environmentId": "d2d1d1f7-aef4-45b1-8f48-2ac4c3a9d0d2",
    "status": "PENDING",
    "image": "ghcr.io/acme/billing-service:sha-123abc",
    "createdAt": "2026-04-01T12:55:00Z"
  }
}
```

## GET `/api/v1/deployments/{deploymentId}`

Fetch the current deployment record and status.

**Responses**

- `200 OK`: Deployment found.
- `404 Not Found`: Deployment does not exist or caller has no access.

**Response**

```json
{
  "deployment": {
    "id": "2a743d8c-47a4-4e5d-90f9-93dca04a1a10",
    "status": "HEALTHY",
    "startedAt": "2026-04-01T12:55:10Z",
    "finishedAt": "2026-04-01T12:56:02Z",
    "failureReason": null,
    "runtimeSnapshot": {
      "readyReplicas": 2,
      "ingressUrl": "https://billing-service-production.launchpad.example.com"
    }
  }
}
```

## POST `/api/v1/deployments/{deploymentId}/rollback`

Create a new deployment that rolls back to the most recent healthy release.

**Headers**

```http
Authorization: Bearer <access_token>
```

**Responses**

- `202 Accepted`: Rollback accepted and queued.
- `403 Forbidden`: Caller cannot roll back this environment.
- `404 Not Found`: Deployment not found.
- `409 Conflict`: No healthy release exists to roll back to.

**Response**

```json
{
  "rollbackDeploymentId": "6d9c5f31-2d12-4a1d-b79a-4b6fe1e7d0e2",
  "sourceDeploymentId": "2a743d8c-47a4-4e5d-90f9-93dca04a1a10",
  "status": "PENDING"
}
```

## GET `/api/v1/projects/{projectId}/runtime`

Return the latest runtime snapshot for all environments in a project.

**Response**

```json
{
  "items": [
    {
      "environmentId": "d2d1d1f7-aef4-45b1-8f48-2ac4c3a9d0d2",
      "deploymentId": "2a743d8c-47a4-4e5d-90f9-93dca04a1a10",
      "status": "HEALTHY",
      "readyReplicas": 2,
      "ingressUrl": "https://billing-service-production.launchpad.example.com",
      "observedAt": "2026-04-01T12:56:05Z"
    }
  ]
}
```

## GET `/api/v1/projects/{projectId}/logs`

Fetch recent application logs for a project.

**Query parameters**

| Name | Type | Description |
| --- | --- | --- |
| `from` | string | Optional time window such as `15m` or `1h` |
| `limit` | integer | Maximum log lines to return |

**Response**

```json
{
  "items": [
    {
      "timestamp": "2026-04-01T12:56:00Z",
      "level": "INFO",
      "message": "Application started successfully",
      "pod": "app-7c9d7f4d4b-2j9p8"
    }
  ]
}
```

## Deployment States

| State | Meaning |
| --- | --- |
| `PENDING` | Deployment accepted and waiting for worker pickup |
| `APPLYING` | Helm release is being applied to the cluster |
| `HEALTH_CHECKING` | Rollout has been applied and health is being observed |
| `HEALTHY` | Deployment is healthy and serving traffic |
| `FAILED` | Deployment failed health or rollout checks |
| `ROLLED_BACK` | Deployment was superseded by a rollback request |

## Deployment Notes

- Use immutable tags or digests for release images.
- Treat deployment creation as asynchronous even when the API returns quickly.
- Always check `Idempotency-Key` handling in CI retries.
- Do not assume the deploy response means the workload is healthy; poll status or runtime endpoints.
