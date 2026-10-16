# 📝 Exercises 15 · Scheduling

## 🟢 Exercise 1 · Read the message
`0/5 nodes are available: 2 node(s) had untolerated taint {dedicated: gpu}, 3 Insufficient memory.`
What are your options?

<details><summary>Show solution</summary>

The Pod has no toleration for the 2 GPU nodes, and the other 3 lack free **requested** memory. Options:
lower the Pod's memory request (if it is oversized), free capacity (scale down other workloads), add
nodes (cluster autoscaler), or — only if appropriate — tolerate the GPU taint.
</details>

## 🟢 Exercise 2 · One per node
Make sure no two replicas of `api` run on the same node (hard rule).

<details><summary>Show solution</summary>

```yaml
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels: { app: api }
        topologyKey: kubernetes.io/hostname
```
Replicas beyond the number of nodes will stay `Pending`.
</details>

## 🟡 Exercise 3 · Co-locate with a cache
Prefer (soft) placing `api` Pods on the **same node** as a `redis` Pod.

<details><summary>Show solution</summary>

```yaml
affinity:
  podAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels: { app: redis }
          topologyKey: kubernetes.io/hostname
```
</details>

## 🟡 Exercise 4 · Tolerate a node failure for longer
By default Pods are evicted 300 s after their node becomes unreachable. Make a Pod wait 10 minutes instead.

<details><summary>Show solution</summary>

```yaml
tolerations:
  - key: node.kubernetes.io/unreachable
    operator: Exists
    effect: NoExecute
    tolerationSeconds: 600
  - key: node.kubernetes.io/not-ready
    operator: Exists
    effect: NoExecute
    tolerationSeconds: 600
```
Useful for stateful Pods whose restart is expensive; harmful for stateless services (slower failover).
</details>

## 🔴 Challenge · Design a placement policy
A cluster has 3 zones, general nodes and GPU nodes. Design the placement for: (1) a 9-replica web
frontend, (2) a GPU inference service, (3) nightly batch jobs that must never disturb (1) or (2).

<details><summary>Show solution</summary>

1. **Frontend**: topology spread by zone (`maxSkew: 1`, `ScheduleAnyway` or `DoNotSchedule` + `minDomains: 3`)
   and by hostname (`ScheduleAnyway`); PriorityClass `high`; PDB (module 28).
2. **GPU inference**: GPU nodes tainted `nvidia.com/gpu=present:NoSchedule`; the service tolerates it and
   uses node affinity to GPU nodes (or DRA device requests, module 27); PriorityClass `high`.
3. **Batch**: PriorityClass `low` (`preemptionPolicy: Never` so they never preempt others), node
   affinity to general nodes, not tolerating the GPU taint; optionally a dedicated tainted batch pool.
</details>

---

## ❓ Quiz

1. A taint with effect `NoExecute`…
   a) Is ignored by DaemonSets b) **Also evicts running Pods without a matching toleration** c) Deletes the node d) Only affects new Pods

2. A toleration…
   a) Sets priority b) Removes the taint c) **Allows the Pod to be scheduled on the tainted node** d) Forces the Pod onto the tainted node

3. Which field defines "the same place" for affinity and spread rules?
   a) `weight` b) **`topologyKey`** c) `maxSkew` d) `labelSelector`

4. What does the scheduler compare against free capacity during filtering?
   a) Priority b) **Requests** c) Usage d) Limits

5. When can preemption happen?
   a) Every hour b) **When a higher-priority Pod cannot be scheduled** c) When a Pod is slow d) On node reboot

<details><summary>Show answers</summary>

1-b · 2-c · 3-b · 4-b · 5-b
</details>

🎯 **End of Level 2.** Do **Practice round 2** (CKA style, part 1): 8 tasks in 60 minutes covering modules 10–15.

➡️ Next module: [16 · Autoscaling](../../level-3-advanced/16-autoscaling/)
