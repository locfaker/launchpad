# Projects API

This section covers team, project, environment, secret, deploy-token, and audit-related endpoints.

## POST `/api/v1/teams`

Create a new team.

**Headers**

```http
Authorization: Bearer <access_token>
```

**Request**

```json
{
  "name": "Platform Team",
  "slug": "platform-team"
}
```

**Responses**

- `201 Created`: Team created.
- `409 Conflict`: Team slug already exists.
- `422 Unprocessable Entity`: Invalid slug or name.

## POST `/api/v1/projects`

Create a project under a team.

**Request**

```json
{
  "teamId": "b1f5dfe7-6c73-41d1-9a04-3bc5a8d3d101",
  "name": "billing-service",
  "slug": "billing-service",
  "runtimePort": 8080,
  "healthcheckPath": "/actuator/health"
}
```

**Responses**

- `201 Created`: Project created.
- `403 Forbidden`: Caller is not a team admin for the target team.
- `409 Conflict`: Project slug already exists within the team.
- `422 Unprocessable Entity`: Invalid runtime config.

**Response**

```json
{
  "project": {
    "id": "ae1d2d39-5bb5-4f8f-8a45-9d3d1f1d0f87",
    "teamId": "b1f5dfe7-6c73-41d1-9a04-3bc5a8d3d101",
    "name": "billing-service",
    "slug": "billing-service",
    "runtimePort": 8080,
    "healthcheckPath": "/actuator/health",
    "createdAt": "2026-04-01T12:45:00Z"
  }
}
```

## POST `/api/v1/projects/{projectId}/environments`

Create an environment for a project.

**Request**

```json
{
  "name": "production",
  "namespace": "lp-platform-team-billing-service-production",
  "baseDomain": "launchpad.example.com",
  "replicas": 2
}
```

**Responses**

- `201 Created`: Environment created.
- `403 Forbidden`: Caller lacks permission.
- `409 Conflict`: Environment name already exists for the project.
- `422 Unprocessable Entity`: Invalid namespace, domain, or replica count.

**Response**

```json
{
  "environment": {
    "id": "d2d1d1f7-aef4-45b1-8f48-2ac4c3a9d0d2",
    "projectId": "ae1d2d39-5bb5-4f8f-8a45-9d3d1f1d0f87",
    "name": "production",
    "namespace": "lp-platform-team-billing-service-production",
    "baseDomain": "launchpad.example.com",
    "replicas": 2,
    "createdAt": "2026-04-01T12:46:00Z"
  }
}
```

## PUT `/api/v1/environments/{environmentId}/secrets`

Set or rotate environment secrets.

**Request**

```json
{
  "secrets": {
    "DATABASE_URL": "postgres://user:pass@host:5432/app",
    "REDIS_URL": "redis://host:6379/0"
  }
}
```

**Responses**

- `200 OK`: Secrets updated.
- `403 Forbidden`: Caller cannot update this environment.
- `422 Unprocessable Entity`: Secret key format or policy violation.

**Response**

```json
{
  "environmentId": "d2d1d1f7-aef4-45b1-8f48-2ac4c3a9d0d2",
  "updatedKeys": ["DATABASE_URL", "REDIS_URL"],
  "updatedAt": "2026-04-01T12:47:00Z"
}
```

Secrets are write-only. The API does not return secret values after they are stored.

## POST `/api/v1/projects/{projectId}/deploy-tokens`

Create a deploy token for CI/CD.

**Request**

```json
{
  "name": "github-actions-prod",
  "environmentId": "d2d1d1f7-aef4-45b1-8f48-2ac4c3a9d0d2",
  "expiresAt": "2026-10-01T00:00:00Z"
}
```

**Responses**

- `201 Created`: Token created and shown once.
- `403 Forbidden`: Caller lacks permission.
- `422 Unprocessable Entity`: Invalid expiry or environment.

**Response**

```json
{
  "deployToken": {
    "id": "3fb23b8e-5af8-4e14-9b35-5a8d6c0c9a74",
    "name": "github-actions-prod",
    "token": "lp_dt_eyJhbGciOiJIUzI1NiIs...",
    "environmentId": "d2d1d1f7-aef4-45b1-8f48-2ac4c3a9d0d2",
    "expiresAt": "2026-10-01T00:00:00Z"
  }
}
```

The raw token is returned only once at creation time.

## GET `/api/v1/projects/{projectId}/audit-events`

Return audit events for a project.

**Headers**

```http
Authorization: Bearer <access_token>
```

**Responses**

- `200 OK`: Audit events returned.
- `403 Forbidden`: Caller lacks access.

**Response**

```json
{
  "items": [
    {
      "id": "f8f3ad7f-5e5d-4ee2-9c21-f3f5b42e1000",
      "action": "PROJECT_SECRET_UPDATED",
      "actorId": "7f7e2d8c-4b4b-4f48-9f91-2a8f88f0a7be",
      "resourceType": "environment",
      "resourceId": "d2d1d1f7-aef4-45b1-8f48-2ac4c3a9d0d2",
      "createdAt": "2026-04-01T12:47:00Z"
    }
  ],
  "nextCursor": null
}
```

## Project Notes

- Project slugs should be stable and human-readable.
- Environment names are typically `staging` and `production`.
- Production secret changes and deploy-token rotation should be restricted to elevated roles.
