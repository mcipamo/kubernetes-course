# 🧪 Lab 12 · Data That Survives

| Goal | Duration | Environment |
|---|---|---|
| Compare ephemeral and persistent storage, and recover from a storage mistake | 75 min | `kc` kind cluster (local-path provisioner) |

## Setup

```bash
kubectl create namespace lab-12
kubectl config set-context --current --namespace=lab-12
mkdir -p ~/kc/lab12 && cd ~/kc/lab12
kubectl get storageclass
```

**Expected output:** a `standard (default)` class with provisioner `rancher.io/local-path` and
`VOLUMEBINDINGMODE WaitForFirstConsumer`.

## Step 1 · `emptyDir` dies with the Pod

```bash
kubectl run scratch --image=busybox:1.36 --restart=Never \
  --overrides='{"spec":{"containers":[{"name":"scratch","image":"busybox:1.36","command":["sh","-c","echo hello > /data/f && sleep 3600"],"volumeMounts":[{"name":"d","mountPath":"/data"}]}],"volumes":[{"name":"d","emptyDir":{}}]}}'
kubectl wait --for=condition=Ready pod/scratch
kubectl exec scratch -- cat /data/f
kubectl delete pod scratch --now
```

The data is gone with the Pod. `emptyDir` survives **container** restarts, but not Pod deletion.

## Step 2 · Claim persistent storage

Create `pvc.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: notes
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 1Gi
```

```bash
kubectl apply -f pvc.yaml
kubectl get pvc notes
```

✅ **Check:** the PVC is `Pending`. `kubectl describe pvc notes` says `waiting for first consumer to be created before binding`. That is `WaitForFirstConsumer` at work.

## Step 3 · Use it

Create `writer.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: writer
spec:
  replicas: 1
  strategy:
    type: Recreate                 # RWO volume: never run two Pods at once
  selector:
    matchLabels: { app: writer }
  template:
    metadata:
      labels: { app: writer }
    spec:
      securityContext:
        fsGroup: 1000              # make the volume writable by GID 1000
      containers:
        - name: writer
          image: busybox:1.36
          securityContext:
            runAsUser: 1000
          command: ["sh", "-c", "while true; do date >> /data/notes.log; sleep 5; done"]
          volumeMounts:
            - { name: notes, mountPath: /data }
      volumes:
        - name: notes
          persistentVolumeClaim:
            claimName: notes
```

```bash
kubectl apply -f writer.yaml
kubectl rollout status deployment/writer
kubectl get pvc notes; kubectl get pv
sleep 15; kubectl exec deploy/writer -- cat /data/notes.log
```

✅ **Check:** PVC `Bound`, a PV was created automatically, and the log has several lines.

## Step 4 · Survive Pod deletion

```bash
kubectl delete pod -l app=writer
kubectl rollout status deployment/writer
kubectl exec deploy/writer -- sh -c 'wc -l /data/notes.log; head -1 /data/notes.log'
```

✅ **Check:** the first timestamp is from **before** the deletion. The data survived.

## Step 5 · Where does it physically live?

```bash
PV=$(kubectl get pvc notes -o jsonpath='{.spec.volumeName}')
kubectl get pv $PV -o jsonpath='{.spec.local.path}{.spec.hostPath.path}{"\n"}{.spec.nodeAffinity}{"\n"}'
```

The local-path provisioner created a directory on **one node** and set `nodeAffinity`. The Pod can now
only run on that node — a real-world trade-off of local storage.

## Step 6 · Generic ephemeral volume

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: cache
spec:
  containers:
    - name: c
      image: busybox:1.36
      command: ["sleep", "3600"]
      volumeMounts: [{ name: cache, mountPath: /cache }]
  volumes:
    - name: cache
      ephemeral:
        volumeClaimTemplate:
          spec:
            accessModes: [ReadWriteOnce]
            resources: { requests: { storage: 256Mi } }
EOF
kubectl get pvc
kubectl delete pod cache --now; sleep 3; kubectl get pvc
```

✅ **Check:** a PVC `cache-cache` appears with the Pod and disappears with it.

## 💥 Step 7 · Break it on purpose: delete the claim

```bash
kubectl get pv $PV -o jsonpath='{.spec.persistentVolumeReclaimPolicy}'; echo
kubectl delete deployment writer
kubectl delete pvc notes
sleep 5; kubectl get pv
```

**Diagnosis:** where did the data go? How would you have prevented this?

<details><summary>Show solution</summary>

The reclaim policy is **`Delete`**, so deleting the PVC deleted the PV and the data. Prevention:
- Set the PV to `Retain` **before** deleting anything:
  `kubectl patch pv $PV -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'`
- Use a StorageClass with `reclaimPolicy: Retain` for important data.
- Protect PVCs with GitOps/RBAC, and keep **backups** (Velero, CSI snapshots).

Try it again: recreate the PVC and writer, patch the PV to `Retain`, delete the PVC, and check that the
PV stays in `Released` state with the data on disk.
</details>

## 🧹 Cleanup

```bash
kubectl config set-context --current --namespace=default
kubectl delete namespace lab-12
kubectl get pv     # delete any Released PVs you created: kubectl delete pv <name>
```

➡️ Next: [📝 EXERCISES.md](EXERCISES.md)
