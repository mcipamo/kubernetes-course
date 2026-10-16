# 🧪 Lab 22 · Day-2 Operations

| Goal | Duration | Environment |
|---|---|---|
| Drain a node under a PDB, back up and restore etcd, check certificates and encrypt Secrets at rest | 150 min | `kc` kind cluster |

> ⚠️ This lab modifies control-plane files. If something goes wrong, you can always recreate the
> cluster (module 03). That is the point of a disposable lab.

## Setup

```bash
kubectl create namespace lab-22
kubectl config set-context --current --namespace=lab-22
kubectl create deployment web --image=nginx:1.27 --replicas=4
kubectl rollout status deployment/web
```

## Part A — Node maintenance

### Step 1 · Protect the app with a PDB

```bash
kubectl create pdb web --selector=app=web --min-available=3
kubectl get pdb web
```

✅ **Check:** `ALLOWED DISRUPTIONS 1` (4 replicas, at least 3 must stay available).

### Step 2 · Drain a worker

```bash
kubectl get pods -o wide
kubectl drain kc-worker --ignore-daemonsets --delete-emptydir-data --timeout=120s
kubectl get nodes
kubectl get pods -o wide
```

✅ **Check:** `kc-worker` shows `SchedulingDisabled`, and all `web` Pods now run elsewhere. The drain
evicted them **one at a time**, waiting for replacements to be Ready to respect the PDB.

```bash
kubectl uncordon kc-worker
```

## Part B — etcd backup and restore

### Step 3 · Create some state, then back it up

```bash
kubectl create configmap before-backup --from-literal=state=present
# Helper functions (work in bash and zsh)
etcdctl_kc() {
  kubectl -n kube-system exec etcd-kc-control-plane -- etcdctl \
    --endpoints=https://127.0.0.1:2379 \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    --cert=/etc/kubernetes/pki/etcd/server.crt \
    --key=/etc/kubernetes/pki/etcd/server.key "$@"
}
etcdutl_kc() { kubectl -n kube-system exec etcd-kc-control-plane -- etcdutl "$@"; }
etcdctl_kc snapshot save /var/lib/etcd/kc-snapshot.db
etcdutl_kc snapshot status /var/lib/etcd/kc-snapshot.db -w table
```

✅ **Check:** the status table shows a hash, revision and total keys (hundreds or thousands).

> The snapshot is written inside `/var/lib/etcd`, a hostPath on the control-plane node — in real life
> you would copy it **off** the machine immediately.

### Step 4 · Disaster: someone deletes things

```bash
kubectl create configmap after-backup --from-literal=state=lost-after-restore
kubectl delete deployment web
kubectl delete configmap before-backup
kubectl get deploy,cm
```

### Step 5 · Restore to a new data directory

```bash
etcdutl_kc snapshot restore /var/lib/etcd/kc-snapshot.db --data-dir /var/lib/etcd/restored
docker exec kc-control-plane ls /var/lib/etcd/restored
```

Point the etcd static Pod at the restored data (the kubelet restarts it automatically):

```bash
docker exec kc-control-plane cp /etc/kubernetes/manifests/etcd.yaml /root/etcd.yaml.bak
docker exec kc-control-plane sed -i 's#path: /var/lib/etcd$#path: /var/lib/etcd/restored#' /etc/kubernetes/manifests/etcd.yaml
docker exec kc-control-plane grep -A1 'hostPath' /etc/kubernetes/manifests/etcd.yaml | grep path
sleep 60
kubectl get --raw='/readyz' ; echo
```

Wait until the API server answers again (it can take about a minute while etcd restarts):

```bash
until [ "$(kubectl get --raw='/readyz' 2>/dev/null)" = "ok" ]; do sleep 5; done; echo "API is back"
```

The API server still serves its **cached** view of the old data. As the official guide says, restart
the control-plane components so they read the restored state:

```bash
for c in kube-apiserver kube-controller-manager kube-scheduler; do
  docker exec kc-control-plane sh -c "crictl stop \$(crictl ps --name $c -q)"
done
until [ "$(kubectl get --raw='/readyz' 2>/dev/null)" = "ok" ]; do sleep 5; done
kubectl -n lab-22 get deploy,cm
```

✅ **Check:** `web` and `before-backup` are **back**; `after-backup` is **gone** (it was created after the snapshot).
That is your **RPO** in action: you lose everything since the last backup.

> In production, stop **all** API servers before restoring, restore every etcd member, then start
> the control plane again. Read the official guide before doing this for real.

## Part C — Certificates

### Step 6 · Check expiry

```bash
docker exec kc-control-plane kubeadm certs check-expiration
docker exec kc-control-plane openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -enddate
```

✅ **Check:** leaf certificates expire in ~1 year, CAs in ~10 years. Write down the API server's date —
that is a future outage unless something renews it.

## Part D — Encryption at rest

### Step 7 · Prove Secrets are plain text today

```bash
kubectl create secret generic plain --from-literal=password=VisibleInEtcd
etcdctl_kc get /registry/secrets/lab-22/plain | strings | grep Visible
```

### Step 8 · Enable an `aescbc` provider

```bash
KEY=$(head -c 32 /dev/urandom | base64)
docker exec -i kc-control-plane tee /etc/kubernetes/pki/encryption.yaml >/dev/null <<EOF
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources: ["secrets"]
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: ${KEY}
      - identity: {}
EOF
# /etc/kubernetes/pki is already mounted into the API server Pod
docker exec kc-control-plane sed -i \
  's#- kube-apiserver#- kube-apiserver\n    - --encryption-provider-config=/etc/kubernetes/pki/encryption.yaml#' \
  /etc/kubernetes/manifests/kube-apiserver.yaml
sleep 60; kubectl get --raw='/readyz'; echo
```

### Step 9 · Encrypt new and existing Secrets

```bash
kubectl create secret generic sealed --from-literal=password=HiddenInEtcd
etcdctl_kc get /registry/secrets/lab-22/sealed | strings | head -3
kubectl get secrets -n lab-22 -o json | kubectl replace -f -
etcdctl_kc get /registry/secrets/lab-22/plain | strings | grep -c Visible
```

✅ **Check:** the stored value starts with `k8s:enc:aescbc:v1:key1` and the password is not readable;
after the rewrite, `grep -c Visible` returns `0`. `kubectl get secret sealed -o yaml` still works (the API decrypts).

## 💥 Step 10 · Break it on purpose: the blocked upgrade

Simulate the most common upgrade blocker:

```bash
kubectl create deployment singleton --image=nginx:1.27 --replicas=1
kubectl create pdb singleton --selector=app=singleton --min-available=1
NODE=$(kubectl get pod -l app=singleton -o jsonpath='{.items[0].spec.nodeName}')
kubectl drain $NODE --ignore-daemonsets --delete-emptydir-data --timeout=60s; echo "exit=$?"
```

**Diagnosis:** why does the drain fail? List three ways to unblock it and say which is safest.

<details><summary>Show solution</summary>

`error when evicting pods/"singleton-…": Cannot evict pod as it would violate the pod's disruption budget`.
With 1 replica and `minAvailable: 1`, **allowed disruptions = 0**, forever.

Options, safest first:
1. **Scale to 2 replicas** temporarily (if the app supports it) → 1 disruption allowed → drain proceeds.
2. **Fix the PDB** to something meaningful (`maxUnavailable: 1`) after talking to the owner — a single-replica app has no HA anyway.
3. Delete the Pod / PDB manually (`--disable-eviction`) — causes downtime; only with the owner's agreement.

Platform teams should **lint PDBs** (no PDB that allows 0 disruptions) to keep upgrades unblocked.
```bash
kubectl uncordon $NODE
```
</details>

## 🧹 Cleanup

```bash
kubectl config set-context --current --namespace=default
kubectl delete namespace lab-22
```

Your cluster now runs from the restored etcd data and encrypts Secrets. If anything is unstable,
recreate it: `kind delete cluster --name kc && kind create cluster --name kc --config kind-kc.yaml`.

➡️ Next: [📝 EXERCISES.md](EXERCISES.md)
