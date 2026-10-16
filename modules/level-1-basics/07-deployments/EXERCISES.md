# 📝 Exercises 07 · ReplicaSets and Deployments

## 🟢 Exercise 1 · Self-healing
Delete one Pod of `web` and time how long until there are 4 Pods again.

<details><summary>Show solution</summary>

```bash
kubectl delete $(kubectl get pod -l app=web,track=stable -o name | head -1) --wait=false
kubectl get pods -l app=web,track=stable -w
```
Usually 1–3 seconds. The ReplicaSet controller noticed 3 < 4 and created a new Pod.
</details>

## 🟢 Exercise 2 · Rolling update math
`replicas: 10`, `maxSurge: 25%`, `maxUnavailable: 25%`. What is the maximum number of Pods during the
rollout, and the minimum number available?

<details><summary>Show solution</summary>

- maxSurge 25% of 10 = 2.5 → rounds **up** to 3 → up to **13** Pods.
- maxUnavailable 25% of 10 = 2.5 → rounds **down** to 2 → at least **8** available.
</details>

## 🟡 Exercise 3 · Batch changes
Change the image **and** add an env var `LOG_LEVEL=debug` in a single rollout (one new revision only).

<details><summary>Show solution</summary>

```bash
kubectl rollout pause deployment/web
kubectl set image deployment/web web=ghcr.io/stefanprodan/podinfo:6.7.0
kubectl set env deployment/web LOG_LEVEL=debug
kubectl rollout resume deployment/web
kubectl rollout history deployment/web     # only one new revision
```
</details>

## 🟡 Exercise 4 · Recreate strategy
When is `Recreate` the right choice? Change `web` to use it and observe the downtime in the traffic log.

<details><summary>Show solution</summary>

Use it when two versions **must not** run at the same time: incompatible DB migrations, a singleton
that holds a lock, or a `ReadWriteOnce` volume that only one Pod can mount.
```bash
kubectl patch deployment web -p '{"spec":{"strategy":{"type":"Recreate","rollingUpdate":null}}}'
kubectl set image deployment/web web=ghcr.io/stefanprodan/podinfo:6.6.0
```
The traffic log shows `ERROR` lines while no Pod is Ready.
</details>

## 🔴 Challenge · Automatic rollback in CI
Write a bash snippet for a pipeline that deploys a new image, waits up to 2 minutes, and rolls back
automatically (and fails the pipeline) if the rollout does not succeed.

<details><summary>Show solution</summary>

```bash
set -euo pipefail
IMAGE="$1"
kubectl set image deployment/web web="$IMAGE"
kubectl annotate deployment/web kubernetes.io/change-cause="CI deploy $IMAGE" --overwrite
if ! kubectl rollout status deployment/web --timeout=120s; then
  echo "❌ Rollout failed, rolling back"
  kubectl rollout undo deployment/web
  kubectl rollout status deployment/web --timeout=120s
  exit 1
fi
echo "✅ Deployed $IMAGE"
```
In module 24 you will replace this with GitOps and progressive delivery tools (Argo Rollouts / Flagger).
</details>

---

## ❓ Quiz

1. What does a Deployment directly manage?
   a) Nodes b) Pods c) **ReplicaSets** d) Services

2. Which change creates a new revision?
   a) `replicas: 5` b) **A new container image** c) A new label on the Deployment itself d) Scaling to 0

3. With `maxUnavailable: 0` and a broken new image, users…
   a) See errors b) **Keep being served by the old Pods** c) Get redirected d) Lose all Pods

4. After `progressDeadlineSeconds`, Kubernetes…
   a) Rolls back automatically b) Deletes the Deployment c) Restarts nodes d) **Marks the rollout as failed (`ProgressDeadlineExceeded`)**

5. A native, instant traffic switch between two versions is done by…
   a) Scaling to 0 b) Editing Pods c) `rollout undo` d) **Changing the Service selector (blue/green)**

<details><summary>Show answers</summary>

1-c · 2-b · 3-b · 4-d · 5-d
</details>

➡️ Next module: [08 · Services and DNS](../08-services-dns/)
