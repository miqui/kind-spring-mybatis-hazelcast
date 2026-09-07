# Troubleshooting Guide

## 1. Prometheus Intermittent Probe Timeouts (Liveness / Readiness)

### Symptom
When inspecting the Prometheus pod via `kubectl describe pod -n observability <pod-name>`, periodic warning events appeared indicating probe timeouts:

```text
Events:
  Type     Reason     Age                    From     Message
  ----     ------     ----                   ----     -------
  Warning  Unhealthy  6m49s (x4 over 39h)    kubelet  spec.containers{prometheus}: Readiness probe failed: Get "http://10.244.4.9:9090/-/ready": context deadline exceeded (Client.Timeout exceeded while awaiting headers)
  Warning  Unhealthy  6m10s (x7 over 4h17m)  kubelet  spec.containers{prometheus}: Liveness probe failed: Get "http://10.244.4.9:9090/-/healthy": context deadline exceeded (Client.Timeout exceeded while awaiting headers)
```

### Root Cause
1. **Strict Default Probe Timeout (`timeoutSeconds: 1`)**:
   Kubernetes defaults `timeoutSeconds` to `1s` when not explicitly declared.
2. **Periodic High CPU / Disk / Remote-Write Contention**:
   During active scrape cycles across multiple jobs (`kubernetes-nodes-cadvisor`, `node-exporter`, `kube-state-metrics`, `otel-collector`), TSDB block compactions, and `remote_write` batch flushes to OpenObserve, Prometheus occasionally took slightly longer than 1 second to respond with HTTP headers for `/-/ready` and `/-/healthy`.
3. **Operational Impact**:
   - Readiness probe failures temporarily dropped the Prometheus pod from endpoint routing.
   - Repeated liveness probe failures (if exceeding `failureThreshold: 3`) would trigger unexpected container restarts.

### Resolution
Updated `k8s/observability/prometheus-deployment.yaml` to increase `timeoutSeconds` from the default `1s` to `3s` and explicitly define `failureThreshold: 3` on both probes:

```yaml
          readinessProbe:
            httpGet:
              path: /-/ready
              port: 9090
            initialDelaySeconds: 5
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /-/healthy
              port: 9090
            initialDelaySeconds: 10
            periodSeconds: 10
            timeoutSeconds: 3
            failureThreshold: 3
```

### Application & Verification
Applied the configuration and monitored deployment rollout:

```bash
kubectl apply -f k8s/observability/prometheus-deployment.yaml
kubectl rollout status deployment/prometheus -n observability
```

Verified the active pod configuration via:

```bash
kubectl get pods -n observability -l app=prometheus
kubectl describe pod -n observability -l app=prometheus
```

Confirmed probe settings:
* **Liveness**: `http-get http://:9090/-/healthy delay=10s timeout=3s period=10s #success=1 #failure=3`
* **Readiness**: `http-get http://:9090/-/ready delay=5s timeout=3s period=5s #success=1 #failure=3`
* **Status**: `Running`, `Ready: True`, `Restart Count: 0`.
