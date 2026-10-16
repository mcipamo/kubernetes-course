<img src="../../../assets/kubernetes-icon-color.svg" alt="Kubernetes logo" width="56" align="right">

# 08 · Services and DNS

| Level | Duration | Prerequisites |
|---|---|---|
| 🧱 Basics | 6 h | Modules 06–07 |

**By the end you will be able to:**
- **Explain** why Pods need Services and how a virtual IP reaches changing Pods.
- **Choose** between ClusterIP, NodePort, LoadBalancer, ExternalName and headless Services.
- **Describe** how EndpointSlices, kube-proxy and CoreDNS work together.
- **Debug** a Service that "does not work" with a repeatable checklist.

---

## 🧠 The analogy: a company switchboard

Pods are employees who change desks — and phone extensions — every time they come back from lunch
(new Pod, new IP). Calling them directly is hopeless.

A **Service** is the **company switchboard number**: `555-0100 Support`. It never changes. When you
call it, the operator (kube-proxy) looks at the **list of who is at their desk right now**
(EndpointSlices — only Ready Pods) and connects you to one of them.

**DNS** is the **company directory**: you look up "support" and get `555-0100`. You never need to
memorize numbers.

---

## 📖 Concepts

### 1. The problem

Pods are ephemeral and their IPs change. Clients need a **stable name and IP** that always lead to a
healthy backend, and some load balancing across replicas.

### 2. Anatomy of a Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api
spec:
  type: ClusterIP              # default
  selector:                    # which Pods are backends (by label)
    app: api
  ports:
    - name: http               # name ports: Ingress/Gateway and probes can refer to them
      port: 80                 # the port clients use on the Service IP
      targetPort: http         # the container port (number or the port's name)
      protocol: TCP
```

### 3. Service types

| Type | Reachable from | How | Typical use |
|---|---|---|---|
| **ClusterIP** | Inside the cluster | Virtual IP from the Service CIDR | Service-to-service traffic (most Services) |
| **NodePort** | `<any-node-IP>:30000-32767` | ClusterIP + a port on every node | Labs, bare metal, behind an external LB |
| **LoadBalancer** | The Internet / your network | NodePort + a cloud load balancer | Exposing a single service publicly |
| **ExternalName** | Inside the cluster | DNS `CNAME` to an external name | Aliasing an external DB (`db.example.com`) |
| **Headless** (`clusterIP: None`) | Inside the cluster | DNS returns Pod IPs directly; no virtual IP | StatefulSets, client-side load balancing |

> For HTTP routing (hosts, paths, TLS) you put an **Ingress** or **Gateway** in front of ClusterIP
> Services instead of one LoadBalancer per app (module 14).

### 4. EndpointSlices: the live backend list

The **EndpointSlice controller** watches Services and Pods. For every Pod matching the selector it
records IP, port, node, zone and conditions (`ready`, `serving`, `terminating`). Only **Ready** Pods
receive new traffic. (The older `Endpoints` API is deprecated in favor of EndpointSlices.)

### 5. kube-proxy: how a virtual IP works

The Service IP is not on any network interface. **kube-proxy** on each node watches Services and
EndpointSlices and programs the kernel (iptables, **nftables** or IPVS) to rewrite packets: "traffic to
`10.96.12.34:80` → pick one of `10.244.1.7:9898`, `10.244.2.4:9898`". Load balancing is per
*connection*, not per request: a long-lived HTTP/2 or gRPC connection sticks to one Pod.

### 6. Useful Service options

| Field | Effect |
|---|---|
| `sessionAffinity: ClientIP` | Same client IP → same Pod |
| `externalTrafficPolicy: Local` | Preserve client source IP; only nodes with local Pods receive traffic |
| `internalTrafficPolicy: Local` | In-cluster traffic stays on the same node |
| `trafficDistribution: PreferClose` | Prefer endpoints in the same zone (saves cross-zone cost and latency) |
| `publishNotReadyAddresses: true` | Include not-ready Pods (for peer discovery in StatefulSets) |

### 7. DNS in Kubernetes (CoreDNS)

| Record | Name | Resolves to |
|---|---|---|
| Service | `api.shop.svc.cluster.local` | ClusterIP |
| Headless Service | `db.shop.svc.cluster.local` | All Ready Pod IPs |
| StatefulSet Pod | `db-0.db.shop.svc.cluster.local` | That Pod's IP |
| SRV | `_http._tcp.api.shop.svc.cluster.local` | Port + name |

Each Pod's `/etc/resolv.conf` has a `search` list (`shop.svc.cluster.local svc.cluster.local
cluster.local`) and `ndots:5`, so short names like `api` work. Side effect: a name with fewer than 5
dots (like `api.example.com`) is first tried with every search suffix — use a trailing dot
(`api.example.com.`) for external names in latency-sensitive apps.

---

## 🗺️ Diagram

```mermaid
flowchart LR
    C["Client Pod"] -->|"1. DNS: api.shop?"| DNS["CoreDNS"]
    DNS -->|"2. 10.96.12.34"| C
    C -->|"3. connect 10.96.12.34:80"| KP{{"kube-proxy rules<br/>(iptables / nftables / IPVS)<br/>on the client's node"}}
    KP -->|"4. DNAT"| P1["Pod api-1<br/>10.244.1.7:9898 ✅ Ready"]
    KP --> P2["Pod api-2<br/>10.244.2.4:9898 ✅ Ready"]
    KP -.x.- P3["Pod api-3<br/>❌ Not Ready"]
    EPC["EndpointSlice controller"] -->|watches Pods + Service| ES[("EndpointSlice api-xxxx")]
    ES --> KP
```

---

## ⌨️ Essential commands

| Command | What it does |
|---|---|
| `kubectl expose deployment api --port=80 --target-port=9898` | Create a ClusterIP Service |
| `kubectl get svc -o wide` | Type, IP, ports and **selector** |
| `kubectl get endpointslices -l kubernetes.io/service-name=api` | The live backends |
| `kubectl describe svc api` | Selector, endpoints, events |
| `kubectl run t --image=busybox:1.36 --rm -it --restart=Never -- nslookup api` | DNS test |
| `kubectl port-forward svc/api 8080:80` | Reach a Service from your laptop |
| `kubectl -n kube-system logs -l k8s-app=kube-dns` | CoreDNS logs |

**The "my Service does not work" checklist:**

```bash
kubectl get svc api -o wide                                    # 1. Does it exist? Which selector?
kubectl get pods -l app=api -o wide                            # 2. Do Pods match that selector? Ready?
kubectl get endpointslices -l kubernetes.io/service-name=api   # 3. Are there endpoints?
kubectl run t --rm -it --image=nicolaka/netshoot --restart=Never -- \
  sh -c 'nslookup api && curl -sv --max-time 3 api:80'         # 4. DNS + connectivity from a Pod
# 5. Is targetPort really the port the container listens on?
```

---

## 🧯 SRE mindset

- **A Service with zero endpoints is an outage**. Alert on `kube_endpoint_address_available == 0` (or the EndpointSlice equivalent).
- **Readiness controls traffic.** A wrong readiness probe can take all endpoints out at once (module 11).
- **Long-lived connections defeat L4 load balancing.** For gRPC/HTTP2, use client-side balancing (headless) or an L7 proxy/mesh.
- **DNS is a shared dependency.** Scale CoreDNS, consider NodeLocal DNSCache, and watch for `ndots` amplification.
- **Prefer topology-aware routing** in multi-zone clusters to cut latency and cross-zone cost.

## ⚠️ Common mistakes

| Symptom | Cause | Fix |
|---|---|---|
| `Connection refused` | `targetPort` ≠ container port | Align `targetPort` with the listening port |
| Timeout / no endpoints | Selector does not match Pod labels, or Pods not Ready | Fix labels; check readiness |
| `NXDOMAIN` | Wrong namespace or typo | Use `name.namespace` |
| LoadBalancer `EXTERNAL-IP <pending>` forever | No cloud controller (e.g., kind) | Use cloud-provider-kind / MetalLB, or NodePort/port-forward |
| Uneven load | Long-lived connections | Client-side balancing or L7 proxy |

## 🎓 Official tutorial companion

After the lab, run the upstream tutorial to see the same idea from the official docs:

- [Kubernetes Basics 4 · Expose your app](https://kubernetes.io/docs/tutorials/kubernetes-basics/expose/expose-intro/)
- [Connecting applications with Services](https://kubernetes.io/docs/tutorials/services/connect-applications-service/)
- [Using source IP](https://kubernetes.io/docs/tutorials/services/source-ip/)
- [Exposing an external IP address](https://kubernetes.io/docs/tutorials/stateless-application/expose-external-ip-address/)

> Tutorials that use minikube work on `kc` too; replace `minikube service` with `kubectl port-forward`.

## 📚 Official references

- [Service](https://kubernetes.io/docs/concepts/services-networking/service/)
- [EndpointSlices](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/)
- [Virtual IPs and service proxies](https://kubernetes.io/docs/reference/networking/virtual-ips/)
- [DNS for Services and Pods](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)
- [Debug Services](https://kubernetes.io/docs/tasks/debug/debug-application/debug-service/)
- [Traffic distribution](https://kubernetes.io/docs/concepts/services-networking/service/#traffic-distribution)

➡️ Next: [🧪 LAB.md](LAB.md)
