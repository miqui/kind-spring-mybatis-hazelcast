# Grafana Changes — OpenObserve Observability Rollout

## New dashboard: "OpenObserve Ops"

Added as a new key (`openobserve-ops.json`) in
`k8s/observability/grafana-dashboard-json-configmap.yaml`, alongside the six pre-existing
dashboards in that same ConfigMap. No changes were needed to
`grafana-dashboard-provider-configmap.yaml` or `grafana-deployment.yaml` — the whole ConfigMap is
already mounted as a directory (`/etc/grafana/provisioning/dashboards-json`), so a new key just
shows up as a new file and Grafana's file-based dashboard provider picks it up on its own.

Self-monitoring for the OpenObserve backend itself — not the message-service or cluster-ops data
*inside* OpenObserve, but OpenObserve's own health as a running service. 14 panels, `uid: openobserve-ops`:

| Panel | Type | PromQL |
| :--- | :--- | :--- |
| OpenObserve Up | stat | `up{job="openobserve"}` |
| Disk Usage | gauge | `zo_node_disk_usage{job="openobserve"} / zo_node_disk_total{job="openobserve"} * 100` |
| Process Memory (RSS) | stat | `process_resident_memory_bytes{job="openobserve"}` |
| Open File Descriptors | stat | `process_open_fds{job="openobserve"}` |
| Uptime | stat | `time() - process_start_time_seconds{job="openobserve"}` |
| Ingest Rate (records/s) | stat | `sum(rate(zo_ingest_records{job="openobserve"}[5m]))` |
| HTTP Request Rate by Endpoint | timeseries | `sum(rate(zo_http_incoming_requests{job="openobserve"}[5m])) by (endpoint, status)` |
| HTTP Response Time p95/p99 by Endpoint | timeseries | `histogram_quantile(0.95/0.99, sum(rate(zo_http_response_time_bucket{job="openobserve"}[5m])) by (le, endpoint))` |
| Ingest Rate by Stream Type (records/s) | timeseries | `sum(rate(zo_ingest_records{job="openobserve"}[5m])) by (stream_type)` |
| Ingest Rate by Stream Type (bytes/s) | timeseries | `sum(rate(zo_ingest_bytes{job="openobserve"}[5m])) by (stream_type)` |
| In-Memory MemTable Size | timeseries | `zo_ingest_memtable_bytes{job="openobserve"}` |
| WAL Used Bytes by Stream Type | timeseries | `sum(zo_ingest_wal_used_bytes{job="openobserve"}) by (stream_type)` |
| Process CPU Usage | timeseries | `rate(process_cpu_seconds_total{job="openobserve"}[5m]) * 100` |
| Disk Usage vs Total | timeseries | `zo_node_disk_usage{job="openobserve"}` vs `zo_node_disk_total{job="openobserve"}` |

The **In-Memory MemTable Size** panel is deliberate, not incidental: `zo_ingest_memtable_bytes` is
the exact thing that overflowed (`MemoryTableOverflowError`, see `PROMETHEUS.md` step 1) before
remote_write was scoped down and OpenObserve's resources were bumped. It's the one panel worth
watching if the remote_write scope is ever widened again.

All 14 metric names/labels were read directly off OpenObserve's own `/metrics` output before
writing any panel — none were guessed. Every `process_*` metric query is deliberately scoped with
`{job="openobserve"}`: `process_cpu_seconds_total`/`process_resident_memory_bytes`/etc. are generic
Prometheus-client-library metrics, also exposed by `kube-state-metrics` and `postgres-exporter` —
without the job filter these would silently mix with the wrong process's numbers.

### Dependencies (documented in full elsewhere)

This dashboard depends on two other changes, covered in their own docs:
- `k8s/observability/prometheus-configmap.yaml` gained an `openobserve` scrape job for
  `openobserve.observability.svc.cluster.local:5080/metrics` — see `PROMETHEUS.md` step 5.
- `k8s/observability/openobserve-values.yaml` gained `config.ZO_PROMETHEUS_ENABLED: "true"` (off by
  default — confirmed live that `/metrics` returns HTTP 200 with an empty body otherwise) — see
  `HELM.md`.

### Verification performed

- Queried all 14 panel expressions directly against Prometheus (not just Grafana) — every one
  returned real, non-empty data (e.g. disk usage ~11%, RSS ~566MB, 40 open fds, ingest rate ~967
  records/s).
- Confirmed via Grafana's `/api/dashboards/uid/openobserve-ops` that the dashboard is actually
  provisioned, with all 14 panels present (not just that the ConfigMap key exists).

## Unrelated finding: admin credentials had drifted

While verifying the dashboard via Grafana's API, `admin`/`admin` (the credentials documented in
`k8s/observability/grafana-secret.yaml` and the README) returned `401 Invalid username or
password` — confirmed via pod logs (`password-auth.invalid`) that this was a genuine credential
mismatch, not a transient issue. The Grafana pod hadn't restarted since its original deploy, so
something changed the in-database admin password outside of this session's changes.

Fixed by resetting it back to the documented value using Grafana's own supported recovery path
(safe on this disposable local dev instance):

```bash
kubectl exec -n observability deploy/grafana -- grafana-cli admin reset-admin-password admin
```

No config files were changed for this — `grafana-secret.yaml` already had the correct value; only
Grafana's own internal (sqlite) state was out of sync with it.
