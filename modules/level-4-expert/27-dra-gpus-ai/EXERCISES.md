# 📝 Exercises 27 · DRA, GPUs and AI Workloads

## 🟢 Exercise 1 · Device plugin or DRA?
Which fits? (a) "Any 1 GPU" for a simple inference service on a managed cluster that only offers the
device plugin. (b) "A GPU with ≥ 40 Gi memory on the same PCIe root as a fast NIC". (c) Two containers
in one Pod must share one GPU.

<details><summary>Show solution</summary>

(a) Device plugin (`nvidia.com/gpu: 1`) · (b) DRA (attributes, CEL selectors, constraints across devices) ·
(c) DRA (a claim referenced by both containers)
</details>

## 🟢 Exercise 2 · Claim vs template
When do you use a `ResourceClaim` and when a `ResourceClaimTemplate`?

<details><summary>Show solution</summary>

`ResourceClaim`: one concrete allocation, possibly **shared** by several Pods (they must run where the
device is). `ResourceClaimTemplate`: a blueprint that creates **one claim per Pod** — what you want in
Deployments, Jobs and StatefulSets where each replica needs its own device.
</details>

## 🟡 Exercise 3 · Fallback with prioritized list
Write a request that prefers one large GPU (≥ 80 Gi) but accepts two smaller ones (≥ 40 Gi each). ⚠️ Check that `firstAvailable` is enabled in your version.

<details><summary>Show solution</summary>

```yaml
spec:
  devices:
    requests:
      - name: gpu
        firstAvailable:
          - name: big
            deviceClassName: gpu.example.com
            selectors:
              - cel: { expression: "device.capacity['gpu.example.com'].memory.compareTo(quantity('80Gi')) >= 0" }
          - name: two-small
            deviceClassName: gpu.example.com
            count: 2
            selectors:
              - cel: { expression: "device.capacity['gpu.example.com'].memory.compareTo(quantity('40Gi')) >= 0" }
```
The scheduler tries the sub-requests in order and allocates the first one that fits.
</details>

## 🟡 Exercise 4 · GPU node hygiene
List the scheduling settings that keep ordinary workloads off GPU nodes and AI workloads on them.

<details><summary>Show solution</summary>

- Taint GPU nodes: `nvidia.com/gpu=present:NoSchedule`.
- AI workloads tolerate the taint and either request the device (device plugin / DRA claim) or use node affinity to GPU node labels.
- PriorityClasses: inference > training > experiments; preemption policies set deliberately.
- Kueue quotas per team, with borrowing limits.
- Cluster autoscaler / Karpenter node pools dedicated to GPU types, scaling to zero when idle.
</details>

## 🔴 Challenge · Platform for LLM serving
Design the Kubernetes platform for serving three LLMs (one large, two small with LoRA adapters) with a
p95 time-to-first-token SLO of 500 ms.

<details><summary>Show solution</summary>

- **Compute**: GPU node pools per GPU type; large model on whole GPUs (or multi-GPU via DRA claims with
  topology constraints / LeaderWorkerSet for multi-host); small models on MIG slices.
- **Serving**: vLLM (or similar) Deployments per model; LoRA adapters loaded dynamically; model weights
  from a cache (image volumes / pre-pulled images / node-local cache) to cut cold starts.
- **Routing**: Gateway API + **Inference Extension** (InferencePool / model-aware endpoint picking by
  queue depth and KV-cache use); per-model routes; rate limits per tenant.
- **Autoscaling**: KEDA/HPA on queue length or tokens/s, not CPU; warm minimum replicas for the SLO;
  node autoscaling with GPU-aware pools.
- **Reliability**: PDBs, topology spread across zones, readiness tied to "model loaded", canary for new
  model versions with quality + latency analysis.
- **Observability**: TTFT, tokens/s, queue depth, GPU utilization and memory (DCGM), error ratio → SLO
  burn-rate alerts (module 28).
- **Cost**: Kueue for batch fine-tuning on spare capacity; scale experiments to zero.
</details>

---

## ❓ Quiz

1. Which object does a DRA driver publish to describe available devices?
   a) **ResourceSlice** b) ResourceClaim c) Node label d) DeviceClass

2. DRA device selection criteria are written in…
   a) Labels only b) Rego c) **CEL** d) JSONPath

3. With the device plugin model, a GPU request is…
   a) **An integer extended resource like `nvidia.com/gpu: 1`** b) A CRD c) A claim object d) A node selector

4. Gang scheduling means…
   a) Preempting everything b) Using one node c) Scheduling Pods alphabetically d) **Placing all Pods of a group together or none**

5. Which project provides job queueing and quotas for batch/AI workloads?
   a) **Kueue** b) Velero c) CoreDNS d) Argo CD

<details><summary>Show answers</summary>

1-a · 2-c · 3-a · 4-d · 5-a
</details>

➡️ Next module: [28 · SRE Reliability Engineering](../28-sre-reliability/)
