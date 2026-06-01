# Monitoring

Prometheus + Grafana stack for the CloudKitchen platform, deployed via the
[`kube-prometheus-stack`](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)
Helm chart into the `monitoring` namespace.

## Contents

| Path                         | Purpose |
|------------------------------|---------|
| `prometheus-values.yaml`     | Helm values for kube-prometheus-stack (Prometheus, Alertmanager, Grafana, node-exporter, kube-state-metrics) |
| `servicemonitor.yaml`        | `ServiceMonitor` selecting the CloudKitchen services |
| `grafana/dashboards/`        | JSON dashboards + loading instructions |

## How metrics are collected

Every Go microservice (`auth`, `user`, `restaurant`, `menu`, `order`,
`payment`, `delivery`, `notification`) listens on port **8080** and exposes:

- `GET /metrics`  — Prometheus exposition format
- `GET /healthz`  — liveness
- `GET /readyz`   — readiness

Two discovery mechanisms are wired up (either is sufficient):

1. **ServiceMonitor (preferred).** `servicemonitor.yaml` selects Services
   labeled `app.kubernetes.io/part-of: cloudkitchen` in the `cloudkitchen`
   namespace and scrapes their named `http` port at `/metrics`.

2. **Annotation-based fallback.** `prometheus-values.yaml` adds a scrape job
   that keeps any pod annotated with:

   ```yaml
   prometheus.io/scrape: "true"
   prometheus.io/port:   "8080"
   prometheus.io/path:   "/metrics"
   ```

## Install

```sh
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm upgrade --install kube-prom-stack \
  prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace \
  -f monitoring/prometheus-values.yaml

kubectl apply -f monitoring/servicemonitor.yaml
```

Then load dashboards as described in `grafana/dashboards/README.md`.

## Access (local)

```sh
# Grafana
./scripts/port-forward-grafana.sh        # -> http://localhost:3001 (admin / changeme-use-a-secret)

# Prometheus
kubectl -n monitoring port-forward svc/kube-prom-stack-prometheus 9090:9090
```

> The Grafana admin password in `prometheus-values.yaml` is a placeholder.
> In real deployments source it from a Secret / External Secrets, never commit it.
