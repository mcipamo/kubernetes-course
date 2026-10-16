# 📝 Exercises 04 · Kubernetes Architecture

## 🟢 Exercise 1 · Who did it?
For each event, name the component that produced it: `Scheduled`, `Pulled`, `ScalingReplicaSet`,
`SuccessfulCreate`, `NodeNotReady`.

<details><summary>Show solution</summary>

| Event | Component |
|---|---|
| `Scheduled` | kube-scheduler |
| `Pulled` | kubelet |
| `ScalingReplicaSet` | Deployment controller (kube-controller-manager) |
| `SuccessfulCreate` | ReplicaSet controller (kube-controller-manager) |
| `NodeNotReady` | Node lifecycle controller (kube-controller-manager) |
</details>

## 🟢 Exercise 2 · Versions
Find the version of the API server and of the kubelet on each node.

<details><summary>Show solution</summary>

```bash
kubectl version
kubectl get nodes -o custom-columns=NODE:.metadata.name,KUBELET:.status.nodeInfo.kubeletVersion,RUNTIME:.status.nodeInfo.containerRuntimeVersion
```
</details>

## 🟡 Exercise 3 · Kill the controller manager
Repeat the 💥 step of the lab but with `kube-controller-manager.yaml`. Then delete one Pod of a
Deployment. What happens? Why?

<details><summary>Show solution</summary>

The Pod is deleted and **not replaced**: the ReplicaSet controller that would notice "2 < 3" lives in
the controller manager. When you restore the manifest, the controller starts, sees the difference and
creates the missing Pod — level-triggered reconciliation in action.
</details>

## 🟡 Exercise 4 · Static Pod
Create a static Pod called `static-web` on `kc-worker2` by placing a manifest in its
`/etc/kubernetes/manifests`. Try to delete it with `kubectl`. What happens?

<details><summary>Show solution</summary>

```bash
docker exec kc-worker2 mkdir -p /etc/kubernetes/manifests
docker exec -i kc-worker2 tee /etc/kubernetes/manifests/static-web.yaml >/dev/null <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: static-web
spec:
  containers:
    - name: web
      image: nginx:1.27
EOF
kubectl get pods -A | grep static-web   # static-web-kc-worker2 (a "mirror Pod")
kubectl delete pod static-web-kc-worker2
kubectl get pods -A | grep static-web   # it comes back
docker exec kc-worker2 rm /etc/kubernetes/manifests/static-web.yaml
```
The API only holds a read-only **mirror Pod**. The kubelet owns it, so only removing the file deletes it.
</details>

## 🔴 Challenge · Draw it yourself
Without looking, draw the sequence diagram for `kubectl scale deployment web --replicas=5` from 3
replicas. Include which objects change in etcd and which components watch them.

<details><summary>Show solution</summary>

1. kubectl → API server: PATCH `deployments/web/scale` (replicas 5) → etcd.
2. Deployment controller (watch Deployments) → updates the ReplicaSet `replicas: 5`.
3. ReplicaSet controller (watch ReplicaSets) → creates 2 new Pod objects with no `nodeName`.
4. Scheduler (watch unscheduled Pods) → `binding` for each Pod.
5. kubelet on each chosen node (watch Pods for its node) → starts containers → updates Pod status.
6. EndpointSlice controller → adds the new Pod IPs once they are Ready (module 08).
</details>

---

## ❓ Quiz

1. Which component is the only one that talks to etcd?
   a) **kube-apiserver** b) kubelet c) scheduler d) kube-proxy

2. If the control plane goes down, running Pods…
   a) **Keep running** b) Are deleted c) Move to another node d) Restart

3. A Pod is `Pending` with **no events**. The most likely cause is:
   a) Failing probe b) **The scheduler is not running** c) Bad image d) Not enough CPU

4. What does "level-triggered" mean for controllers?
   a) They react only to events b) They run on a timer c) **They act on the current state, whatever the event history** d) They have priority levels

5. How does a node tell the control plane it is alive?
   a) Writing to etcd directly b) ICMP ping c) **Renewing its Lease in `kube-node-lease`** d) SSH

<details><summary>Show answers</summary>

1-a · 2-a · 3-b · 4-c · 5-c
</details>

➡️ Next module: [05 · Pods](../05-pods/)
