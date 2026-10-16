# 📝 Exercises 16 · Autoscaling

## 🟢 Exercise 1 · Do the math
3 replicas, current average CPU utilization 120%, target 60%. Desired replicas? And if `maxReplicas: 5`?

<details><summary>Show solution</summary>

`ceil(3 × 120 / 60) = 6`. With `maxReplicas: 5` → **5**, and the condition `ScalingLimited=True` (`TooManyReplicas`).
</details>

## 🟢 Exercise 2 · Memory-based HPA
Write an HPA metric block that targets an average of 400 Mi of memory per Pod. Is memory a good scaling signal?

<details><summary>Show solution</summary>

```yaml
metrics:
  - type: Resource
    resource:
      name: memory
      target:
        type: AverageValue
        averageValue: 400Mi
```
Often **not**: many runtimes (JVM, caches) never release memory, so the HPA scales up and never down.
Prefer CPU, requests/s or queue length unless memory tracks load linearly.
</details>

## 🟡 Exercise 3 · Gentle scale-down
Configure an HPA to scale up as fast as possible, but scale down by at most 10% per minute after 10
minutes of low load.

<details><summary>Show solution</summary>

```yaml
behavior:
  scaleUp:
    stabilizationWindowSeconds: 0
    selectPolicy: Max
    policies:
      - { type: Percent, value: 100, periodSeconds: 15 }
      - { type: Pods, value: 4, periodSeconds: 15 }
  scaleDown:
    stabilizationWindowSeconds: 600
    policies:
      - { type: Percent, value: 10, periodSeconds: 60 }
```
</details>

## 🟡 Exercise 4 · Resize policy
Write a container `resizePolicy` that allows CPU changes without a restart but restarts the container
for memory changes.

<details><summary>Show solution</summary>

```yaml
containers:
  - name: app
    image: my-app:1.0
    resizePolicy:
      - { resourceName: cpu, restartPolicy: NotRequired }
      - { resourceName: memory, restartPolicy: RestartContainer }
```
</details>

## 🔴 Challenge · Queue workers
A worker Deployment processes jobs from a RabbitMQ queue. At night the queue is empty for 8 hours.
Design autoscaling that scales to zero at night and handles a burst of 50,000 messages in the morning.

<details><summary>Show solution</summary>

Use **KEDA** with a `ScaledObject`:
```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: workers
spec:
  scaleTargetRef:
    name: workers
  minReplicaCount: 0
  maxReplicaCount: 50
  cooldownPeriod: 300
  triggers:
    - type: rabbitmq
      metadata:
        queueName: jobs
        mode: QueueLength
        value: "100"            # ~100 messages per replica
      authenticationRef:
        name: rabbitmq-auth
```
Also: set `requests` so the node autoscaler can add nodes for the burst; make workers idempotent; use
graceful shutdown so scale-down does not drop in-flight jobs; cap `maxReplicaCount` by what the
downstream DB can handle. (With the beta `HPAScaleToZero` gate, a plain HPA on an external metric can
also do it ⚠️.)
</details>

---

## ❓ Quiz

1. CPU utilization targets in an HPA are a percentage of…
   a) **The request** b) Node capacity c) The limit d) The average of the cluster

2. Which component provides `metrics.k8s.io` in most clusters?
   a) Prometheus b) kubelet directly c) etcd d) **metrics-server**

3. With several metrics, the HPA chooses…
   a) **The highest recommendation** b) The average c) The lowest recommendation d) The first metric

4. Which tool adds nodes when Pods are Pending for lack of capacity?
   a) **Cluster Autoscaler / Karpenter** b) KEDA c) VPA d) HPA

5. In-place Pod resize uses…
   a) **The `resize` subresource** b) A new Deployment c) `kubectl scale` d) `kubectl edit pod` on `spec.containers`

<details><summary>Show answers</summary>

1-a · 2-d · 3-a · 4-a · 5-a
</details>

➡️ Next module: [17 · Security I: AuthN, AuthZ and RBAC](../17-security-rbac/)
