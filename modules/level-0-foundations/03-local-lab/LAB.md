# 🧪 Lab 03 · Build the `kc` Cluster

| Goal | Duration | Environment |
|---|---|---|
| Create the course cluster and run a first workload | 60 min | Docker + kind + kubectl |

## Step 1 · Write the cluster config

```bash
mkdir -p ~/kc && cd ~/kc
```

Save this as `kind-kc.yaml` — every lab in the course uses it:

```yaml
# kind-kc.yaml — the Kubernetes Course lab cluster
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    kubeadmConfigPatches:
      - |
        kind: InitConfiguration
        nodeRegistration:
          kubeletExtraArgs:
            node-labels: "ingress-ready=true"
    extraPortMappings:        # browser → localhost:80/443 → this node
      - containerPort: 80
        hostPort: 80
        protocol: TCP
      - containerPort: 443
        hostPort: 443
        protocol: TCP
  - role: worker
    labels:
      topology.kubernetes.io/zone: zone-a
  - role: worker
    labels:
      topology.kubernetes.io/zone: zone-b
```

## Step 2 · Create the cluster

```bash
kind create cluster --name kc --config kind-kc.yaml
# Optional: pin a version to match production, e.g. --image kindest/node:v1.37.0
```

**Expected output (abridged):**
```text
Creating cluster "kc" ...
 ✓ Ensuring node image
 ✓ Preparing nodes 📦 📦 📦
 ✓ Starting control-plane 🕹️
 ✓ Joining worker nodes 🚜
Set kubectl context to "kind-kc"
```

✅ **Check:**
```bash
kubectl wait --for=condition=Ready nodes --all --timeout=180s
kubectl get nodes -L topology.kubernetes.io/zone
```
Three nodes `Ready`; the two workers show `zone-a` and `zone-b`.

## Step 3 · Look at your kubeconfig

```bash
kubectl config current-context        # kind-kc
kubectl config view --minify          # only the current context (secrets redacted)
```

## Step 4 · Meet the system components

```bash
kubectl get pods -n kube-system -o wide
```

You will see `etcd-kc-control-plane`, `kube-apiserver-…`, `kube-scheduler-…`,
`kube-controller-manager-…`, `coredns-…`, `kube-proxy-…` (one per node) and `kindnet-…` (the network
plugin). Module 04 explains every one of them.

✅ **Check:** every Pod in `kube-system` is `Running`.

## Step 5 · Your first workload

```bash
kubectl create namespace lab-03
kubectl -n lab-03 create deployment hello --image=nginx:1.27 --replicas=2
kubectl -n lab-03 rollout status deployment/hello
kubectl -n lab-03 get pods -o wide
```

✅ **Check:** two Pods `Running`, ideally on different worker nodes.

## Step 6 · Reach it from your laptop

```bash
kubectl -n lab-03 port-forward deployment/hello 8080:80
# In another terminal:
curl -s localhost:8080 | grep -i title
```

**Expected output:** `<title>Welcome to nginx!</title>`. Press `Ctrl+C` to stop the port-forward.

## Step 7 · Load your own image from Lab 01

```bash
kind load docker-image kc/hello:1.0 --name kc
kubectl -n lab-03 run myapp --image=kc/hello:1.0 --image-pull-policy=IfNotPresent --port=8000
kubectl -n lab-03 wait --for=condition=Ready pod/myapp
kubectl -n lab-03 port-forward pod/myapp 8000:8000 &
sleep 2; curl localhost:8000; kill %1
```

> 🦭 **Using Podman instead of Docker?** Podman names local images `localhost/<name>`, and
> `kind load docker-image` needs the `docker` binary. Load the image through an archive instead:
> ```bash
> podman save -o hello.tar localhost/kc/hello:1.0
> kind load image-archive hello.tar --name kc
> kubectl -n lab-03 run myapp --image=localhost/kc/hello:1.0 --image-pull-policy=IfNotPresent --port=8000
> ```
> In every lab, replace `docker exec <node>` with `podman exec <node>`.

✅ **Check:** you get `Hello from myapp · version 1.0` — your image, running on Kubernetes.

## 💥 Step 8 · Break it on purpose

Kill a whole worker node and watch Kubernetes react:

```bash
kubectl -n lab-03 get pods -o wide            # note which node each Pod is on
docker stop kc-worker
kubectl get nodes -w                           # wait ~40 s until kc-worker is NotReady, then Ctrl+C
kubectl -n lab-03 get pods -o wide
```

**Diagnosis questions:** What state is the node in? What happened to the Pods on it? Why did the
Deployment Pods behave differently from the `myapp` Pod?

<details><summary>Show explanation</summary>

- The kubelet on `kc-worker` stopped sending heartbeats (Leases), so the node controller marked it `NotReady`.
- After about **5 minutes** (the default `tolerationSeconds` for the `not-ready`/`unreachable` taints),
  its Pods are evicted. The **Deployment** creates replacements on healthy nodes.
- The bare Pod `myapp` has **no controller**, so nobody recreates it. Lesson: never run bare Pods for real
  workloads (module 07).

Bring the node back:
```bash
docker start kc-worker
kubectl wait --for=condition=Ready node/kc-worker --timeout=120s
```
</details>

## Step 9 · Practice the reset

```bash
time (kind delete cluster --name kc && kind create cluster --name kc --config kind-kc.yaml)
```

✅ **Check:** the full reset takes about 1–2 minutes. You now have a disposable lab.

## 🧹 Cleanup

```bash
kubectl delete namespace lab-03
```

Keep the cluster — you will use it in every module.

> 💡 **Optional addons** — install them now or when a module asks for them:
> ```bash
> # metrics-server (modules 10, 16, 20). kind kubelets use self-signed certs, hence the extra flag.
> kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
> kubectl -n kube-system patch deployment metrics-server --type=json \
>   -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'
> ```

➡️ Next: [📝 EXERCISES.md](EXERCISES.md)
