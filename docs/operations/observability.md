# Observability

## Objective

Launchpad must provide enough telemetry to answer four questions quickly:

1. Is the control plane healthy?
2. Are deployments succeeding or failing?
3. What is happening in the worker and reconciliation loop?
4. What changed immediately before an incident?

## Signals

### Metrics

Minimum metrics:

- deployment count by status
- deployment duration
- queue depth
- queue retry count
- API latency by route
- authentication failures
- pod restart count by project

### Logs

Launchpad application logs should be structured JSON and include:

- timestamp
- level
- request id
- actor id when available
- deployment id when relevant
- trace id when available

Application workload logs should be centralized in Loki and queried through safe, server-generated filters.

### Traces

The following flows should be traced:

- login and token refresh
- deployment creation
- deployment execution
- rollout reconciliation
- rollback creation

## Dashboards

Minimum dashboards:

- control plane overview
- deployment pipeline health
- queue and worker health
- application runtime health by project

## Alerting Guidance

The initial project does not need a full paging system, but it should define alert candidates:

- deployment failure rate above threshold
- queue depth growing continuously
- control plane health endpoint unavailable
- repeated authentication failures
- abnormal pod restart rate
