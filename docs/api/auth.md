# Authentication API

Authentication is JWT-based for user sessions. A successful login returns an access token and refresh token. Access tokens are short-lived and used in the `Authorization` header.

## POST `/api/v1/auth/register`

Create the first user account or invite-free account bootstrap in a demo environment.

**Request**

```json
{
  "email": "admin@launchpad.dev",
  "password": "P@ssw0rd123!"
}
```

**Responses**

- `201 Created`: User created successfully.
- `409 Conflict`: Email already exists.
- `422 Unprocessable Entity`: Invalid email or password policy violation.

**Response**

```json
{
  "user": {
    "id": "7f7e2d8c-4b4b-4f48-9f91-2a8f88f0a7be",
    "email": "admin@launchpad.dev",
    "createdAt": "2026-04-01T12:30:00Z"
  },
  "accessToken": "eyJhbGciOiJIUzI1NiIs...",
  "refreshToken": "eyJhbGciOiJIUzI1NiIs..."
}
```

## POST `/api/v1/auth/login`

Exchange credentials for tokens.

**Request**

```json
{
  "email": "admin@launchpad.dev",
  "password": "P@ssw0rd123!"
}
```

**Responses**

- `200 OK`: Tokens issued.
- `401 Unauthorized`: Invalid credentials.
- `429 Too Many Requests`: Too many failed attempts.

**Response**

```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiIs...",
  "refreshToken": "eyJhbGciOiJIUzI1NiIs...",
  "expiresInSeconds": 900
}
```

## POST `/api/v1/auth/refresh`

Rotate a refresh token and issue a new access token.

**Request**

```json
{
  "refreshToken": "eyJhbGciOiJIUzI1NiIs..."
}
```

**Responses**

- `200 OK`: New token pair issued.
- `401 Unauthorized`: Refresh token expired, revoked, or invalid.

**Response**

```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiIs...",
  "refreshToken": "eyJhbGciOiJIUzI1NiIs...",
  "expiresInSeconds": 900
}
```

## GET `/api/v1/me`

Return the current authenticated user and role context.

**Headers**

```http
Authorization: Bearer <access_token>
```

**Responses**

- `200 OK`: Current user data.
- `401 Unauthorized`: Missing or invalid token.

**Response**

```json
{
  "user": {
    "id": "7f7e2d8c-4b4b-4f48-9f91-2a8f88f0a7be",
    "email": "admin@launchpad.dev"
  },
  "memberships": [
    {
      "teamId": "b1f5dfe7-6c73-41d1-9a04-3bc5a8d3d101",
      "role": "TEAM_ADMIN"
    }
  ]
}
```

## Auth Notes

- Passwords are never returned.
- Refresh tokens are stored server-side as hashes, not plaintext.
- Login attempts should be audited.
- Human users should use JWT access tokens, not deploy tokens, for management actions.
