# ADR-002: PostgreSQL-Backed Deployment Job Queue

## Status

Accepted

## Context

Launchpad needs a reliable queue for deployment jobs. The expected MVP workload is low frequency and operational simplicity is more important than maximum throughput.

Introducing RabbitMQ or Kafka at this stage would add infrastructure and deployment overhead without solving an immediate scale problem.

## Decision

Store deployment jobs in PostgreSQL and claim work using row locking with `FOR UPDATE SKIP LOCKED`.

The queue implementation will support:

- retries with backoff
- lock expiry
- failure metadata
- deterministic status transitions

## Consequences

### Positive

- fewer moving parts in the initial release
- simpler local development
- stronger transactional consistency with deployment records

### Negative

- not suitable for very high throughput
- requires care around polling intervals and lock contention

### Follow-Up

If workload or team size grows, the queue can be extracted behind an adapter and migrated to a dedicated broker.
