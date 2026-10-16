# 🧪 Lab 10 · Requests, Limits and Budgets

| Goal | Duration | Environment |
|---|---|---|
| See throttling, OOM kills, QoS and quotas with your own eyes | 90 min | `kc` kind cluster + metrics-server |

## Setup

```bash
kubectl create namespace lab-10
kubectl config set-context --current --namespace=lab-10
kubectl top nodes || echo "Install metrics-server first (module 03 LAB, optional addons)"
```

## Step 1 · Three QoS classes

Create `qos.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: guaranteed
spec:
  containers:
    - name: app
      image: nginx:1.27
      resources:
        requests: { cpu: 100m, memory: 64Mi }
        limits:   { cpu: 100m, memory: 64Mi }
---
apiVersion: v1
kind: Pod
metadata:
  name: burstable
spec:
  containers:
    - name: app
      image: nginx:1.27
      resources:
        requests: { cpu: 50m, memory: 32Mi }
        limits:   { memory: 128Mi }
---
apiVersion: v1
kind: Pod
metadata:
  name: besteffort
spec:
  containers:
    - name: app
      image: nginx:1.27
```

```bash
kubectl apply -f qos.yaml
kubectl get pods -o custom-columns=NAME:.metadata.name,QOS:.status.qosClass
```

✅ **Check:** `Guaranteed`, `Burstable`, `BestEffort`.

## Step 2 · CPU throttling

```bash
kubectl run cpu-hog --image=busybox:1.36 --restart=Never \
  --overrides='{"spec":{"containers":[{"name":"cpu-hog","image":"busybox:1.36","command":["sh","-c","while true; do :; done"],"resources":{"requests":{"cpu":"100m"},"limits":{"cpu":"200m"}}}]}}'
sleep 60; kubectl top pod cpu-hog
```

✅ **Check:** usage stays around **200m** even though the loop wants a full core.

```bash
NODE=$(kubectl get pod cpu-hog -o jsonpath='{.spec.nodeName}')
UID_=$(kubectl get pod cpu-hog -o jsonpath='{.metadata.uid}' | tr - _)
docker exec $NODE sh -c "d=\$(find /sys/fs/cgroup -type d -name \"*pod${UID_}*\" | head -1); grep throttled \$d/cpu.stat"
```

`nr_throttled` keeps growing: the kernel pauses the container every period.

## Step 3 · OOM kill

```bash
kubectl run mem-hog --image=polinux/stress:1.0.4 --restart=Never \
  --overrides='{"spec":{"containers":[{"name":"mem-hog","image":"polinux/stress:1.0.4","command":["stress","--vm","1","--vm-bytes","150M","--vm-hang","1"],"resources":{"limits":{"memory":"100Mi"}}}]}}'
sleep 15; kubectl get pod mem-hog
kubectl get pod mem-hog -o jsonpath='{.status.containerStatuses[0].state.terminated.reason} {.status.containerStatuses[0].state.terminated.exitCode}'; echo
```

✅ **Check:** `OOMKilled 137`.

## Step 4 · Scheduling uses requests, not usage

```bash
kubectl describe node kc-worker | sed -n '/Allocatable/,/System Info/p' | head -8
kubectl create deployment big --image=nginx:1.27 --replicas=3
kubectl set resources deployment big --requests=cpu=3,memory=64Mi
kubectl get pods -l app=big
kubectl describe pod -l app=big | grep -A3 Events | tail -4
```

✅ **Check:** some `big` Pods stay `Pending` with `FailedScheduling … Insufficient cpu` even though the nodes are idle.

```bash
kubectl delete deployment big
```

## Step 5 · LimitRange defaults

Create `limits.yaml`:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: defaults
spec:
  limits:
    - type: Container
      defaultRequest: { cpu: 50m, memory: 64Mi }
      default:        { memory: 128Mi }
      max:            { cpu: "1", memory: 512Mi }
```

```bash
kubectl apply -f limits.yaml
kubectl run defaulted --image=nginx:1.27
kubectl get pod defaulted -o jsonpath='{.spec.containers[0].resources}'; echo
kubectl run too-big --image=nginx:1.27 --overrides='{"spec":{"containers":[{"name":"too-big","image":"nginx:1.27","resources":{"limits":{"memory":"1Gi"}}}]}}'
```

✅ **Check:** `defaulted` received the defaults; `too-big` is rejected (`maximum memory usage per Container is 512Mi`).

## 💥 Step 6 · Break it on purpose: exhaust the budget

Create `quota.yaml`:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: budget
spec:
  hard:
    requests.cpu: 500m
    requests.memory: 512Mi
    pods: "8"
```

```bash
kubectl apply -f quota.yaml
kubectl create deployment burst --image=nginx:1.27 --replicas=10
sleep 10; kubectl get deploy burst
kubectl describe quota budget
```

**Diagnosis:** the Deployment wants 10 replicas but has fewer. No Pod is `Pending`. Where is the error
reported, and why?

<details><summary>Show solution</summary>

The quota is enforced by **admission** at Pod creation time, so failed Pods are never created and never
appear as `Pending`. The error lives in the **ReplicaSet** events:
```bash
kubectl describe rs -l app=burst | grep -i 'exceeded quota' | head -2
# Error creating: pods "burst-..." is forbidden: exceeded quota: budget, requested: requests.cpu=50m, used: …
```
`describe quota` shows which limit was hit (CPU requests, or `pods: 8` counting the Pods from earlier steps).
Lesson: when replicas are missing but nothing is Pending, **look at the ReplicaSet events**.
</details>

## 🧹 Cleanup

```bash
kubectl config set-context --current --namespace=default
kubectl delete namespace lab-10
```

➡️ Next: [📝 EXERCISES.md](EXERCISES.md)
