# 🧪 Lab 20 · See Everything

| Goal | Duration | Environment |
|---|---|---|
| Install Prometheus + Grafana, query golden signals, and find a hidden problem using only telemetry | 120 min | `kc` kind cluster + `helm` (≥ 8 GB RAM for Docker) |

## Step 1 · Install the monitoring stack

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install kps prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace \
  --set grafana.adminPassword=kc-admin \
  --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false \
  --set prometheus.prometheusSpec.podMonitorSelectorNilUsesHelmValues=false
kubectl -n monitoring rollout status deployment/kps-grafana --timeout=300s
kubectl -n monitoring get pods
```

✅ **Check:** Prometheus, Alertmanager, Grafana, kube-state-metrics and one node-exporter per node are `Running`.

## Step 2 · Open the UIs

```bash
kubectl -n monitoring port-forward svc/kps-kube-prometheus-stack-prometheus 9090:9090 >/dev/null 2>&1 &
kubectl -n monitoring port-forward svc/kps-grafana 3000:80 >/dev/null 2>&1 &
```

- Prometheus: http://localhost:9090 → **Status → Targets**: every target should be `UP`.
- Grafana: http://localhost:3000 (admin / `kc-admin`) → Dashboards → *Kubernetes / Compute Resources / Cluster*.

> On kind, some control plane targets (etcd, scheduler, controller-manager) may show `DOWN` because
> they listen on localhost. That is expected in this lab.

## Step 3 · Deploy an instrumented app

```bash
kubectl create namespace lab-20
kubectl config set-context --current --namespace=lab-20
kubectl create deployment shop --image=ghcr.io/stefanprodan/podinfo:6.7.0 --replicas=3 --port=9898
# Name the container port: PodMonitors select ports by name
kubectl patch deploy shop --type=json -p='[{"op":"replace","path":"/spec/template/spec/containers/0/ports/0","value":{"name":"http","containerPort":9898}}]'
kubectl expose deployment shop --port=9898
```

Create `podmonitor.yaml` so Prometheus scrapes the app's `/metrics`:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: shop
  namespace: lab-20
spec:
  selector:
    matchLabels: { app: shop }
  podMetricsEndpoints:
    - port: http
      path: /metrics
      interval: 15s
```

```bash
kubectl apply -f podmonitor.yaml
```

✅ **Check:** after ~1 minute, Prometheus → Status → Targets shows `podMonitor/lab-20/shop` with 3 targets `UP`.

## Step 4 · Generate traffic (with some errors)

```bash
kubectl run load --image=curlimages/curl:8.10.1 --restart=Never -- sh -c \
  'while true; do curl -s -o /dev/null shop:9898/; curl -s -o /dev/null shop:9898/status/500; curl -s -o /dev/null shop:9898/delay/1; sleep 0.2; done'
```

## Step 5 · Query the golden signals (Prometheus → Graph)

```promql
# Traffic (requests per second) by status
sum by (status) (rate(http_requests_total{namespace="lab-20"}[2m]))

# Error ratio
sum(rate(http_requests_total{namespace="lab-20",status=~"5.."}[2m]))
  / sum(rate(http_requests_total{namespace="lab-20"}[2m]))

# p95 latency
histogram_quantile(0.95, sum by (le) (rate(http_request_duration_seconds_bucket{namespace="lab-20"}[2m])))

# Saturation: CPU per Pod
sum by (pod) (rate(container_cpu_usage_seconds_total{namespace="lab-20",container!=""}[2m]))
```

✅ **Check:** the error ratio is around **33%** (one of three calls is `/status/500`), and p95 latency
is close to **1 s** (the `/delay/1` calls).

## Step 6 · Kubernetes state with kube-state-metrics

```promql
kube_deployment_status_replicas_available{namespace="lab-20"}
kube_pod_container_status_restarts_total{namespace="lab-20"}
sum by (namespace) (kube_pod_container_resource_requests{resource="cpu"})
```

## Step 7 · Logs and events

```bash
kubectl logs -l app=shop --prefix --tail=3
kubectl events --types=Warning -A | tail -5
docker exec kc-worker ls /var/log/pods | grep lab-20
```

Logs live as files on each node — that is what a Fluent Bit/Vector DaemonSet would read and ship.

## 💥 Step 8 · Break it on purpose: find the silent problem

Run this without reading the YAML (your instructor "deploys a new version"):

```bash
kubectl apply -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: checkout
  namespace: lab-20
spec:
  replicas: 2
  selector:
    matchLabels: { app: checkout }
  template:
    metadata:
      labels: { app: checkout }
    spec:
      containers:
        - name: checkout
          image: polinux/stress:1.0.4
          command: ["sh", "-c", "while true; do stress --vm 1 --vm-bytes 120M --vm-hang 0 --timeout 20s; sleep 5; done"]
          resources:
            requests: { cpu: 50m, memory: 64Mi }
            limits:   { cpu: 100m, memory: 100Mi }
EOF
```

Wait 3 minutes. **Using only Prometheus and Grafana** (no `kubectl describe`), answer: is `checkout`
healthy? What is happening, and what is the evidence?

<details><summary>Show solution</summary>

```promql
increase(kube_pod_container_status_restarts_total{namespace="lab-20",container="checkout"}[5m])
kube_pod_container_status_last_terminated_reason{namespace="lab-20",container="checkout"}
max by (pod) (container_memory_working_set_bytes{namespace="lab-20",container="checkout"})
rate(container_cpu_cfs_throttled_periods_total{namespace="lab-20",container="checkout"}[2m])
  / rate(container_cpu_cfs_periods_total{namespace="lab-20",container="checkout"}[2m])
```
- Restarts keep increasing; last terminated reason = **OOMKilled** (allocating 120 MB with a 100 Mi limit).
- Memory climbs to the limit in a saw-tooth pattern.
- CPU is heavily **throttled** (100m limit).

The Deployment still shows "Available" most of the time — **state metrics alone would not page you**.
That is why you alert on restarts/OOMs and on user-facing SLOs. Clean up: `kubectl delete deploy checkout`.
</details>

## 🧹 Cleanup

```bash
kill %1 %2 2>/dev/null
kubectl config set-context --current --namespace=default
kubectl delete namespace lab-20
# Keep the monitoring stack for modules 21 and 28, or remove it:
# helm uninstall kps -n monitoring && kubectl delete namespace monitoring
```

➡️ Next: [📝 EXERCISES.md](EXERCISES.md)
