# 🧪 Lab 29 · Explore the Newest Release

| Goal | Duration | Environment |
|---|---|---|
| Inspect feature gates, try recent GA/beta features and turn on a gate in a scratch cluster | 120 min | A scratch kind cluster on the newest node image |

## Step 1 · A scratch cluster with a feature gate

Create `kind-new.yaml` (use the newest `kindest/node` image your kind release supports):

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
featureGates:
  HPAScaleToZero: true            # beta in v1.37 — explicitly on for this lab
nodes:
  - role: control-plane
  - role: worker
```

```bash
kind create cluster --name kc-new --config kind-new.yaml
kubectl config use-context kind-kc-new
kubectl version
```

✅ **Check:** the server version is the newest release your kind supports.

## Step 2 · Read the feature gates from the cluster itself

```bash
kubectl get --raw /metrics | grep '^kubernetes_feature_enabled' | sed 's/kubernetes_feature_enabled//' | sort | head -40
kubectl get --raw /metrics | grep kubernetes_feature_enabled | grep -E 'InPlacePodVerticalScaling|UserNamespacesSupport|HPAScaleToZero|DynamicResourceAllocation'
```

✅ **Check:** each line shows the gate name, its `stage` (ALPHA/BETA/"" for GA) and value `1` (on) or
`0` (off). `HPAScaleToZero` is `1` because you enabled it.

## Step 3 · GA in v1.35: in-place resize

```bash
kubectl create namespace lab-29
kubectl config set-context --current --namespace=lab-29
kubectl run resize --image=nginx:1.27 --overrides='{"spec":{"containers":[{"name":"resize","image":"nginx:1.27","resources":{"requests":{"cpu":"100m","memory":"64Mi"},"limits":{"cpu":"200m","memory":"128Mi"}}}]}}'
kubectl wait --for=condition=Ready pod/resize
kubectl patch pod resize --subresource resize --patch '{"spec":{"containers":[{"name":"resize","resources":{"requests":{"cpu":"300m"},"limits":{"cpu":"500m"}}}]}}'
kubectl get pod resize -o jsonpath='{.status.containerStatuses[0].restartCount}{" "}{.status.containerStatuses[0].resources}{"\n"}'
```

✅ **Check:** restart count `0`, actual CPU request `300m`.

## Step 4 · GA in v1.36: user namespaces

```bash
kubectl run userns --image=busybox:1.36 --overrides='{"spec":{"hostUsers":false,"containers":[{"name":"userns","image":"busybox:1.36","command":["sleep","3600"]}]}}'
kubectl wait --for=condition=Ready pod/userns --timeout=60s && kubectl exec userns -- cat /proc/self/uid_map
```

✅ **Check:** UID 0 in the container maps to a high, unprivileged UID on the host.

## Step 5 · GA in v1.36: PSI metrics from the kubelet

```bash
NODE=$(kubectl get nodes -o name | grep worker | head -1 | cut -d/ -f2)
kubectl get --raw "/api/v1/nodes/$NODE/proxy/metrics/cadvisor" | grep -E '^container_pressure_(cpu|memory|io)' | head -5
```

✅ **Check:** `container_pressure_*_seconds_total` metrics appear (requires cgroup v2 and a kernel with
PSI — Docker Desktop and modern Linux have both). ⚠️ If empty, check `kubectl get --raw "/api/v1/nodes/$NODE/proxy/stats/summary" | jq '.node.cpu.psi'`.

## Step 6 · Beta in v1.37: HPA to zero

```bash
kubectl create deployment idle --image=nginx:1.27 --replicas=1
cat <<'EOF' | kubectl apply -f - && kubectl get hpa idle
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: idle
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: idle
  minReplicas: 0
  maxReplicas: 3
  metrics:
    - type: Object
      object:
        describedObject:
          apiVersion: v1
          kind: Service
          name: idle
        metric:
          name: requests_per_second
        target:
          type: Value
          value: "10"
EOF
```

✅ **Check:** the API **accepts** `minReplicas: 0` (without the gate it is rejected). Without a custom
metrics adapter the HPA cannot read the metric, so it will not actually scale — that is fine: the
point is to see how a gate changes API validation. Try the same manifest on your `kc` cluster (gate
off) and compare the error.

## Step 7 · Deprecation warnings in practice

```bash
kubectl create service clusterip ext --tcp=80:80
kubectl patch service ext -p '{"spec":{"externalIPs":["203.0.113.10"]}}'
```

✅ **Check:** on v1.36+ kubectl prints a **deprecation warning** for `externalIPs` (deprecated in v1.36).
Warnings like this are your early-warning system — CI should fail on them.

## 💥 Step 8 · Break it on purpose: a manifest from the future

Your teammate copied this from a blog post about an alpha feature:

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: future
spec:
  containers:
    - name: c
      image: busybox:1.36
      command: ["sleep", "3600"]
      resources:
        requests: { cpu: 100m }
  tolerations:
    - key: example.com/score
      operator: Gt
      value: "5"
      effect: NoSchedule
EOF
```

**Diagnosis:** what happens and why? How do you check, from the cluster, whether the feature exists?

<details><summary>Show solution</summary>

The API server rejects it (`Unsupported value: "Gt"`) or, depending on version, accepts it only if the
alpha gate for numeric toleration operators is on. Check:
```bash
kubectl get --raw /metrics | grep kubernetes_feature_enabled | grep -i toleration
kubectl explain pod.spec.tolerations.operator
```
Lesson: blog posts describe features of a **specific version and stage**. Always check the gate and
`kubectl explain` on your cluster before using new fields — and never rely on alpha in production.
</details>

## 🧹 Cleanup

```bash
kind delete cluster --name kc-new
kubectl config use-context kind-kc
```

➡️ Next: [📝 EXERCISES.md](EXERCISES.md)
