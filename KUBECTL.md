# kubectl Commands — OpenObserve Observability Rollout

`kubectl` commands used while adding OpenObserve observability for the message REST API,
grouped by phase.

## From `deploy-kind.sh` (the automated deployment)

```bash
kubectl config use-context "kind-${CLUSTER_NAME}"
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.11.3/deploy/static/provider/kind/deploy.yaml
kubectl wait --namespace ingress-nginx --for=condition=ready pod --selector=app.kubernetes.io/component=controller --timeout=120s
kubectl apply -k k8s/observability/
kubectl apply -k k8s/
kubectl rollout status statefulset/postgres --timeout=120s
kubectl rollout status deployment/hazelcast --timeout=120s
kubectl rollout status deployment/otel-collector -n observability --timeout=120s
kubectl rollout status deployment/prometheus -n observability --timeout=120s
kubectl rollout status deployment/grafana -n observability --timeout=120s
kubectl rollout status statefulset/openobserve -n observability --timeout=180s
kubectl rollout status deployment/message-service --timeout=180s
kubectl get nodes -L workload -o wide
kubectl get pods -l app=postgres -o wide
kubectl get pods -l app=hazelcast -o wide
kubectl get pods -l app=message-service -o wide
kubectl get svc message-service
kubectl get pods -n observability -o wide
```

## Manual investigation (diagnosing why ingress-nginx wasn't ready)

```bash
kubectl config use-context kind-kind-springboot-mybatis-cluster
kubectl get nodes -o wide
kubectl get pods -n ingress-nginx -o wide
kubectl describe pod -n ingress-nginx -l app.kubernetes.io/component=controller
```

## Manual verification (Prometheus remote_write → OpenObserve)

```bash
kubectl exec -n observability deploy/prometheus -- wget -qO- 'http://localhost:9090/api/v1/query?query=prometheus_remote_storage_succeeded_samples_total'
kubectl exec -n observability deploy/prometheus -- wget -qO- 'http://localhost:9090/api/v1/query?query=prometheus_remote_storage_samples_failed_total'
kubectl exec -n observability deploy/prometheus -- wget -qO- 'http://localhost:9090/api/v1/query?query=up{job="otel-collector"}'
kubectl exec -n observability deploy/prometheus -- wget -qO- 'http://localhost:9090/api/v1/query?query=prometheus_remote_storage_pending_samples'
kubectl exec -n observability deploy/prometheus -- wget -qO- 'http://localhost:9090/metrics'
kubectl exec -n observability deploy/prometheus -- wget -qO- 'http://localhost:9090/api/v1/query?query={__name__=~"prometheus_remote_storage.*"}'
kubectl port-forward -n observability svc/prometheus 9090:9090
kubectl logs -n observability deploy/prometheus --tail=100
kubectl logs -n observability deploy/prometheus --tail=200
kubectl exec -n observability deploy/prometheus -- cat /etc/prometheus/prometheus.yml
kubectl exec -n observability deploy/prometheus -- ls -la /etc/prometheus/openobserve-auth/
kubectl exec -n observability deploy/prometheus -- cat /etc/prometheus/openobserve-auth/password
```

## Diagnosing the OpenObserve `MemoryTableOverflowError`

```bash
kubectl logs -n observability openobserve-0 --tail=100
kubectl top pod -n observability openobserve-0
kubectl get pod -n observability openobserve-0 -o jsonpath='{.status.containerStatuses[0].restartCount}'
kubectl describe pod -n observability openobserve-0
kubectl logs -n observability openobserve-0 --tail=300
kubectl logs -n observability openobserve-0 --tail=30
kubectl logs -n observability openobserve-0
kubectl exec -n observability openobserve-0 -- df -h /data
kubectl exec -n observability openobserve-0 -- du -sh /data/*
```

## Applying the fix (scoped `write_relabel_configs`) and recovering

```bash
kubectl apply -k /Users/miqui/development/k8s-springboot-mybatis-o2/k8s/observability/
kubectl rollout restart deployment/prometheus -n observability
kubectl rollout status deployment/prometheus -n observability --timeout=90s
kubectl delete pod -n observability openobserve-0
kubectl rollout status statefulset/openobserve -n observability --timeout=120s
kubectl port-forward -n observability svc/prometheus 9090:9090
```

## Final verification

```bash
kubectl port-forward -n observability svc/openobserve 5080:5080
kubectl get pods -n observability -o wide
```

## Diagnosing and fixing Prometheus probe timeouts (context deadline exceeded)

```bash
# Inspect pod events and probe failures
kubectl describe pod -n observability -l app=prometheus
kubectl describe -n observability pod/<pod-name>

# Check Prometheus resource usage during scrape and remote-write loads
kubectl top pod -n observability -l app=prometheus

# Apply updated deployment with increased probe timeouts (timeoutSeconds: 3, failureThreshold: 3)
kubectl apply -f k8s/observability/prometheus-deployment.yaml
# or via Kustomize:
kubectl apply -k k8s/observability/

# Monitor rollout
kubectl rollout status deployment/prometheus -n observability

# Verify new pod status and active probe settings
kubectl get pods -n observability -l app=prometheus -o wide
kubectl describe pod -n observability -l app=prometheus
```
