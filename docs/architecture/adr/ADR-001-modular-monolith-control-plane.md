# ADR-001: Modular Monolith Control Plane

## Status

Accepted

## Context

Launchpad needs authentication, project configuration, deployment orchestration, runtime reconciliation, and audit logging. The system is being built by a single engineer as a portfolio-grade production project.

A microservice split would require extra deployment topology, service-to-service authentication, distributed tracing from day one, and additional operational overhead.

## Decision

Implement Launchpad as a modular monolith in a single Spring Boot service named `control-plane`.

The service will be divided into explicit modules:

- auth
- team
- project
- deployment
- audit
- common infrastructure

## Consequences

### Positive

- lower operational complexity
- simpler local development and testing
- easier to reason about transactions and module boundaries
- realistic for a solo engineer to complete

### Negative

- no independent scaling by subsystem
- long-term extraction requires discipline around module coupling

### Follow-Up

Keep Kubernetes, logging, and queue adapters behind interfaces so worker extraction remains possible later.
