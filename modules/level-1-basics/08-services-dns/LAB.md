# 🧪 Lab 08 · Wire Up Services and DNS

| Goal | Duration | Environment |
|---|---|---|
| Use every Service type, inspect the plumbing and fix a broken Service | 90 min | `kc` kind cluster |

## Setup

```bash
kubectl create namespace lab-08
kubectl config set-context --current --namespace=lab-08
kubectl create deployment api --image=ghcr.io/stefanprodan/podinfo:6.7.0 --replicas=3 --port=9898
kubectl rollout status deployment/api
```

## Step 1 · ClusterIP

```bash
kubectl expose deployment api --port=80 --target-port=9898
kubectl get svc api -o wide
kubectl get endpointslices -l kubernetes.io/service-name=api -o wide
```

✅ **Check:** the EndpointSlice lists **3** addresses, the same as `kubectl get pods -l app=api -o wide`.

## Step 2 · DNS from inside the cluster

```bash
kubectl run net --image=nicolaka/netshoot --restart=Never -- sleep infinity
kubectl wait --for=condition=Ready pod/net
kubectl exec net -- cat /etc/resolv.conf
kubectl exec net -- nslookup api
kubectl exec net -- dig +short api.lab-08.svc.cluster.local
kubectl exec net -- dig +short SRV _http._tcp.api.lab-08.svc.cluster.local || true
```

**Expected output:** `search lab-08.svc.cluster.local svc.cluster.local cluster.local` and `ndots:5`
in `resolv.conf`, and the Service ClusterIP from `nslookup`.

> The SRV query only returns a record if the port is named. `kubectl expose` did not name it — try
> again after Step 6.

## Step 3 · Load balancing

```bash
kubectl exec net -- sh -c 'for i in $(seq 1 12); do curl -s api/ | grep hostname; done' | sort | uniq -c
```

✅ **Check:** responses come from all 3 Pods (each connection is balanced independently).

## Step 4 · See the kernel rules kube-proxy wrote

```bash
SVC_IP=$(kubectl get svc api -o jsonpath='{.spec.clusterIP}')
docker exec kc-worker sh -c "iptables-save 2>/dev/null | grep -c $SVC_IP; nft list ruleset 2>/dev/null | grep -c $SVC_IP" 
kubectl -n kube-system get cm kube-proxy -o jsonpath='{.data.config\.conf}' | grep -i '^mode'
```

You will see rules for the Service IP on **every** node — the virtual IP exists only as rules.

## Step 5 · NodePort

```bash
kubectl expose deployment api --name=api-np --type=NodePort --port=80 --target-port=9898
NODE_PORT=$(kubectl get svc api-np -o jsonpath='{.spec.ports[0].nodePort}')
NODE_IP=$(kubectl get node kc-worker -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}')
docker exec kc-control-plane curl -s $NODE_IP:$NODE_PORT/version
```

✅ **Check:** you get the podinfo version JSON through a node IP and a port in the 30000–32767 range.

## Step 6 · Headless Service

Create `headless.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api-headless
spec:
  clusterIP: None              # headless: no virtual IP
  selector:
    app: api
  ports:
    - name: http
      port: 9898
      targetPort: 9898
```

```bash
kubectl apply -f headless.yaml
kubectl exec net -- dig +short api-headless.lab-08.svc.cluster.local
kubectl exec net -- dig +short SRV _http._tcp.api-headless.lab-08.svc.cluster.local
```

✅ **Check:** the A query returns **three Pod IPs**, not one Service IP.

## Step 7 · ExternalName

```bash
kubectl create service externalname docs --external-name=kubernetes.io
kubectl exec net -- dig +short docs.lab-08.svc.cluster.local | head -2
```

The first line is a CNAME to `kubernetes.io.` — a DNS alias, no proxying.

## Step 8 · Readiness removes endpoints

First give the Pods a readiness probe:

```bash
kubectl patch deployment api --type=json -p='[{"op":"add","path":"/spec/template/spec/containers/0/readinessProbe","value":{"httpGet":{"path":"/readyz","port":9898},"periodSeconds":2}}]'
kubectl rollout status deployment/api
```

Now tell one Pod to report "not ready" (podinfo has an endpoint for exactly this):

```bash
POD=$(kubectl get pod -l app=api -o jsonpath='{.items[0].metadata.name}')
kubectl exec $POD -- curl -s -X POST localhost:9898/readyz/disable
sleep 5
kubectl get endpointslices -l kubernetes.io/service-name=api \
  -o jsonpath='{range .items[*].endpoints[*]}{.addresses[0]}{" ready="}{.conditions.ready}{"\n"}{end}'
```

✅ **Check:** one endpoint shows `ready=false`. Repeat Step 3: only 2 hostnames answer.
Re-enable it with `kubectl exec $POD -- curl -s -X POST localhost:9898/readyz/enable`.

## 💥 Step 9 · Break it on purpose

Apply this "fixed" Service a teammate sent you:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api-v2
spec:
  selector:
    app: API
  ports:
    - port: 80
      targetPort: 8080
```

```bash
kubectl apply -f api-v2.yaml
kubectl exec net -- curl -s --max-time 3 api-v2 || echo FAILED
```

Use the checklist from the README to find **both** bugs.

<details><summary>Show solution</summary>

1. `kubectl get endpointslices -l kubernetes.io/service-name=api-v2` → no endpoints. The selector is
   `app: API`, but labels are **case-sensitive**: the Pods have `app: api`.
2. After fixing the selector, `curl` returns `Connection refused`: `targetPort: 8080`, but podinfo
   listens on **9898**.

```bash
kubectl patch svc api-v2 --type=merge -p '{"spec":{"selector":{"app":"api"},"ports":[{"port":80,"targetPort":9898}]}}'
kubectl exec net -- curl -s api-v2/version
```
</details>

## 🧹 Cleanup

```bash
kubectl config set-context --current --namespace=default
kubectl delete namespace lab-08
```

➡️ Next: [📝 EXERCISES.md](EXERCISES.md)
