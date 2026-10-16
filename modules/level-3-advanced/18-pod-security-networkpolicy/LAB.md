# 🧪 Lab 18 · Lock It Down

| Goal | Duration | Environment |
|---|---|---|
| Enforce `restricted` Pod Security and build a zero-trust network for a 3-tier app | 120 min | `kc` kind cluster (recent kind: kindnet enforces NetworkPolicy) |

## Setup

```bash
mkdir -p ~/kc/lab18 && cd ~/kc/lab18
kind version    # NetworkPolicy support in kindnet arrived in kind v0.24+
```

## Part A — Pod Security

### Step 1 · Preview, then enforce

```bash
kubectl create namespace lab-18
kubectl -n lab-18 run root-nginx --image=nginx:1.27
kubectl label --dry-run=server --overwrite ns lab-18 pod-security.kubernetes.io/enforce=restricted
```

**Expected output:** a warning that `root-nginx` violates `restricted` (allowPrivilegeEscalation,
capabilities, runAsNonRoot, seccompProfile). The dry run changes nothing.

```bash
kubectl label --overwrite ns lab-18 \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/warn=restricted
kubectl -n lab-18 run root-nginx-2 --image=nginx:1.27
```

✅ **Check:** the new Pod is **rejected** with `violates PodSecurity "restricted:latest"`. The existing
`root-nginx` keeps running (PSA only checks new Pods).

### Step 2 · A compliant Pod

Create `hardened.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hardened
  namespace: lab-18
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 10001
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: app
      image: ghcr.io/stefanprodan/podinfo:6.7.0
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop: ["ALL"]
      volumeMounts:
        - { name: tmp, mountPath: /tmp }
  volumes:
    - { name: tmp, emptyDir: {} }
```

```bash
kubectl apply -f hardened.yaml
kubectl -n lab-18 wait --for=condition=Ready pod/hardened
kubectl -n lab-18 exec hardened -- id
kubectl -n lab-18 exec hardened -- sh -c 'touch /bin/evil' ; echo "exit=$?"
kubectl -n lab-18 exec hardened -- grep CapEff /proc/1/status
```

✅ **Check:** `uid=10001`, writing to `/bin` fails (`Read-only file system`), and `CapEff: 0000000000000000`.

### Step 3 · User namespaces

```bash
kubectl -n lab-18 run userns --image=busybox:1.36 \
  --overrides='{"spec":{"hostUsers":false,"securityContext":{"runAsNonRoot":true,"runAsUser":1000,"seccompProfile":{"type":"RuntimeDefault"}},"containers":[{"name":"userns","image":"busybox:1.36","command":["sleep","3600"],"securityContext":{"allowPrivilegeEscalation":false,"capabilities":{"drop":["ALL"]}}}]}}'
kubectl -n lab-18 wait --for=condition=Ready pod/userns --timeout=60s && \
  kubectl -n lab-18 exec userns -- cat /proc/self/uid_map
```

✅ **Check:** `uid_map` shows UID 0 inside mapped to a **large, unprivileged host UID** range (e.g., `0 1879048192 65536`).

> ⚠️ User namespaces need a recent kernel and a runtime that supports idmapped mounts. If the Pod fails
> on your laptop, read its events — that is a lesson in itself.

## Part B — NetworkPolicy

### Step 4 · A three-tier app

```bash
kubectl create namespace lab-18-net
kubectl config set-context --current --namespace=lab-18-net
for app in frontend api db; do
  kubectl run $app --image=ghcr.io/stefanprodan/podinfo:6.7.0 --labels=app=$app --port=9898
done
kubectl wait --for=condition=Ready pod --all --timeout=90s
kubectl get pods -o wide
```

Helper to test connectivity:

```bash
check() { kubectl exec $1 -- sh -c "nc -w 2 -z $(kubectl get pod $2 -o jsonpath='{.status.podIP}') 9898 && echo '✅ $1 → $2' || echo '⛔ $1 → $2'"; }
check frontend api; check frontend db; check api db
```

✅ **Check:** all three succeed — by default everything is open.

### Step 5 · Default deny + allow DNS

Create `netpol.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: lab-18-net
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: lab-18-net
spec:
  podSelector: {}
  policyTypes: [Egress]
  egress:
    - to:
        - namespaceSelector:
            matchLabels: { kubernetes.io/metadata.name: kube-system }
          podSelector:
            matchLabels: { k8s-app: kube-dns }
      ports:
        - { protocol: UDP, port: 53 }
        - { protocol: TCP, port: 53 }
```

```bash
kubectl apply -f netpol.yaml
check frontend api; check frontend db; check api db
kubectl exec frontend -- nslookup kubernetes.default | tail -2
```

✅ **Check:** every connection is ⛔, but DNS still resolves.

### Step 6 · Open only the required paths

Create `allow.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-to-api
  namespace: lab-18-net
spec:
  podSelector:
    matchLabels: { app: api }
  policyTypes: [Ingress]
  ingress:
    - from:
        - podSelector:
            matchLabels: { app: frontend }
      ports:
        - { protocol: TCP, port: 9898 }
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-egress
  namespace: lab-18-net
spec:
  podSelector:
    matchLabels: { app: frontend }
  policyTypes: [Egress]
  egress:
    - to:
        - podSelector:
            matchLabels: { app: api }
      ports:
        - { protocol: TCP, port: 9898 }
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-to-db
  namespace: lab-18-net
spec:
  podSelector:
    matchLabels: { app: db }
  policyTypes: [Ingress]
  ingress:
    - from:
        - podSelector:
            matchLabels: { app: api }
      ports:
        - { protocol: TCP, port: 9898 }
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-egress
  namespace: lab-18-net
spec:
  podSelector:
    matchLabels: { app: api }
  policyTypes: [Egress]
  egress:
    - to:
        - podSelector:
            matchLabels: { app: db }
      ports:
        - { protocol: TCP, port: 9898 }
```

```bash
kubectl apply -f allow.yaml
check frontend api; check api db; check frontend db
```

✅ **Check:** `frontend → api` ✅, `api → db` ✅, `frontend → db` ⛔. Remember: traffic needs an
**egress allow on the source** and an **ingress allow on the destination**.

## 💥 Step 7 · Break it on purpose: the AND/OR trap

A teammate wants to allow monitoring Pods from namespace `monitoring` to reach the api, and writes:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-monitoring
  namespace: lab-18-net
spec:
  podSelector:
    matchLabels: { app: api }
  policyTypes: [Ingress]
  ingress:
    - from:
        - namespaceSelector:
            matchLabels: { kubernetes.io/metadata.name: monitoring }
        - podSelector:
            matchLabels: { app: prometheus }
```

```bash
kubectl apply -f allow-monitoring.yaml
kubectl label pod frontend app=prometheus --overwrite
kubectl exec frontend -- true   # frontend is now "prometheus" in lab-18-net
```

**Diagnosis:** who can now reach the `api` Pod on **any** port? Is it what the teammate intended?

<details><summary>Show solution</summary>

The two selectors are **separate list items**, so they are ORed:
- **any Pod** in namespace `monitoring`, **or**
- any Pod labeled `app=prometheus` **in `lab-18-net`** (a bare `podSelector` means "this namespace").

And with no `ports`, **all ports** are allowed. The intent was "Prometheus Pods in the monitoring namespace" — one item with both selectors:
```yaml
  ingress:
    - from:
        - namespaceSelector:
            matchLabels: { kubernetes.io/metadata.name: monitoring }
          podSelector:
            matchLabels: { app: prometheus }
      ports:
        - { protocol: TCP, port: 9090 }
```
One missing dash changes the meaning completely. Review NetworkPolicies like code — with tests.
</details>

## 🧹 Cleanup

```bash
kubectl config set-context --current --namespace=default
kubectl delete namespace lab-18 lab-18-net
```

➡️ Next: [📝 EXERCISES.md](EXERCISES.md)
