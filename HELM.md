# Helm Commands — OpenObserve Observability Rollout

`helm` commands used while adding OpenObserve (and its supporting configuration) as an
observability backend for the message REST API, grouped by phase.

## Chart discovery (finding the right chart/repo before using it)

```bash
helm repo list
helm search hub openobserve
helm search hub openobserve --output json
helm repo add openobserve https://charts.openobserve.ai
helm repo update openobserve
helm show values openobserve/openobserve-standalone
helm show chart openobserve/openobserve-standalone
helm pull openobserve/openobserve-standalone --untar --untardir <scratch-dir>
```

## From `deploy-kind.sh` (the automated deployment)

```bash
# idempotent repo add - only runs if not already registered
if ! helm repo list | grep -q '^openobserve[[:space:]]'; then
  helm repo add openobserve https://charts.openobserve.ai
fi
helm repo update openobserve
helm upgrade --install openobserve openobserve/openobserve-standalone \
  --version 0.92.2 \
  --namespace observability \
  -f k8s/observability/openobserve-values.yaml \
  --wait --timeout 180s
```

## Manual verification

```bash
helm list -n observability
```

## Manual re-applies after values changes

Each of the following config changes to `k8s/observability/openobserve-values.yaml` was applied
live with the same `helm upgrade --install` command (the chart's `checksum/config` annotation on
the StatefulSet's pod template forces an automatic rollout on each upgrade, so no manual restart
was needed):

- Retention cap (`config.ZO_COMPACT_DATA_RETENTION_DAYS: "3"`, down from the chart's 3650-day default)
- Resource bump (`resources.requests`/`resources.limits`, before widening remote_write to cluster ops metrics)
- Self-metrics enabled (`config.ZO_PROMETHEUS_ENABLED: "true"`, for the OpenObserve Ops Grafana dashboard)

```bash
helm upgrade --install openobserve openobserve/openobserve-standalone \
  --version 0.92.2 \
  --namespace observability \
  -f k8s/observability/openobserve-values.yaml \
  --wait --timeout 180s
```
