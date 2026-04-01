# Launchpad API Overview

Launchpad exposes a versioned REST API under `/api/v1`. The API is designed for two kinds of callers:

- Human users authenticating with JWT access tokens.
- CI/CD jobs authenticating with deploy tokens for deployment actions.

## Base URL

Use the deployment-specific host or local development host, then append `/api/v1`.

```text
https://launchpad.example.com/api/v1
http://localhost:8080/api/v1
```

## Authentication Modes

| Caller | Auth mechanism | Typical use |
| --- | --- | --- |
| User | `Authorization: Bearer <access_token>` | UI sessions, admin actions, browsing runtime data |
| CI/CD | `X-Launchpad-Deploy-Token: <token>` | Creating deployments from GitHub Actions or another pipeline |

Deploy tokens are limited to deployment-related actions. They do not grant full user-session privileges.

## Common Request Rules

- JSON request bodies use `Content-Type: application/json`.
- Mutable write operations on deployment creation should include `Idempotency-Key`.
- Identifiers are UUIDs.
- Timestamps are returned in ISO 8601 UTC format.
- Secret values are write-only after creation.

## Common Response Patterns

- Successful create requests usually return `201 Created`.
- Accepted asynchronous operations return `202 Accepted`.
- Read requests return `200 OK`.
- Validation failures return `422 Unprocessable Entity`.
- Authentication failures return `401 Unauthorized`.
- Authorization failures return `403 Forbidden`.

## Resource Areas

- [Authentication](./auth.md)
- [Projects and Environments](./projects.md)
- [Deployments, Runtime, and Rollback](./deployments.md)
- [Error Semantics](./errors.md)

## Typical Deployment Flow

1. Register or sign in as a user.
2. Create a team, project, and environment.
3. Generate a deploy token for CI.
4. Build and push an image from GitHub Actions.
5. Call the deployment API with the image reference and idempotency key.
6. Poll deployment status or query runtime data.
7. Roll back if the rollout fails.

## Conventions

### Pagination

List endpoints use cursor-style pagination when needed.

```json
{
  "items": [],
  "nextCursor": null
}
```

### IDs

All resource IDs are UUID strings.

```json
{
  "id": "7f7e2d8c-4b4b-4f48-9f91-2a8f88f0a7be"
}
```

### Request Correlation

Every response includes a request identifier in the response body so operational logs can be correlated with API calls.

Example:
```json
{
  "requestId": "5e91c9d3-9b05-4f93-b0f8-6a7b7d9e4c35",
  ...
}
```

### Timestamps

Use UTC timestamps.

```json
{
  "createdAt": "2026-04-01T12:30:00Z"
}
```
