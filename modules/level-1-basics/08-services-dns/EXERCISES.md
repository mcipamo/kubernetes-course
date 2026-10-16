# 📝 Exercises 08 · Services and DNS

## 🟢 Exercise 1 · Write it from scratch
Write a ClusterIP Service `web` that sends port 80 to the named container port `http` on Pods with `app=web,tier=frontend`.

<details><summary>Show solution</summary>

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector:
    app: web
    tier: frontend
  ports:
    - name: http
      port: 80
      targetPort: http
```
</details>

## 🟢 Exercise 2 · Pick the type
Which Service type fits each case? (a) Frontend calls the orders API inside the cluster. (b) A
StatefulSet of Kafka brokers that must be addressed one by one. (c) In-cluster apps must use the
managed DB `prod-db.abc.rds.amazonaws.com` by a short name. (d) A single TCP service must be reachable
from the Internet on a cloud cluster.

<details><summary>Show solution</summary>

(a) ClusterIP · (b) Headless · (c) ExternalName · (d) LoadBalancer
</details>

## 🟡 Exercise 3 · Source IP
What does `externalTrafficPolicy: Local` change for a NodePort/LoadBalancer Service? What is the risk?

<details><summary>Show solution</summary>

Traffic arriving at a node goes only to Pods **on that node**, without a second hop, so the Pod sees
the **real client IP** (no SNAT). Risk: nodes without a local Pod drop the traffic (cloud LBs use the
health-check node port to avoid them) and load can become uneven if Pods are not spread evenly.
</details>

## 🟡 Exercise 4 · ndots
Inside a Pod, how many DNS queries can `curl http://api.github.com` generate with the default
`ndots:5`? How do you reduce them?

<details><summary>Show solution</summary>

`api.github.com` has 2 dots (< 5), so the resolver first tries every search suffix:
`api.github.com.<ns>.svc.cluster.local`, `.svc.cluster.local`, `.cluster.local` (and any node search
domains), each for A and AAAA, before the absolute name — often 6–8+ queries. Fixes: use a trailing dot
(`api.github.com.`), or lower `ndots` in the Pod spec:
```yaml
spec:
  dnsConfig:
    options:
      - name: ndots
        value: "2"
```
</details>

## 🔴 Challenge · Zero-downtime Service migration
You must rename Service `orders` to `orders-api` without breaking any client during the migration. Describe the steps.

<details><summary>Show solution</summary>

1. Create `orders-api` with the **same selector and ports** (both Services now point at the same Pods).
2. Update clients (config / env) to `orders-api` and roll them out.
3. Watch traffic on the old name (CoreDNS query logs or mesh/proxy metrics) until it reaches zero.
4. Optionally turn `orders` into an `ExternalName` pointing to `orders-api.<ns>.svc.cluster.local` for stragglers.
5. Delete `orders`.

Because Services are just selectors + rules, two Services can safely share the same backends.
</details>

---

## ❓ Quiz

1. Which Pods receive new connections from a Service?
   a) The oldest Pod b) All matching Pods c) Pods on the same node d) **Matching Pods that are Ready**

2. What does a headless Service return in DNS?
   a) A ClusterIP b) A CNAME c) Nothing d) **The IPs of the backing Pods**

3. Which component turns a Service IP into a Pod IP on each node?
   a) CoreDNS b) kubelet c) **kube-proxy (or a CNI replacement)** d) The scheduler

4. NodePorts are allocated by default in the range:
   a) **30000–32767** b) Any c) 1–1024 d) 8000–9000

5. On kind, a `LoadBalancer` Service shows `<pending>` because:
   a) It is broken b) NodePort is disabled c) DNS fails d) **There is no cloud controller providing load balancers**

<details><summary>Show answers</summary>

1-d · 2-d · 3-c · 4-a · 5-d
</details>

➡️ Next module: [09 · ConfigMaps and Secrets](../09-configmaps-secrets/)
