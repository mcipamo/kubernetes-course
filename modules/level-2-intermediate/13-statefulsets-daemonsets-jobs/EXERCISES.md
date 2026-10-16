# 📝 Exercises 13 · StatefulSets, DaemonSets, Jobs and CronJobs

## 🟢 Exercise 1 · Pick the controller
(a) Prometheus node-exporter. (b) A stateless REST API. (c) A three-node etcd cluster. (d) Resize
10,000 images once. (e) Rotate logs every hour.

<details><summary>Show solution</summary>

(a) DaemonSet · (b) Deployment · (c) StatefulSet · (d) Job (Indexed, parallel) · (e) CronJob
</details>

## 🟢 Exercise 2 · Name the PVCs
A StatefulSet `kafka` with 3 replicas has a `volumeClaimTemplates` entry named `logs`. What are the PVC names?

<details><summary>Show solution</summary>

`logs-kafka-0`, `logs-kafka-1`, `logs-kafka-2` → pattern `<template-name>-<statefulset-name>-<ordinal>`.
</details>

## 🟡 Exercise 3 · Only on some nodes
Run a DaemonSet only on nodes labeled `disk=ssd`. Label one worker and verify.

<details><summary>Show solution</summary>

```yaml
spec:
  template:
    spec:
      nodeSelector:
        disk: ssd
```
```bash
kubectl label node kc-worker disk=ssd
kubectl get ds        # DESIRED = 1
```
</details>

## 🟡 Exercise 4 · CronJob safety
A CronJob runs every 5 minutes, but sometimes a run takes 8 minutes. Two runs at once corrupt data.
Which fields do you set?

<details><summary>Show solution</summary>

```yaml
spec:
  concurrencyPolicy: Forbid           # skip the new run while one is active
  startingDeadlineSeconds: 120
  jobTemplate:
    spec:
      activeDeadlineSeconds: 900      # kill a run that hangs beyond 15 min
```
Also make the job idempotent and alert on the last successful run time.
</details>

## 🔴 Challenge · Scale down a StatefulSet safely
You scale a 5-replica database StatefulSet to 3. What happens to Pods and PVCs? What must the app do
first? How do you get the disks back later?

<details><summary>Show solution</summary>

- Pods `db-4` then `db-3` are terminated (highest ordinal first).
- With the default `whenScaled: Retain`, PVCs `data-db-3` and `data-db-4` **remain** (and keep costing money).
- **Before** scaling down, the app must move data/leadership off those members (decommission brokers,
  remove raft members), or you lose replicas of the data.
- Scaling back up to 5 re-attaches the same PVCs to `db-3`/`db-4`. If you never will, delete the PVCs
  (after backups) or set `whenScaled: Delete`.
</details>

---

## ❓ Quiz

1. Which controller gives each replica its own PVC automatically?

   - a) StatefulSet
   - b) Job
   - c) Deployment
   - d) DaemonSet

2. A Pod of StatefulSet `web` (headless Service `web`, namespace `prod`) is reachable at:

   - a) `web-0.prod`
   - b) `web-0.web.prod.svc.cluster.local`
   - c) `0.web.prod`
   - d) `web.prod.svc.cluster.local`

3. Which `restartPolicy` values are allowed in a Job template?

   - a) Only Never
   - b) Never or OnFailure
   - c) Any
   - d) Always

4. A CronJob with `concurrencyPolicy: Replace`…

   - a) Cancels the running Job and starts the new one
   - b) Skips new runs
   - c) Runs them in parallel
   - d) Fails

5. How do you trigger a CronJob immediately?

   - a) Edit the schedule to `now`
   - b) `kubectl create job x --from=cronjob/name`
   - c) `kubectl run cronjob`
   - d) Restart the controller

<details><summary>Show answers</summary>

1. **a)** StatefulSet
2. **b)** `web-0.web.prod.svc.cluster.local`
3. **b)** Never or OnFailure
4. **a)** Cancels the running Job and starts the new one
5. **b)** `kubectl create job x --from=cronjob/name`
</details>

➡️ Next module: [14 · Ingress and Gateway API](../14-ingress-gateway-api/)
