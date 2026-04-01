# Error Semantics

Launchpad uses a structured JSON error envelope across the API. The shape is stable so clients can inspect both HTTP status and machine-readable error codes.

## Error Envelope

```json
{
  "requestId": "5e91c9d3-9b05-4f93-b0f8-6a7b7d9e4c35",
  "status": 422,
  "error": "Unprocessable Entity",
  "code": "VALIDATION_ERROR",
  "message": "One or more fields are invalid.",
  "details": [
    {
      "field": "image",
      "issue": "Only immutable tags are allowed for production deployments."
    }
  ]
}
```

## Status Code Mapping

| HTTP Status | Typical Use |
| --- | --- |
| `400 Bad Request` | Missing required header, malformed JSON, invalid request shape |
| `401 Unauthorized` | Missing, expired, or invalid credentials |
| `403 Forbidden` | Authenticated but not allowed for the requested resource |
| `404 Not Found` | Resource does not exist or is not visible to the caller |
| `409 Conflict` | Duplicate resource, conflicting deployment, or idempotency collision |
| `422 Unprocessable Entity` | Validation failure or policy rejection |
| `429 Too Many Requests` | Rate limit exceeded |
| `500 Internal Server Error` | Unexpected server failure |
| `503 Service Unavailable` | Dependency outage or maintenance window |

## Common Error Codes

| Code | Meaning |
| --- | --- |
| `AUTH_INVALID_CREDENTIALS` | Email or password was incorrect |
| `AUTH_TOKEN_EXPIRED` | Access token or refresh token is expired |
| `AUTH_TOKEN_INVALID` | Token is malformed, revoked, or unverifiable |
| `AUTH_FORBIDDEN` | Caller lacks the required role or membership |
| `RESOURCE_NOT_FOUND` | The requested resource was not found |
| `VALIDATION_ERROR` | Payload or policy validation failed |
| `IDEMPOTENCY_CONFLICT` | The same idempotency key was reused with a conflicting request |
| `DEPLOYMENT_CONFLICT` | Another deployment blocks the requested action |
| `RATE_LIMITED` | Too many requests in the active window |
| `PROVIDER_ERROR` | A downstream dependency failed |
| `DEPLOYMENT_FAILED` | A rollout failed health or readiness checks |

## Example: Validation Failure

```http
HTTP/1.1 422 Unprocessable Entity
Content-Type: application/json
```

```json
{
  "requestId": "2d4d6d4f-5f5a-44f5-8434-bf0cf7d5cb12",
  "status": 422,
  "error": "Unprocessable Entity",
  "code": "VALIDATION_ERROR",
  "message": "One or more fields are invalid.",
  "details": [
    {
      "field": "runtimePort",
      "issue": "Must be between 1 and 65535."
    }
  ]
}
```

## Example: Unauthorized

```http
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Bearer
```

```json
{
  "requestId": "4f0b3b9f-f62a-4fb3-8c3d-1f6e99d7d3e0",
  "status": 401,
  "error": "Unauthorized",
  "code": "AUTH_TOKEN_INVALID",
  "message": "Authentication is required."
}
```

## Example: Rate Limited

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 60
```

```json
{
  "requestId": "7c9e9bb0-8d2d-4c0d-8a10-5c0f5dc1f4f0",
  "status": 429,
  "error": "Too Many Requests",
  "code": "RATE_LIMITED",
  "message": "Too many requests. Try again later."
}
```

## Error Handling Guidance

- Treat `422` as a caller-correctable error.
- Treat `409` as a state conflict that may require a refreshed read.
- Treat `401` by refreshing or re-authenticating.
- Treat `5xx` as retryable only when the request is safe to repeat and idempotent.
- Always log the `requestId` when escalating an issue.
