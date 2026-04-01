# Staging Bootstrap Runbook

## Purpose

Use this runbook to provision a staging environment for Launchpad and its sample workloads.

## Prerequisites

- A Kubernetes cluster with admin access
- `kubectl`
- `helm`
- A container registry such as GHCR
- DNS control for the staging domain
- TLS issuer access through `cert-manager`

## Desired State

Staging should include:

- Launchpad control plane
- Ingress controller
- cert-manager
- PostgreSQL
- Redis
- Prometheus
- Grafana
- Loki

## Procedure

1. Create or confirm the target namespace set.
2. Install the ingress controller.
3. Install cert-manager and configure the issuer.
4. Install monitoring and logging stacks.
5. Deploy Launchpad with staging configuration.
6. Verify ingress, TLS, metrics, and application health.

## Exact Steps

```bash
kubectl apply -f infra/k8s/bootstrap/namespaces.yaml
```

Install the ingress controller and cert-manager using the cluster standard for your environment.

Apply observability values:

```bash
helm upgrade --install prometheus-stack prometheus-community/kube-prometheus-stack \
  -n monitoring -f infra/k8s/bootstrap/prometheus-values.yaml
```

```bash
helm upgrade --install loki grafana/loki-stack \
  -n monitoring -f infra/k8s/bootstrap/loki-values.yaml
```

```bash
helm upgrade --install grafana grafana/grafana \
  -n monitoring -f infra/k8s/bootstrap/grafana-values.yaml
```

Deploy Launchpad using the staging release configuration for the control plane chart.

## Verification

- `kubectl get pods -A` shows all system pods ready
- `kubectl get ingress -A` shows Launchpad and sample app ingress objects
- `curl https://<launchpad-staging-domain>/actuator/health` returns `UP`
- Grafana can query Launchpad metrics
- Loki returns application logs for a deployed sample workload

## Recovery

If the staging bootstrap fails:

- Check ingress class names and namespace references.
- Check DNS resolution for the staging domain.
- Check cert-manager issuer status.
- Check pod events in the `monitoring` and `launchpad` namespaces.

If TLS does not become ready:

- Verify the ACME challenge flow.
- Verify the DNS record points to the ingress controller address.
- Reconcile the certificate after DNS propagation completes.

## Troubleshooting

### Ingress returns 404

Likely causes:

- wrong host rule
- wrong ingress class
- service selector mismatch

### Certificate stays pending

Likely causes:

- DNS record not propagated
- issuer misconfigured
- rate limits at the certificate authority

### Metrics do not appear in Grafana

Likely causes:

- wrong Prometheus scrape target
- service monitor not enabled
- Launchpad metrics endpoint blocked by auth or network policy

