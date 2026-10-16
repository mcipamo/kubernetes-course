# 📝 Exercises 10 · Resources, QoS and Quotas

## 🟢 Exercise 1 · Name the QoS
Give the QoS class: (a) requests cpu 100m/mem 128Mi, limits cpu 100m/mem 128Mi. (b) Only limits cpu
200m/mem 256Mi. (c) requests mem 128Mi only. (d) Two containers: one Guaranteed-style, one with nothing.

<details><summary>Show solution</summary>

(a) **Guaranteed**. (b) **Guaranteed** — when only limits are set, requests default to the limits.
(c) **Burstable**. (d) **Burstable** — every container must qualify for the Pod to be Guaranteed.
</details>

## 🟢 Exercise 2 · Units
Convert: `0.5` CPU, `1500m` CPU, `1Gi`, `1G`.

<details><summary>Show solution</summary>

`0.5` = `500m` · `1500m` = 1.5 cores · `1Gi` = 1,073,741,824 bytes · `1G` = 1,000,000,000 bytes (≈ 7% less than `1Gi`).
</details>

## 🟡 Exercise 3 · Right-size from data
A service shows p50 = 120m CPU / 300Mi, p95 = 400m / 380Mi, peak memory 450Mi. Propose requests and limits.

<details><summary>Show solution</summary>

One reasonable answer:
```yaml
resources:
  requests: { cpu: 400m, memory: 400Mi }   # p95 CPU; memory above p95
  limits:   { memory: 600Mi }              # ~30% above peak; no CPU limit to avoid throttling
```
Revisit after a week of real traffic. VPA recommendations (module 16) automate this analysis.
</details>

## 🟡 Exercise 4 · Quota per tier
Create a quota that limits the namespace to 2 `LoadBalancer` Services, 5 PVCs and 20 Gi of total storage requests.

<details><summary>Show solution</summary>

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: objects
spec:
  hard:
    services.loadbalancers: "2"
    persistentvolumeclaims: "5"
    requests.storage: 20Gi
```
</details>

## 🔴 Challenge · JVM in a container
A Java app with `limits.memory: 1Gi` is OOMKilled under load, but the heap dashboard shows only 700 MB
used. Explain and fix.

<details><summary>Show solution</summary>

The container limit covers **all** process memory: heap + metaspace + thread stacks + code cache +
direct buffers + GC overhead. 700 MB heap can easily mean > 1 GiB in total. Fixes:
- Size the heap as a percentage: `-XX:MaxRAMPercentage=70` (modern JVMs read the cgroup limit).
- Raise `limits.memory` to match the measured total RSS (`container_memory_working_set_bytes`).
- Cap thread counts and direct memory (`-XX:MaxDirectMemorySize`).
</details>

---

## ❓ Quiz

1. What does the scheduler compare against node allocatable?

   - a) Limits
   - b) QoS
   - c) Requests
   - d) Current usage

2. A container exceeds its CPU limit. It is…

   - a) Evicted
   - b) OOMKilled
   - c) Throttled
   - d) Restarted

3. Under memory pressure, which QoS class is evicted first?

   - a) BestEffort
   - b) Burstable
   - c) Random
   - d) Guaranteed

4. Where do you look when a Deployment has fewer replicas than desired but no Pod is Pending?

   - a) Service
   - b) CoreDNS logs
   - c) Node events
   - d) ReplicaSet events (quota/admission errors)

5. Which object injects default requests into containers that do not set them?

   - a) ResourceQuota
   - b) LimitRange
   - c) HPA
   - d) PriorityClass

<details><summary>Show answers</summary>

1. **c)** Requests
2. **c)** Throttled
3. **a)** BestEffort
4. **d)** ReplicaSet events (quota/admission errors)
5. **b)** LimitRange
</details>

➡️ Next module: [11 · Probes and Pod Lifecycle](../11-probes-lifecycle/)
