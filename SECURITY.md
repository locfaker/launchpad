# Security Policy

## Scope

Launchpad handles deploy credentials, environment secrets, and cluster automation. Even as a portfolio project, the repository should follow production-grade security expectations.

## Supported Security Controls

The target design includes:

- hashed user passwords
- hashed refresh tokens
- hashed deploy tokens
- encrypted environment secrets at rest
- registry host allowlists
- role-based access control
- audit logging for high-risk actions
- production image tag restrictions

## Reporting a Security Issue

Do not open a public issue for a suspected vulnerability that could expose secrets, authentication state, or cluster access.

Instead:

1. Record the issue privately.
2. Describe the impact, affected area, and reproduction steps.
3. Note whether the issue affects:
   - authentication
   - authorization
   - secret storage
   - deployment safety
   - cluster permissions
4. Propose containment steps if exploitation is possible.

## Sensitive Areas

The following areas require extra scrutiny during implementation and review:

- token generation and storage
- secret encryption and key management
- Kubernetes execution adapters
- Helm value rendering
- audit event completeness
- log query filtering

## Secret Handling Rules

- Never commit real secrets.
- Never log raw secrets or tokens.
- Never return stored secret values in full after write operations.
- Rotate any demo credentials used in shared environments.
