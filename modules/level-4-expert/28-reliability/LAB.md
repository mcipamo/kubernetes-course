# 🧪 Lab 28 · Game Day

| Goal | Duration | Environment |
|---|---|---|
| Define an SLO, build burn-rate alerts, harden a service and prove it survives Pod and node failures | 180 min | `kc` kind cluster · kube-prometheus-stack from module 20 |

## Setup

```bash
kubectl -n monitoring get pods | grep -E 'prometheus|grafana' || echo "Install kube-prometheus-stack first (module 20 LAB, Step 1)"
kubectl create namespace lab-28
kubectl config set-context --current --namespace=lab-28
mkdir -p ~/kc/lab28 && cd ~/kc/lab28
```

## Step 1 · A fragile service

```bash
kubectl create deployment checkout --image=ghcr.io/stefanprodan/podinfo:6.7.0 --replicas=1 --port=9898
kubectl patch deploy checkout --type=json -p='[{"op":"replace","path":"/spec/template/spec/containers/0/ports/0","value":{"name":"http","containerPort":9898}}]'
kubectl expose deployment checkout --port=9898
cat <<'EOF' | kubectl apply -f -
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: checkout
spec:
  selector:
    matchLabels: { app: checkout }
  podMetricsEndpoints:
    - port: http
      interval: 15s
EOF
```

## Step 2 · Steady traffic, measured from the client side

```bash
kubectl run traffic --image=curlimages/curl:8.10.1 --restart=Never -- sh -c \
  'while true; do code=$(curl -s -o /dev/null -m 1 -w "%{http_code}" checkout:9898/); echo "$(date +%s) $code"; sleep 0.1; done'
sleep 20; kubectl logs traffic --tail=5
```

> The client log is your "ground truth" SLI. Server-side metrics cannot see requests that never reached a Pod.

## Step 3 · Define the SLO with recording rules

SLO: **99.5% of requests are successful (non-5xx)** over 30 days.

Create `slo.yaml`:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: checkout-slo
  namespace: lab-28
spec:
  groups:
    - name: checkout-sli
      interval: 30s
      rules:
        - record: sli:checkout_errors:ratio_rate5m
          expr: |
            sum(rate(http_requests_total{namespace="lab-28",status=~"5.."}[5m]))
            / sum(rate(http_requests_total{namespace="lab-28"}[5m]))
        - record: sli:checkout_errors:ratio_rate1h
          expr: |
            sum(rate(http_requests_total{namespace="lab-28",status=~"5.."}[1h]))
            / sum(rate(http_requests_total{namespace="lab-28"}[1h]))
    - name: checkout-slo-alerts
      rules:
        - alert: CheckoutErrorBudgetBurnFast
          expr: |
            sli:checkout_errors:ratio_rate1h > (14.4 * 0.005)
            and
            sli:checkout_errors:ratio_rate5m > (14.4 * 0.005)
          for: 2m
          labels:
            severity: page
          annotations:
            summary: "checkout is burning its 30-day error budget >14x too fast"
            runbook_url: https://runbooks.example.com/checkout-slo
```

```bash
kubectl apply -f slo.yaml
```

✅ **Check:** in Prometheus (http://localhost:9090 via the module 20 port-forward) → **Alerts**, the
rule `CheckoutErrorBudgetBurnFast` is listed and `inactive`.

## 💥 Step 4 · Chaos round 1: kill Pods (fragile version)

```bash
for i in 1 2 3 4 5; do kubectl delete pod -l app=checkout --wait=false; sleep 12; done
kubectl logs traffic --since=90s | awk '{print $2}' | sort | uniq -c
```

**Record the result:** how many requests failed (`000` = connection failed/timeout, `5xx`)? With one
replica, every Pod kill is a full outage.

## Step 5 · Harden the service

Create `hardened.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: checkout
spec:
  replicas: 3
  selector:
    matchLabels: { app: checkout }
  strategy:
    rollingUpdate: { maxSurge: 1, maxUnavailable: 0 }
  template:
    metadata:
      labels: { app: checkout }
    spec:
      terminationGracePeriodSeconds: 30
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: ScheduleAnyway
          labelSelector: { matchLabels: { app: checkout } }
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: ScheduleAnyway
          labelSelector: { matchLabels: { app: checkout } }
      containers:
        - name: podinfo
          image: ghcr.io/stefanprodan/podinfo:6.7.0
          ports:
            - { name: http, containerPort: 9898 }
          readinessProbe:
            httpGet: { path: /readyz, port: http }
            periodSeconds: 2
            failureThreshold: 2
          livenessProbe:
            httpGet: { path: /healthz, port: http }
            periodSeconds: 10
          lifecycle:
            preStop:
              sleep: { seconds: 5 }
          resources:
            requests: { cpu: 50m, memory: 64Mi }
            limits: { memory: 128Mi }
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: checkout
spec:
  maxUnavailable: 1
  selector:
    matchLabels: { app: checkout }
  unhealthyPodEvictionPolicy: AlwaysAllow
```

```bash
kubectl apply -f hardened.yaml
kubectl rollout status deployment/checkout
kubectl get pods -l app=checkout -o wide -L topology.kubernetes.io/zone
kubectl get pdb checkout
```

✅ **Check:** 3 replicas spread over both zones; the PDB allows 1 disruption.

## 💥 Step 6 · Chaos round 2: kill Pods (hardened)

```bash
for i in 1 2 3 4 5; do kubectl delete $(kubectl get pod -l app=checkout -o name | shuf -n1) --wait=false; sleep 12; done
kubectl logs traffic --since=90s | awk '{print $2}' | sort | uniq -c
```

✅ **Check:** failures drop to **zero or nearly zero**. Compare with round 1.

## 💥 Step 7 · Chaos round 3: lose a "zone"

```bash
kubectl drain kc-worker2 --ignore-daemonsets --delete-emptydir-data --timeout=120s
kubectl get pods -l app=checkout -o wide
kubectl logs traffic --since=60s | awk '{print $2}' | sort | uniq -c
kubectl uncordon kc-worker2
```

✅ **Check:** the drain evicts one Pod at a time (PDB), replicas move to `kc-worker`, and the client
sees no errors. Then try the **unplanned** version — `docker stop kc-worker2` — and measure again:
what changes, and why is an unplanned failure worse than a drain?

<details><summary>Show explanation</summary>

With `docker stop`, nothing evicts the Pods gracefully: the node goes `NotReady` after ~40 s, its Pods
stay in the endpoints until the node controller marks them not ready, and new Pods are only created
after the ~5-minute eviction timeout. Clients may see errors during that window. Mitigations: more
replicas than zones need, readiness-based endpoint removal (faster with Pod-level health signals at the
gateway), shorter `tolerationSeconds` for stateless apps, and outlier detection at the load balancer or
mesh. Run `docker start kc-worker2` afterwards.
</details>

## Step 8 · Burn the budget on purpose

```bash
kubectl set env deployment/checkout PODINFO_RANDOM_ERROR=true
kubectl rollout status deployment/checkout
```

Wait 5–10 minutes and watch Prometheus → Alerts.

✅ **Check:** the error ratio rises well above `0.072` (14.4 × 0.5%) and `CheckoutErrorBudgetBurnFast`
goes `pending` → `firing`. Roll back first, investigate later:

```bash
kubectl rollout undo deployment/checkout
```

## 📝 Step 9 · Game day report

Write a short report: hypothesis, experiments, results per round (error counts), what you changed,
remaining risks, and two action items. This is the format you will use in the capstone.

## 🧹 Cleanup

```bash
docker start kc-worker2 2>/dev/null; kubectl uncordon kc-worker2
kubectl config set-context --current --namespace=default
kubectl delete namespace lab-28
```

➡️ Next: [📝 EXERCISES.md](EXERCISES.md)
