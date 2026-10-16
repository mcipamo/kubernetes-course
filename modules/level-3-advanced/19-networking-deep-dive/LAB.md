# 🧪 Lab 19 · Follow the Packet

| Goal | Duration | Environment |
|---|---|---|
| Walk the real path of a request through veths, routes, NAT rules and conntrack | 120 min | `kc` kind cluster |

## Setup

```bash
kubectl create namespace lab-19
kubectl config set-context --current --namespace=lab-19
kubectl create deployment server --image=ghcr.io/stefanprodan/podinfo:6.7.0 --replicas=1 --port=9898
kubectl expose deployment server --port=80 --target-port=9898
kubectl run client --image=nicolaka/netshoot --overrides='{"spec":{"nodeName":"kc-worker"}}' -- sleep infinity
kubectl rollout status deployment/server; kubectl wait --for=condition=Ready pod/client
kubectl get pods -o wide
```

Note on which node each Pod runs. If `server` is also on `kc-worker`, move it:
`kubectl patch deploy server -p '{"spec":{"template":{"spec":{"nodeName":"kc-worker2"}}}}'`.

## Step 1 · Addresses and CIDRs

```bash
kubectl get nodes -o custom-columns=NAME:.metadata.name,INTERNAL-IP:.status.addresses[0].address,POD-CIDR:.spec.podCIDR
kubectl get svc server -o jsonpath='{.spec.clusterIP}{"\n"}'
kubectl -n kube-system get cm kubeadm-config -o yaml | grep -E 'podSubnet|serviceSubnet'
```

✅ **Check:** each node has its own `/24` inside the Pod subnet, and the Service IP comes from the separate Service subnet.

## Step 2 · Inside the client Pod

```bash
kubectl exec client -- ip -br addr
kubectl exec client -- ip route
```

The Pod has `eth0` with its Pod IP and a default route to the node.

## Step 3 · The other end of the veth, on the node

```bash
CLIENT_IP=$(kubectl get pod client -o jsonpath='{.status.podIP}')
docker exec kc-worker ip route | grep -E "$CLIENT_IP|10.244"
docker exec kc-worker ip -br link | grep veth | head
```

✅ **Check:** the node has a route `CLIENT_IP dev vethXXXX` (the Pod's driveway) and routes to the other
nodes' Pod CIDRs **via their node IPs** — kindnet uses native routing, no tunnel.

## Step 4 · The Service rules

```bash
SVC_IP=$(kubectl get svc server -o jsonpath='{.spec.clusterIP}')
SERVER_IP=$(kubectl get pod -l app=server -o jsonpath='{.items[0].status.podIP}')
echo "Service $SVC_IP → Pod $SERVER_IP"
docker exec kc-worker sh -c "nft list ruleset 2>/dev/null | grep -n '$SVC_IP' | head -5; iptables-save 2>/dev/null | grep '$SVC_IP' | head -5"
```

Depending on the kube-proxy mode you will see an nftables map entry or an iptables `KUBE-SVC-…` chain for
the Service IP that jumps to a `KUBE-SEP-…` (endpoint) rule doing `DNAT --to-destination $SERVER_IP:9898`.

## Step 5 · Capture traffic on the destination node

Terminal 2:

```bash
SERVER_NODE=$(kubectl get pod -l app=server -o jsonpath='{.items[0].spec.nodeName}')
kubectl debug node/$SERVER_NODE -it --image=nicolaka/netshoot -- tcpdump -ni any tcp port 9898 -c 10
```

Terminal 1:

```bash
kubectl exec client -- curl -s server/version
```

✅ **Check:** tcpdump shows packets from `$CLIENT_IP` to `$SERVER_IP:9898` — **not** to the Service IP.
The DNAT happened on the **client's** node, and the Pod IP traveled unchanged across nodes (no NAT between Pods).

## Step 6 · conntrack: why load balancing is per connection

```bash
kubectl scale deployment server --replicas=3 && kubectl rollout status deployment/server
# New connection each time → different Pods
kubectl exec client -- sh -c 'for i in 1 2 3 4 5 6; do curl -s server/ | grep hostname; done'
# One keep-alive connection → always the same Pod
kubectl exec client -- sh -c 'curl -s server/ server/ server/ server/ | grep hostname'
```

✅ **Check:** separate `curl` calls hit several Pods; multiple requests on **one** connection hit the same Pod.

## Step 7 · DNS packets

```bash
kubectl debug -it client --image=nicolaka/netshoot --target=client -- \
  sh -c 'tcpdump -ni eth0 udp port 53 -c 12 & sleep 1; nslookup github.com >/dev/null; sleep 2'
```

Count the queries: `github.com.lab-19.svc.cluster.local`, `github.com.svc.cluster.local`, … before the
real name. That is `ndots:5` in action (module 08).

## 💥 Step 8 · Break it on purpose: delete a route

```bash
kubectl scale deployment server --replicas=1 && kubectl rollout status deployment/server
SERVER_NODE=$(kubectl get pod -l app=server -o jsonpath='{.items[0].spec.nodeName}')
SERVER_IP=$(kubectl get pod -l app=server -o jsonpath='{.items[0].status.podIP}')
OTHER_CIDR=$(kubectl get node $SERVER_NODE -o jsonpath='{.spec.podCIDR}')
docker exec kc-worker ip route show $OTHER_CIDR
docker exec kc-worker ip route del $OTHER_CIDR
kubectl exec client -- curl -s --max-time 3 server/version || echo "FAILED"
```

> This only breaks the path if `server` runs on `kc-worker2` (see Setup).

**Diagnosis:** the Service, endpoints and DNS are fine. Prove where the packet dies, then repair it.

<details><summary>Show solution</summary>

```bash
kubectl get endpointslices -l kubernetes.io/service-name=server   # endpoints OK
kubectl exec client -- nslookup server                               # DNS OK
docker exec kc-worker ip route get $SERVER_IP                       # → default route, wrong path!
```
Without the per-node route, packets for the other node's Pod CIDR follow the default route and are
dropped. This is exactly what a broken CNI looks like. Repair: kindnet re-adds routes on its
reconcile loop; force it by restarting the kindnet Pod on that node:
```bash
kubectl -n kube-system delete pod -l app=kindnet --field-selector spec.nodeName=kc-worker
sleep 10; docker exec kc-worker ip route show $OTHER_CIDR
kubectl exec client -- curl -s server/version
```
</details>

## 🧹 Cleanup

```bash
kubectl config set-context --current --namespace=default
kubectl delete namespace lab-19        # also removes the node-debugger Pods created in this namespace
```

➡️ Next: [📝 EXERCISES.md](EXERCISES.md)
