# 🧪 Lab 16 · Scale Out, Scale Up, Scale Back

| Goal | Duration | Environment |
|---|---|---|
| Drive an HPA with real load, tune its behavior and resize a Pod in place | 90 min | `kc` kind cluster + metrics-server |

## Setup

```bash
kubectl top nodes   # must work; otherwise install metrics-server (module 03 LAB)
kubectl create namespace lab-16
kubectl config set-context --current --namespace=lab-16
mkdir -p ~/kc/lab16 && cd ~/kc/lab16
```

## Step 1 · A CPU-hungry app with requests

Create `app.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
spec:
  selector:
    matchLabels: { app: php-apache }
  template:
    metadata:
      labels: { app: php-apache }
    spec:
      containers:
        - name: php-apache
          image: registry.k8s.io/hpa-example
          ports:
            - containerPort: 80
          resources:
            requests: { cpu: 200m }       # HPA utilization is relative to this
            limits:   { cpu: 500m }
---
apiVersion: v1
kind: Service
metadata:
  name: php-apache
spec:
  selector: { app: php-apache }
  ports:
    - port: 80
```

```bash
kubectl apply -f app.yaml
kubectl rollout status deployment/php-apache
```

> This is the same app used by the [official HPA walkthrough](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/): each request burns CPU.

## Step 2 · Create the HPA

Create `hpa.yaml`:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 8
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 60      # short for the lab; use ~300 in production
```

```bash
kubectl apply -f hpa.yaml
sleep 30; kubectl get hpa php-apache
```

✅ **Check:** `TARGETS` shows something like `cpu: 0%/50%` (not `<unknown>`), `REPLICAS 1`.

## Step 3 · Generate load and watch it scale

In a second terminal: `kubectl -n lab-16 get hpa php-apache -w`

In the first terminal:

```bash
kubectl run load --image=busybox:1.36 --restart=Never -- \
  sh -c 'while true; do wget -q -O- http://php-apache >/dev/null; done'
sleep 120
kubectl get hpa php-apache
kubectl get pods -l app=php-apache
kubectl describe hpa php-apache | tail -8
```

✅ **Check:** utilization climbed above 50%, and replicas grew (typically to 4–7). Events say
`New size: N; reason: cpu resource utilization (percentage of request) above target`.

## Step 4 · Stop the load and watch it scale down

```bash
kubectl delete pod load
sleep 150; kubectl get hpa php-apache
```

✅ **Check:** after the 60 s stabilization window, replicas drop back toward 1.

## Step 5 · In-place vertical resize (no restart)

```bash
POD=$(kubectl get pod -l app=php-apache -o jsonpath='{.items[0].metadata.name}')
kubectl get pod $POD -o jsonpath='{.status.containerStatuses[0].restartCount}{" restarts, cpu req="}{.spec.containers[0].resources.requests.cpu}{"\n"}'
kubectl patch pod $POD --subresource resize --patch \
  '{"spec":{"containers":[{"name":"php-apache","resources":{"requests":{"cpu":"300m"},"limits":{"cpu":"700m"}}}]}}'
sleep 5
kubectl get pod $POD -o jsonpath='{.status.containerStatuses[0].restartCount}{" restarts, actual="}{.status.containerStatuses[0].resources}{"\n"}'
```

✅ **Check:** the restart count is unchanged and the actual resources show `cpu: 300m` request / `700m` limit.

> Note: resizing a Pod owned by a Deployment is great for emergencies, but the next rollout restores
> the template values. Make permanent changes in the Deployment (or let VPA do it).

## 💥 Step 6 · Break it on purpose

```bash
kubectl patch deployment php-apache --type=json -p='[{"op":"remove","path":"/spec/template/spec/containers/0/resources"}]'
kubectl rollout status deployment/php-apache
sleep 45; kubectl get hpa php-apache
kubectl describe hpa php-apache | grep -A4 Conditions
```

**Diagnosis:** what does the HPA show, and why can it no longer scale?

<details><summary>Show solution</summary>

`TARGETS` becomes `<unknown>/50%` and the condition `ScalingActive=False` says
`FailedGetResourceMetric … missing request for cpu`. Utilization is a percentage **of the request** —
without a request, there is nothing to divide by. Fix: `kubectl apply -f app.yaml`.
Prevent it with a LimitRange default or an admission policy requiring CPU requests (module 26).
</details>

## 🧹 Cleanup

```bash
kubectl config set-context --current --namespace=default
kubectl delete namespace lab-16
```

➡️ Next: [📝 EXERCISES.md](EXERCISES.md)
