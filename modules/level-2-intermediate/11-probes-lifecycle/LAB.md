# 🧪 Lab 11 · Health Checks That Help (and Hurt)

| Goal | Duration | Environment |
|---|---|---|
| Watch each probe act, cause a probe-driven outage and fix graceful shutdown | 90 min | `kc` kind cluster |

## Setup

```bash
kubectl create namespace lab-11
kubectl config set-context --current --namespace=lab-11
mkdir -p ~/kc/lab11 && cd ~/kc/lab11
```

## Step 1 · Deploy with a full probe set

Create `app.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  replicas: 3
  selector:
    matchLabels: { app: app }
  template:
    metadata:
      labels: { app: app }
    spec:
      terminationGracePeriodSeconds: 30
      containers:
        - name: app
          image: ghcr.io/stefanprodan/podinfo:6.7.0
          ports:
            - { name: http, containerPort: 9898 }
          startupProbe:
            httpGet: { path: /healthz, port: http }
            periodSeconds: 2
            failureThreshold: 30
          readinessProbe:
            httpGet: { path: /readyz, port: http }
            periodSeconds: 2
            failureThreshold: 2
          livenessProbe:
            httpGet: { path: /healthz, port: http }
            periodSeconds: 5
            failureThreshold: 3
          lifecycle:
            preStop:
              sleep: { seconds: 5 }
---
apiVersion: v1
kind: Service
metadata:
  name: app
spec:
  selector: { app: app }
  ports:
    - { port: 80, targetPort: http }
```

```bash
kubectl apply -f app.yaml
kubectl rollout status deployment/app
kubectl describe pod -l app=app | grep -E 'Startup|Readiness|Liveness' | head -3
```

✅ **Check:** 3 Pods `1/1 Running`.

## Step 2 · Readiness takes a Pod out of traffic

```bash
P=$(kubectl get pod -l app=app -o jsonpath='{.items[0].metadata.name}')
kubectl exec $P -- curl -s -X POST localhost:9898/readyz/disable
sleep 6; kubectl get pods -l app=app
kubectl get endpointslices -l kubernetes.io/service-name=app -o jsonpath='{range .items[*].endpoints[*]}{.targetRef.name}{" ready="}{.conditions.ready}{"\n"}{end}'
```

✅ **Check:** `$P` shows `0/1` (still `Running`, **0 restarts**) and `ready=false`.

```bash
kubectl exec $P -- curl -s -X POST localhost:9898/readyz/enable
```

## Step 3 · Liveness restarts a stuck container

```bash
kubectl exec $P -- curl -s -X POST localhost:9898/healthz/disable 2>/dev/null || true
kubectl get pod $P -w      # wait ~15 s: RESTARTS goes 0 → 1; Ctrl+C
kubectl get events --field-selector involvedObject.name=$P,reason=Unhealthy | tail -3
```

> If your podinfo version has no `/healthz/disable`, simulate it with
> `kubectl exec $P -- kill -STOP 1` (freezes the process).

✅ **Check:** the event says `Liveness probe failed` and then `Container app failed liveness probe, will be restarted`.

## Step 4 · Measure errors during a rollout

Start a load loop in a second terminal:

```bash
kubectl run load --image=curlimages/curl:8.10.1 --restart=Never -- \
  sh -c 'ok=0; err=0; while true; do if curl -s --max-time 1 -o /dev/null -w "%{http_code}" app | grep -q 200; then ok=$((ok+1)); else err=$((err+1)); fi; echo "ok=$ok err=$err"; sleep 0.1; done'
kubectl logs -f load
```

In the first terminal:

```bash
kubectl rollout restart deployment/app && kubectl rollout status deployment/app
```

✅ **Check:** `err` stays at **0** (or almost) during the whole rollout. Write down the final numbers.

## 💥 Step 5 · Break it on purpose: the "smart" liveness probe

A teammate wants the app to restart "when Redis is down", so the liveness probe now depends on a
Redis Service:

```bash
kubectl create deployment redis --image=redis:7.4
kubectl expose deployment redis --port=6379
kubectl patch deployment app --type=json -p='[
 {"op":"replace","path":"/spec/template/spec/containers/0/livenessProbe","value":
   {"exec":{"command":["sh","-c","nc -z -w1 redis 6379"]},"periodSeconds":5,"failureThreshold":2}}]'
kubectl rollout status deployment/app
# Now Redis has a short "blip"
kubectl scale deployment redis --replicas=0
sleep 30; kubectl get pods -l app=app
kubectl logs load --tail=3
```

**Diagnosis:** what happened to the app Pods and to user traffic? Was restarting useful?

<details><summary>Show solution</summary>

All 3 app Pods fail liveness at the same time and **restart together** (`RESTARTS` grows on every
Pod). During restarts there are no Ready endpoints, so `err` climbs. Restarting did **not** fix Redis —
it only turned a dependency blip into a full outage.

Fix: liveness must check only the process itself; express dependency health through readiness (or
better, degrade gracefully):
```bash
kubectl scale deployment redis --replicas=1
kubectl apply -f app.yaml       # restore the original local-only probes
```
</details>

## Step 6 · Graceful shutdown done wrong

```bash
kubectl patch deployment app --type=json -p='[{"op":"remove","path":"/spec/template/spec/containers/0/lifecycle"}]'
kubectl rollout status deployment/app
kubectl rollout restart deployment/app && kubectl rollout status deployment/app
kubectl logs load --tail=1
```

Compare the `err` delta with Step 4. Without the `preStop` delay, some requests can hit Pods that
already received `SIGTERM` but were not yet removed from every node's rules. Put it back with
`kubectl apply -f app.yaml`.

## 🧹 Cleanup

```bash
kubectl config set-context --current --namespace=default
kubectl delete namespace lab-11
```

➡️ Next: [📝 EXERCISES.md](EXERCISES.md)
