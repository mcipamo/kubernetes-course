# 🧪 Lab 15 · Put Every Pod in Its Place

| Goal | Duration | Environment |
|---|---|---|
| Control placement with affinity, taints, spread and priority | 90 min | `kc` kind cluster (workers labeled `zone-a`, `zone-b`) |

## Setup

```bash
kubectl create namespace lab-15
kubectl config set-context --current --namespace=lab-15
kubectl get nodes -L topology.kubernetes.io/zone
```

## Step 1 · nodeSelector

```bash
kubectl label node kc-worker2 disktype=ssd
kubectl run fast --image=nginx:1.27 --overrides='{"spec":{"nodeSelector":{"disktype":"ssd"}}}'
kubectl get pod fast -o wide
```

✅ **Check:** `fast` runs on `kc-worker2`.

## Step 2 · Taints keep Pods away

```bash
kubectl taint nodes kc-worker2 dedicated=batch:NoSchedule
kubectl create deployment spread-me --image=nginx:1.27 --replicas=4
kubectl get pods -l app=spread-me -o wide
```

✅ **Check:** all 4 Pods run on `kc-worker` (and none on the tainted `kc-worker2` or the control plane).

## Step 3 · Tolerations + affinity = dedicated nodes

Create `batch.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: batch
spec:
  replicas: 2
  selector:
    matchLabels: { app: batch }
  template:
    metadata:
      labels: { app: batch }
    spec:
      tolerations:                         # allowed on the tainted node…
        - { key: dedicated, operator: Equal, value: batch, effect: NoSchedule }
      affinity:
        nodeAffinity:                      # …and required to go there
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - { key: disktype, operator: In, values: [ssd] }
      containers:
        - name: worker
          image: busybox:1.36
          command: ["sleep", "3600"]
```

```bash
kubectl apply -f batch.yaml
kubectl get pods -l app=batch -o wide
```

✅ **Check:** both `batch` Pods run on `kc-worker2`.

## Step 4 · NoExecute evicts

```bash
kubectl taint nodes kc-worker maintenance=true:NoExecute
sleep 10; kubectl get pods -o wide
kubectl taint nodes kc-worker maintenance-
```

**What happened?** Pods on `kc-worker` without a matching toleration were evicted. The `spread-me`
replacements are `Pending` (worker2 is tainted for batch) until you remove the NoExecute taint.

## Step 5 · Topology spread across zones

```bash
kubectl taint nodes kc-worker2 dedicated-
kubectl delete deployment spread-me batch
```

Create `web.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 6
  selector:
    matchLabels: { app: web }
  template:
    metadata:
      labels: { app: web }
    spec:
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels: { app: web }
      containers:
        - name: web
          image: nginx:1.27
          resources:
            requests: { cpu: 10m, memory: 16Mi }
```

```bash
kubectl apply -f web.yaml
kubectl get pods -l app=web -o custom-columns=POD:.metadata.name,NODE:.spec.nodeName --sort-by=.spec.nodeName
```

✅ **Check:** exactly **3** Pods in `zone-a` (`kc-worker`) and **3** in `zone-b` (`kc-worker2`).

## Step 6 · Priority and preemption

Create `priority.yaml`:

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: low
value: 100
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: critical
value: 100000
```

```bash
kubectl apply -f priority.yaml
# Allocatable CPU of a worker, in millicores (handles both "8" and "7800m")
CPU_M=$(kubectl get node kc-worker -o jsonpath='{.status.allocatable.cpu}' | awk '/m$/{sub("m","");print;next}{print $1*1000}')
REQ="$((CPU_M * 7 / 10))m"; echo "each filler Pod requests $REQ"
# Fill both workers with low-priority Pods that request 70% of a node's CPU
kubectl create deployment filler --image=busybox:1.36 --replicas=2 -- sleep 3600
kubectl patch deployment filler --type=json -p='[
  {"op":"add","path":"/spec/template/spec/priorityClassName","value":"low"},
  {"op":"add","path":"/spec/template/spec/containers/0/resources","value":{"requests":{"cpu":"'$REQ'"}}}]'
kubectl rollout status deployment/filler
```

Now a critical Pod that needs the same amount of CPU — no node has room for it:

```bash
kubectl run vip --image=nginx:1.27 --overrides='{"spec":{"priorityClassName":"critical","containers":[{"name":"vip","image":"nginx:1.27","resources":{"requests":{"cpu":"'$REQ'"}}}]}}'
sleep 10; kubectl get pods -o wide
kubectl get events --field-selector reason=Preempted
```

✅ **Check:** one `filler` Pod was **preempted**, `vip` is `Running`, and the evicted filler replacement is `Pending`.

## 💥 Step 7 · Break it on purpose: "lose" a zone

```bash
kubectl delete pod vip; kubectl delete deployment filler
kubectl cordon kc-worker2                    # simulate zone-b unavailable for new Pods
kubectl scale deployment web --replicas=8
sleep 10; kubectl get pods -l app=web -o wide | grep -c Pending
kubectl describe pod $(kubectl get pod -l app=web --field-selector=status.phase=Pending -o name | head -1) | grep -A2 FailedScheduling
```

**Diagnosis:** why are new replicas `Pending` although `kc-worker` has free resources? Is that what you
want during a real zone outage?

<details><summary>Show solution</summary>

`whenUnsatisfiable: DoNotSchedule` with `maxSkew: 1` forbids placing more Pods in zone-a than zone-b + 1.
Zone-b cannot take Pods (cordoned), so the scheduler refuses — the message says
`didn't match pod topology spread constraints`.

During a real zone outage you usually prefer **availability over balance**: use
`whenUnsatisfiable: ScheduleAnyway`, or keep `DoNotSchedule` with `minDomains` thoughtfully and rely on
capacity planning (N+1 zones). There is no universal answer — but you must choose **before** the incident.

```bash
kubectl uncordon kc-worker2
```
</details>

## 🧹 Cleanup

```bash
kubectl label node kc-worker2 disktype-
kubectl delete priorityclass low critical
kubectl config set-context --current --namespace=default
kubectl delete namespace lab-15
kubectl describe nodes | grep -i taints     # only the control-plane taint should remain
```

➡️ Next: [📝 EXERCISES.md](EXERCISES.md)
