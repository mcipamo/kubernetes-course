# 🧪 Lab 07 · Zero-Downtime Releases

| Goal | Duration | Environment |
|---|---|---|
| Roll out, break, roll back, and run blue/green and canary releases | 90 min | `kc` kind cluster |

## Setup

```bash
kubectl create namespace lab-07
kubectl config set-context --current --namespace=lab-07
mkdir -p ~/kc/lab07 && cd ~/kc/lab07
```

## Step 1 · Create the Deployment

Create `web.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 4
  minReadySeconds: 3
  progressDeadlineSeconds: 60
  selector:
    matchLabels:
      app: web
      track: stable          # each Deployment owns a distinct label set
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: web
        track: stable
    spec:
      containers:
        - name: web
          image: ghcr.io/stefanprodan/podinfo:6.6.0
          ports:
            - containerPort: 9898
          readinessProbe:
            httpGet: { path: /readyz, port: 9898 }
            periodSeconds: 2
---
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector:
    app: web                 # the Service selects ALL tracks (stable + canary)
  ports:
    - port: 80
      targetPort: 9898
```

```bash
kubectl apply -f web.yaml
kubectl rollout status deployment/web
kubectl get deploy,rs,pods -l app=web
```

✅ **Check:** 1 Deployment, 1 ReplicaSet, 4 Pods `1/1 Running`.

## Step 2 · Generate continuous traffic

In a second terminal, leave this running (it prints the version answering each request):

```bash
kubectl -n lab-07 run traffic --image=curlimages/curl:8.10.1 --restart=Never -- \
  sh -c 'while true; do curl -s --max-time 1 web/version | grep -o "\"version\":\"[^\"]*\"" || echo ERROR; sleep 0.3; done'
kubectl -n lab-07 logs -f traffic
```

## Step 3 · Rolling update

```bash
kubectl set image deployment/web web=ghcr.io/stefanprodan/podinfo:6.7.0
kubectl annotate deployment/web kubernetes.io/change-cause="podinfo 6.6.0 -> 6.7.0" --overwrite
kubectl get rs -l app=web -w      # watch the old RS shrink and the new one grow; Ctrl+C when done
```

✅ **Check:** in the traffic terminal, responses move from `6.6.0` to `6.7.0` with **no `ERROR` lines**.

## Step 4 · History and rollback

```bash
kubectl rollout history deployment/web
kubectl rollout undo deployment/web
kubectl rollout status deployment/web
kubectl rollout history deployment/web
```

✅ **Check:** the traffic is back on `6.6.0`, and the history shows a new revision number (rollbacks are also revisions).

## 💥 Step 5 · Break it on purpose: a bad release

```bash
kubectl set image deployment/web web=ghcr.io/stefanprodan/podinfo:9.9.9-broken
kubectl rollout status deployment/web --timeout=90s; echo "exit code: $?"
kubectl get pods -l app=web
```

**Diagnosis questions:**
1. How many old Pods are still serving? Did users see errors?
2. What does the Deployment condition say?
3. What is the fastest safe action?

<details><summary>Show solution</summary>

1. All **4 old Pods keep serving**: with `maxUnavailable: 0`, Kubernetes never removed an old Pod
   because the new one never became available. The traffic log shows no errors.
2. ```bash
   kubectl get deploy web -o jsonpath='{.status.conditions[?(@.type=="Progressing")].reason}'; echo
   # ProgressDeadlineExceeded
   ```
   And `rollout status` exits non-zero — a CI pipeline would stop here.
3. Roll back first, investigate later:
   ```bash
   kubectl rollout undo deployment/web && kubectl rollout status deployment/web
   ```
</details>

## Step 6 · Canary (≈ 20% of traffic)

The Service selects `app=web`, so any Pod with that label receives traffic. Add **one** canary Pod next
to the 4 stable ones. Create `canary.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-canary
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
      track: canary          # does not overlap with the stable Deployment selector
  template:
    metadata:
      labels:
        app: web             # matched by the Service
        track: canary
    spec:
      containers:
        - name: web
          image: ghcr.io/stefanprodan/podinfo:6.7.0
          ports:
            - containerPort: 9898
          readinessProbe:
            httpGet: { path: /readyz, port: 9898 }
            periodSeconds: 2
```

```bash
kubectl apply -f canary.yaml
kubectl get pods -l app=web -L track
```

✅ **Check:** about 1 in 5 responses in the traffic log show `6.7.0`.

To **promote** the canary: `kubectl set image deployment/web web=ghcr.io/stefanprodan/podinfo:6.7.0`, then
`kubectl delete -f canary.yaml`. To **abort**: just delete the canary.

## Step 7 · Blue/green

Create a full "green" copy and switch the Service in one move:

```bash
kubectl delete -f canary.yaml --ignore-not-found
kubectl create deployment web-green --image=ghcr.io/stefanprodan/podinfo:6.7.0 --replicas=4 --port=9898
kubectl rollout status deployment/web-green
# web-green Pods carry the label app=web-green; point the Service at them
kubectl patch service web -p '{"spec":{"selector":{"app":"web-green"}}}'
```

✅ **Check:** the traffic log jumps to `6.7.0` instantly, with no mixed responses.
Roll back just as fast: `kubectl patch service web -p '{"spec":{"selector":{"app":"web"}}}'`.

## 🧹 Cleanup

```bash
kubectl config set-context --current --namespace=default
kubectl delete namespace lab-07
```

➡️ Next: [📝 EXERCISES.md](EXERCISES.md)
