# Service Level Objectives

## Scope

These SLOs are lightweight targets for a small internal deployment platform. They exist to frame operational expectations rather than to impose enterprise-grade controls.

## Proposed Objectives

| Capability | Objective | Measurement |
| --- | --- | --- |
| Control plane availability | 99.5% monthly in staging or demo production | Health endpoint and ingress reachability |
| Deployment API latency | p95 under 500 ms for accepted requests | HTTP metrics |
| Deployment processing start time | 90% of jobs start within 30 seconds | Job creation to worker pickup |
| Rollback execution start time | 90% of rollback jobs start within 30 seconds | Rollback request to job pickup |

## Error Budget Guidance

If availability or latency drifts outside target:

- stop adding optional scope
- prioritize reliability fixes
- tighten observability gaps
- document the incident and response

## Notes

These targets are intentionally modest. The project is a production-style portfolio system, not a large-scale hosted platform.
