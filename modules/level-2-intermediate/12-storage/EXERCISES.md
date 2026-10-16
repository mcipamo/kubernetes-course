# 📝 Exercises 12 · Storage

## 🟢 Exercise 1 · Pick the volume
(a) Scratch space for image processing, 2 GB, deleted with the Pod. (b) A PostgreSQL data directory.
(c) Shared uploads written by 5 web replicas on different nodes. (d) A config file.

<details><summary>Show solution</summary>

(a) `emptyDir` (with `sizeLimit`) or a generic ephemeral volume · (b) PVC, RWO (or RWOP), `Retain`
· (c) PVC with **RWX** on a shared filesystem (or better: object storage like S3) · (d) `configMap` volume
</details>

## 🟢 Exercise 2 · Memory-backed scratch
Write a volume that uses RAM, limited to 64 Mi.

<details><summary>Show solution</summary>

```yaml
volumes:
  - name: fast-scratch
    emptyDir:
      medium: Memory
      sizeLimit: 64Mi
```
It counts against the container's memory limit.
</details>

## 🟡 Exercise 3 · Expand a volume
How do you grow a PVC from 1 Gi to 5 Gi? What must be true first?

<details><summary>Show solution</summary>

The StorageClass must have `allowVolumeExpansion: true` and the CSI driver must support expansion.
```bash
kubectl patch pvc data -p '{"spec":{"resources":{"requests":{"storage":"5Gi"}}}}'
kubectl get pvc data -w
```
Most drivers resize online; some need a Pod restart for the filesystem resize. You cannot shrink a PVC.
</details>

## 🟡 Exercise 4 · Clone a volume
Write a PVC `data-copy` that clones PVC `data`.

<details><summary>Show solution</summary>

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-copy
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: standard
  dataSource:
    kind: PersistentVolumeClaim
    name: data
  resources:
    requests:
      storage: 1Gi
```
The driver must support cloning, and both PVCs must be in the same namespace and StorageClass.
</details>

## 🔴 Challenge · Backup strategy
Design a backup and restore strategy for a PostgreSQL StatefulSet on Kubernetes, with an RPO of 15
minutes and an RTO of 1 hour.

<details><summary>Show solution</summary>

- **Logical + continuous**: WAL archiving to object storage every few minutes (RPO ≤ 15 min) plus a
  nightly base backup — operators like CloudNativePG or Zalando do this natively.
- **Volume level**: scheduled CSI VolumeSnapshots (group snapshots for multiple volumes) for fast restore.
- **Cluster level**: Velero for the Kubernetes objects (StatefulSet, Secrets, Services).
- **Store backups outside the cluster** (different account/region), encrypted, immutable.
- **Test restores** monthly into a scratch namespace and measure the time (RTO).
</details>

---

## ❓ Quiz

1. Which object is a *request* for storage made by a user?

   - a) PVC
   - b) CSIDriver
   - c) StorageClass
   - d) PV

2. `WaitForFirstConsumer` means…

   - a) The volume is provisioned after the Pod is scheduled
   - b) Nothing
   - c) Pods wait for the volume to fill
   - d) The PVC waits for an admin

3. Which access mode allows only a single Pod to mount the volume?

   - a) RWOP
   - b) RWO
   - c) RWX
   - d) ROX

4. With `reclaimPolicy: Delete`, deleting the PVC…

   - a) Keeps the data
   - b) Moves it
   - c) Snapshots it
   - d) Deletes the PV and the underlying storage

5. `emptyDir` data survives…

   - a) Container restarts within the same Pod
   - b) Node failure
   - c) Everything
   - d) Pod deletion

<details><summary>Show answers</summary>

1. **a)** PVC
2. **a)** The volume is provisioned after the Pod is scheduled
3. **a)** RWOP
4. **d)** Deletes the PV and the underlying storage
5. **a)** Container restarts within the same Pod
</details>

➡️ Next module: [13 · StatefulSets, DaemonSets, Jobs and CronJobs](../13-statefulsets-daemonsets-jobs/)
