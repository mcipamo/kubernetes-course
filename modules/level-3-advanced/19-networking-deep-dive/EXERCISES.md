# 📝 Exercises 19 · Networking Deep Dive

## 🟢 Exercise 1 · Capacity from CIDRs
A cluster uses Pod CIDR `10.244.0.0/16` and gives each node a `/24`. How many nodes can it have, and
roughly how many Pods per node?

<details><summary>Show solution</summary>

`/16` split into `/24`s → 2⁸ = **256 nodes**. Each `/24` has 256 addresses → ~**254 usable Pod IPs** per
node (the kubelet default `maxPods` is 110, so that is usually the real limit).
</details>

## 🟢 Exercise 2 · Which mode?
How do you find which kube-proxy mode your cluster uses?

<details><summary>Show solution</summary>

```bash
kubectl -n kube-system get cm kube-proxy -o jsonpath='{.data.config\.conf}' | grep -i mode
kubectl -n kube-system logs -l k8s-app=kube-proxy | grep -i 'Using .* Proxier'
```
An empty `mode` means the platform default (iptables on Linux unless configured otherwise).
</details>

## 🟡 Exercise 3 · Dual-stack Service
Write a Service that prefers dual-stack and lists IPv6 first.

<details><summary>Show solution</summary>

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  ipFamilyPolicy: PreferDualStack
  ipFamilies: [IPv6, IPv4]
  selector: { app: web }
  ports:
    - port: 80
```
On a single-stack cluster it falls back to the available family.
</details>

## 🟡 Exercise 4 · MTU math
Node interfaces have MTU 1500. Your overlay uses VXLAN (50 bytes of overhead). What should the Pod MTU
be, and what symptom do you get if it is left at 1500?

<details><summary>Show solution</summary>

**1450** (1500 − 50). If Pods use 1500, large packets exceed the node MTU after encapsulation and get
dropped or fragmented; if ICMP "fragmentation needed" is blocked, TCP connections **hang on large
responses** (TLS handshakes with big certificates, large JSON) while small requests work.
</details>

## 🔴 Challenge · Mesh or not?
Your company asks for: mTLS between all services, per-route retries, and golden-signal metrics for 80
services. Compare (a) doing it in each app, (b) a sidecar mesh, (c) a sidecar-less mesh.

<details><summary>Show solution</summary>

| | App libraries | Sidecar mesh | Sidecar-less mesh |
|---|---|---|---|
| mTLS | Per-language work, hard to keep consistent | Automatic | Automatic (L4 per node) |
| Retries / routing | Code changes per app | Config (HTTPRoute/VirtualService) | Needs L7 waypoints |
| Metrics | Per-app instrumentation | Uniform per proxy | Uniform; L7 only with waypoints |
| Overhead | Lowest | CPU/memory per Pod, more latency | Lower per Pod, shared proxies |
| Operational cost | Distributed across teams | Mesh control plane + upgrades | Same, newer tech |

For 80 services, a mesh usually wins on consistency; sidecar-less reduces cost if its maturity fits
your risk appetite. Either way, start with mTLS + metrics, add L7 policies gradually.
</details>

---

## ❓ Quiz

1. In the Kubernetes network model, Pod-to-Pod traffic across nodes…
   a) Needs a Service b) **Uses Pod IPs without NAT** c) Goes through the API server d) Is always NATed

2. Which component assigns a Pod its IP address?
   a) kube-proxy b) CoreDNS c) The scheduler d) **The CNI plugin (IPAM)**

3. Where is the Service IP translated to a Pod IP in iptables/nftables mode?
   a) In CoreDNS b) On the destination node c) In etcd d) **On the node where the client Pod runs**

4. Why does one keep-alive connection always reach the same Pod?
   a) Session affinity default b) DNS caching c) Readiness d) **conntrack keeps the NAT decision for the connection**

5. Which API lets you add Service IP ranges without rebuilding the cluster?
   a) EndpointSlice b) IPPool c) Ingress d) **ServiceCIDR**

<details><summary>Show answers</summary>

1-b · 2-d · 3-d · 4-d · 5-d
</details>

➡️ Next module: [20 · Observability](../20-observability/)
