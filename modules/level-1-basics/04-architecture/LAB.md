# 🧪 Lab 04 · Dissect a Cluster

| Goal | Duration | Environment |
|---|---|---|
| Observe every component at work and break the scheduler | 60 min | `kc` kind cluster |

## Setup

```bash
kubectl config use-context kind-kc
kubectl create namespace lab-04
```

## Step 1 · Health of the control plane

```bash
kubectl get --raw='/readyz?verbose' | tail -5
kubectl get --raw='/livez'; echo
```

**Expected output:** lines like `[+]etcd ok` and finally `readyz check passed`, then `ok`.

✅ **Check:** no line starts with `[-]`.

## Step 2 · Find the static Pods

```bash
docker exec kc-control-plane ls /etc/kubernetes/manifests
kubectl -n kube-system get pods -l tier=control-plane -o wide
```

These Pods are not created through the API: the kubelet on the control-plane node reads the YAML files
in that folder and runs them directly. They are called **static Pods**.

## Step 3 · Watch the controllers react

Open a second terminal with a live view:

```bash
kubectl -n lab-04 get deploy,rs,pods -w
```

In the first terminal:

```bash
kubectl -n lab-04 create deployment web --image=nginx:1.27 --replicas=3
```

Watch the order of events: Deployment → ReplicaSet → 3 Pods (`Pending` → `ContainerCreating` → `Running`).

✅ **Check:** `kubectl -n lab-04 get pods -o wide` shows 3 Pods across the two workers.

## Step 4 · Read the story in the events

```bash
kubectl -n lab-04 get events --sort-by=.lastTimestamp | tail -15
```

Find the `Scheduled` event (from `default-scheduler`), the `Pulling`/`Pulled`/`Created`/`Started`
events (from `kubelet`) and the `ScalingReplicaSet` event (from `deployment-controller`). Each component
leaves its fingerprint.

## Step 5 · Node heartbeats and leader election

```bash
kubectl -n kube-node-lease get leases
kubectl -n kube-system get lease kube-scheduler -o jsonpath='{.spec.holderIdentity}'; echo
```

✅ **Check:** there is one Lease per node, and `renewTime` updates every ~10 seconds
(`kubectl -n kube-node-lease get lease kc-worker -o yaml | grep renewTime`).

## Step 6 · Peek into etcd

```bash
kubectl -n kube-system exec etcd-kc-control-plane -- etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/deployments/lab-04/web --keys-only
```

**Expected output:** `/registry/deployments/lab-04/web`. Your Deployment is literally a key in etcd.

## 💥 Step 7 · Break it on purpose: kill the scheduler

```bash
# Move the scheduler manifest away; the kubelet stops the static Pod
docker exec kc-control-plane mv /etc/kubernetes/manifests/kube-scheduler.yaml /tmp/
sleep 20
kubectl -n kube-system get pods -l component=kube-scheduler

# Now scale up
kubectl -n lab-04 scale deployment web --replicas=6
kubectl -n lab-04 get pods
```

**Diagnosis questions:**
1. What state are the new Pods in? What does `kubectl describe pod` say in Events?
2. Did the existing 3 Pods stop serving?
3. Which components still worked, and which one did not?

<details><summary>Show explanation</summary>

1. The 3 new Pods are `Pending` with **no events at all** — nobody tried to schedule them. A `Pending`
   Pod *with* a `FailedScheduling` event means "the scheduler tried and no node fits"; *without events*
   it means "the scheduler is not running".
2. No. The data plane is independent. The running Pods keep serving.
3. The API server, etcd and controllers worked (the ReplicaSet created 6 Pod objects). Only the
   **binding** step was missing.

Fix it and watch the Pods get scheduled immediately:
```bash
docker exec kc-control-plane mv /tmp/kube-scheduler.yaml /etc/kubernetes/manifests/
kubectl -n lab-04 get pods -w
```
</details>

## 🧹 Cleanup

```bash
kubectl delete namespace lab-04
kubectl -n kube-system get pods -l component=kube-scheduler   # make sure it is Running
```

➡️ Next: [📝 EXERCISES.md](EXERCISES.md)
