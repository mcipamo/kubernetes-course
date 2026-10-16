# 🧪 Lab 27 · GPUs Without GPUs

| Goal | Duration | Environment |
|---|---|---|
| Allocate (simulated) GPUs with DRA, use CEL selectors and shared claims, and queue AI jobs with Kueue | 150 min | `kc` kind cluster (≥ v1.34) · `helm` · `git` |

> You do not need real GPUs. The upstream **DRA example driver** publishes fake GPUs that behave like
> real ones from Kubernetes' point of view.

## Part A — Dynamic Resource Allocation

### Step 1 · Install the example DRA driver

Follow the official tutorial
[Install drivers and allocate devices with DRA](https://kubernetes.io/docs/tutorials/cluster-management/install-use-dra/)
(section *Install an example DRA driver*) against your `kc` cluster. It deploys the
[`dra-example-driver`](https://github.com/kubernetes-sigs/dra-example-driver) as a DaemonSet and a
DeviceClass for simulated GPUs.

```bash
kubectl get deviceclasses
kubectl get resourceslices -o wide
kubectl get resourceslices -o yaml | yq '.items[0].spec.devices[0]'
```

✅ **Check:** a DeviceClass (e.g., `gpu.example.com`) exists and each worker publishes a ResourceSlice
listing several devices with attributes (model, driver version, UUID) and capacity (memory).

> ⚠️ Names in the commands below assume the DeviceClass `gpu.example.com`. If the tutorial created a
> different name, replace it.

### Step 2 · One GPU per Pod

```bash
kubectl create namespace lab-27
kubectl config set-context --current --namespace=lab-27
mkdir -p ~/kc/lab27 && cd ~/kc/lab27
```

Create `one-gpu.yaml`:

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaimTemplate
metadata:
  name: single-gpu
spec:
  spec:
    devices:
      requests:
        - name: gpu
          exactly:
            deviceClassName: gpu.example.com
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: inference
spec:
  replicas: 2
  selector:
    matchLabels: { app: inference }
  template:
    metadata:
      labels: { app: inference }
    spec:
      resourceClaims:
        - name: gpu
          resourceClaimTemplateName: single-gpu
      containers:
        - name: server
          image: ubuntu:24.04
          command: ["sh", "-c", "env | grep -i -E 'gpu|device' ; sleep infinity"]
          resources:
            claims:
              - name: gpu
```

```bash
kubectl apply -f one-gpu.yaml
kubectl rollout status deployment/inference
kubectl get resourceclaims
kubectl logs deploy/inference | head
```

✅ **Check:** two claims (one per Pod) are `allocated,reserved`, and each container sees environment
variables describing **its own** simulated GPU.

### Step 3 · Precise requests with CEL

```bash
kubectl get resourceslices -o json | jq -r '.items[].spec.devices[] | "\(.name) \(.attributes|tostring) \(.capacity|tostring)"' | head -5
```

Pick an attribute you see (for example the device `index` or `model`) and request only matching devices.
Create `picky.yaml` (adjust the expression to your attributes):

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaim
metadata:
  name: picky
spec:
  devices:
    requests:
      - name: gpu
        exactly:
          deviceClassName: gpu.example.com
          selectors:
            - cel:
                expression: "device.attributes['gpu.example.com'].index == 0"
---
apiVersion: v1
kind: Pod
metadata:
  name: picky
spec:
  resourceClaims:
    - name: gpu
      resourceClaimName: picky
  containers:
    - name: c
      image: ubuntu:24.04
      command: ["sleep", "infinity"]
      resources:
        claims: [{ name: gpu }]
```

```bash
kubectl apply -f picky.yaml
kubectl get resourceclaim picky -o jsonpath='{.status.allocation.devices.results}'; echo
```

✅ **Check:** the allocated device matches your CEL condition.

### Step 4 · Sharing one claim between Pods

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: picky-sidecar
spec:
  resourceClaims:
    - name: gpu
      resourceClaimName: picky          # the SAME claim as Pod "picky"
  containers:
    - name: c
      image: ubuntu:24.04
      command: ["sleep", "infinity"]
      resources:
        claims: [{ name: gpu }]
EOF
kubectl get pods -o wide
kubectl get resourceclaim picky -o jsonpath='{.status.reservedFor}'; echo
```

✅ **Check:** both Pods run **on the same node** (the device lives there), and the claim lists two consumers.

## 💥 Step 5 · Break it on purpose: ask for the impossible

```bash
kubectl scale deployment inference --replicas=20
sleep 20; kubectl get pods -l app=inference | grep -c Pending
kubectl describe pod $(kubectl get pod -l app=inference --field-selector=status.phase=Pending -o name | head -1) | grep -A3 Events
```

**Diagnosis:** why are some Pods Pending although nodes have spare CPU and memory? What would you
put in place so that, on a real cluster, teams wait in a fair queue instead?

<details><summary>Show solution</summary>

The scheduler cannot allocate a device for every claim: all simulated GPUs are taken (`cannot allocate
all claims` / `0/3 nodes are available … ResourceClaim`). CPU and memory are irrelevant — the scarce
resource is the device.

On a real cluster: **Kueue** with a `ClusterQueue` quota per team (including device resources),
`LocalQueue`s per namespace, priorities and preemption; workloads wait **suspended** (no Pods created)
until capacity is available. Scale back down:
```bash
kubectl scale deployment inference --replicas=2
```
</details>

## Part B — Queueing batch jobs with Kueue

### Step 6 · Install Kueue

```bash
# Check the latest release at https://github.com/kubernetes-sigs/kueue/releases
KUEUE_VERSION=v0.20.2          # tested; newer releases should work too
kubectl apply --server-side -f https://github.com/kubernetes-sigs/kueue/releases/download/${KUEUE_VERSION}/manifests.yaml
kubectl -n kueue-system rollout status deployment/kueue-controller-manager --timeout=180s
```

### Step 7 · A quota and a queue

> ⚠️ Kueue v0.20 serves only `kueue.x-k8s.io/v1beta2`. If you install a different release, check with
> `kubectl api-resources --api-group=kueue.x-k8s.io` and adjust `apiVersion` below.

Create `kueue.yaml`:

```yaml
apiVersion: kueue.x-k8s.io/v1beta2
kind: ResourceFlavor
metadata:
  name: default-flavor
---
apiVersion: kueue.x-k8s.io/v1beta2
kind: ClusterQueue
metadata:
  name: ml-team
spec:
  namespaceSelector: {}
  resourceGroups:
    - coveredResources: ["cpu", "memory"]
      flavors:
        - name: default-flavor
          resources:
            - { name: cpu, nominalQuota: 1 }
            - { name: memory, nominalQuota: 1Gi }
---
apiVersion: kueue.x-k8s.io/v1beta2
kind: LocalQueue
metadata:
  name: training
  namespace: lab-27
spec:
  clusterQueue: ml-team
```

```bash
kubectl apply -f kueue.yaml
```

### Step 8 · Submit more work than the quota allows

```bash
for i in 1 2 3; do
cat <<EOF | kubectl create -f -
apiVersion: batch/v1
kind: Job
metadata:
  generateName: train-
  labels:
    kueue.x-k8s.io/queue-name: training
spec:
  suspend: true
  parallelism: 2
  completions: 2
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: train
          image: busybox:1.36
          command: ["sh", "-c", "echo training epoch; sleep 30"]
          resources:
            requests: { cpu: 250m, memory: 128Mi }
EOF
done
kubectl get workloads
kubectl get jobs -w      # watch them run in turn; Ctrl+C
```

✅ **Check:** only **two** Jobs run at once (each needs 500m CPU of a 1-CPU quota); the third waits as
a pending Workload and starts when one finishes — no Pods are created for it while it waits.

## 🧹 Cleanup

```bash
kubectl config set-context --current --namespace=default
kubectl delete namespace lab-27
kubectl delete clusterqueue ml-team; kubectl delete resourceflavor default-flavor
# Uninstall Kueue and the example driver if you do not need them:
# kubectl delete -f https://github.com/kubernetes-sigs/kueue/releases/download/${KUEUE_VERSION}/manifests.yaml
```

➡️ Next: [📝 EXERCISES.md](EXERCISES.md)
