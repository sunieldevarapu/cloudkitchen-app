# Grafana Dashboards

JSON dashboard definitions for the CloudKitchen platform. All dashboards are
namespaced to `cloudkitchen` (via a `$namespace` template variable) and assume
Prometheus is the default datasource.

| File                     | UID                    | What it shows |
|--------------------------|------------------------|---------------|
| `cpu-usage.json`         | `ck-cpu-usage`         | CPU cores per pod via `rate(container_cpu_usage_seconds_total[5m])`, plus usage vs. requests |
| `memory-usage.json`      | `ck-memory-usage`      | `container_memory_working_set_bytes` per pod, plus usage vs. limits |
| `pod-health.json`        | `ck-pod-health`        | `up` targets, container restarts, pods not ready |
| `http-request-rate.json` | `ck-http-request-rate` | `rate(http_requests_total[5m])` total / by service / by method+code |
| `http-error-rate.json`   | `ck-http-error-rate`   | 5xx ratio and rate using `http_requests_total{code=~"5.."}` |

## Loading the dashboards

### Option A — Grafana sidecar (recommended)

The kube-prometheus-stack Grafana sidecar auto-imports any ConfigMap labeled
`grafana_dashboard=1`. Create one ConfigMap per dashboard:

```sh
for f in monitoring/grafana/dashboards/*.json; do
  name="ck-$(basename "$f" .json)"
  kubectl -n monitoring create configmap "$name" \
    --from-file="$f" --dry-run=client -o yaml \
  | kubectl label -f - --local -o yaml grafana_dashboard=1 \
  | kubectl apply -f -
done
```

### Option B — Manual import

In the Grafana UI: **Dashboards -> New -> Import -> Upload JSON file**.

## Notes

- Metric `http_requests_total` is expected to carry the labels `service`,
  `method`, and `code` (3-digit status code). Adjust the queries if your
  instrumentation uses different label names (e.g. `status` instead of `code`).
- `container_cpu_usage_seconds_total` / `container_memory_working_set_bytes`
  come from cAdvisor (kubelet), scraped by kube-prometheus-stack out of the box.
- `kube_pod_*` series come from kube-state-metrics.
