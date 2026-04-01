# Contributing

## Purpose

This repository is structured as a production-style portfolio project. Contributions should improve clarity, correctness, or operational realism. Changes that add scope without improving the core deployment-control-plane story should be avoided.

## Workflow

1. Start from an issue, plan item, or explicit task.
2. Keep changes scoped to one concern.
3. Update documentation when behavior, architecture, or operational procedures change.
4. Prefer small commits with clear intent.
5. Run relevant verification before opening a review.

## Branching

- Use short-lived feature branches.
- Keep branch names descriptive, for example `feat/deploy-worker` or `docs/api-reference`.
- Rebase or merge from the main branch regularly if the branch lives longer than one day.

## Coding Expectations

- Use Java 21 and Spring Boot conventions consistently.
- Keep module boundaries explicit.
- Do not let controllers reach directly into infrastructure adapters.
- Add comments only when they explain intent or non-obvious trade-offs.
- Do not store secrets, raw tokens, or environment-specific credentials in the repository.

## Testing Expectations

- Unit tests for domain and service logic.
- Integration tests for persistence, security, and deployment orchestration boundaries.
- End-to-end verification for critical flows such as deploy, failed rollout, and rollback.
- Documentation changes should be reviewed for consistency with the current design and implementation plan.

## Documentation Requirements

Update the relevant docs when a change affects:

- architecture or module boundaries
- API contracts
- configuration or environment variables
- operational procedures
- security controls

At minimum, update the corresponding file under `docs/`.

## Pull Request Checklist

- Scope is clear and limited.
- Tests or verification steps are included.
- Documentation is updated when required.
- Security and operational impact are noted for non-trivial changes.
- Trade-offs are stated for architectural decisions.
