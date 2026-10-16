# 🧪 Lab 14 · Route Traffic with Gateway API

| Goal | Duration | Environment |
|---|---|---|
| Install a Gateway API implementation, route by host/path/header and run a weighted canary | 90 min | `kc` kind cluster + `helm` |

We use **Envoy Gateway**, a CNCF implementation of Gateway API. Any conformant implementation works
the same way — only the `gatewayClassName` changes.

## Step 1 · Install Envoy Gateway (includes the Gateway API CRDs)

```bash
# Check the latest version at https://gateway.envoyproxy.io/news/releases/
EG_VERSION=v1.5.0
helm install eg oci://docker.io/envoyproxy/gateway-helm --version $EG_VERSION \
  -n envoy-gateway-system --create-namespace
kubectl -n envoy-gateway-system wait --for=condition=Available deployment/envoy-gateway --timeout=180s
kubectl api-resources --api-group=gateway.networking.k8s.io
```

✅ **Check:** the API group lists `gatewayclasses`, `gateways`, `httproutes`, `grpcroutes`, `referencegrants`…

## Step 2 · Platform team: GatewayClass and Gateway

```bash
kubectl create namespace lab-14-infra
mkdir -p ~/kc/lab14 && cd ~/kc/lab14
```

Create `gateway.yaml`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: eg
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: public
  namespace: lab-14-infra
spec:
  gatewayClassName: eg
  listeners:
    - name: http
      protocol: HTTP
      port: 80
      allowedRoutes:
        namespaces:
          from: Selector
          selector:
            matchLabels:
              gateway-access: "true"
```

```bash
kubectl apply -f gateway.yaml
kubectl -n lab-14-infra wait --for=condition=Accepted gateway/public --timeout=120s
kubectl -n lab-14-infra get gateway public -o jsonpath='{range .status.conditions[*]}{.type}={.status} ({.reason}){"\n"}{end}'
kubectl -n envoy-gateway-system get pods
```

✅ **Check:** `Accepted=True`, and an `envoy-lab-14-infra-public-…` Pod is `Running`.

> 💡 On kind you will see `Programmed=False (AddressNotAssigned)`. That is expected: the Gateway's
> Service is of type `LoadBalancer` and kind has no cloud load balancer to give it an address. The Envoy
> proxy works anyway — in the next step you reach it with a port-forward. On a cloud cluster (or with
> [cloud-provider-kind](https://github.com/kubernetes-sigs/cloud-provider-kind)) it becomes `Programmed=True`.

## Step 3 · Expose the Gateway on your laptop

```bash
ENVOY_SVC=$(kubectl -n envoy-gateway-system get svc \
  -l gateway.envoyproxy.io/owning-gateway-namespace=lab-14-infra,gateway.envoyproxy.io/owning-gateway-name=public \
  -o jsonpath='{.items[0].metadata.name}')
kubectl -n envoy-gateway-system port-forward svc/$ENVOY_SVC 8888:80 >/dev/null 2>&1 &
sleep 2; curl -s -o /dev/null -w "%{http_code}\n" localhost:8888
```

**Expected output:** `404` — the Gateway is listening, but no route matches yet.

## Step 4 · App team: two versions of an API and a website

```bash
kubectl create namespace lab-14-shop
kubectl label namespace lab-14-shop gateway-access=true
kubectl config set-context --current --namespace=lab-14-shop
for v in v1 v2; do
  kubectl create deployment api-$v --image=ghcr.io/stefanprodan/podinfo:6.7.0 --port=9898
  kubectl set env deployment/api-$v PODINFO_UI_MESSAGE="api $v"
  kubectl expose deployment api-$v --port=80 --target-port=9898
done
kubectl create deployment web --image=nginx:1.27 --port=80
kubectl expose deployment web --port=80
kubectl rollout status deployment/api-v1 && kubectl rollout status deployment/api-v2
```

## Step 5 · Host and path routing

Create `route.yaml`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: shop
  namespace: lab-14-shop
spec:
  parentRefs:
    - name: public
      namespace: lab-14-infra
  hostnames: ["shop.kc.local"]
  rules:
    - matches:
        - path: { type: PathPrefix, value: /api }
      filters:
        - type: URLRewrite
          urlRewrite:
            path: { type: ReplacePrefixMatch, replacePrefixMatch: / }
      backendRefs:
        - { name: api-v1, port: 80 }
    - backendRefs:
        - { name: web, port: 80 }
```

```bash
kubectl apply -f route.yaml
kubectl get httproute shop -o jsonpath='{range .status.parents[*].conditions[*]}{.type}={.status} {end}'; echo
curl -s -H 'Host: shop.kc.local' localhost:8888/api | grep message
curl -s -H 'Host: shop.kc.local' localhost:8888/ | grep -o '<title>.*</title>'
curl -s -o /dev/null -w "%{http_code}\n" -H 'Host: other.kc.local' localhost:8888/
```

✅ **Check:** `Accepted=True ResolvedRefs=True`; `/api` → `"message": "api v1"`; `/` → nginx; other host → `404`.

## Step 6 · Header-based routing and a 80/20 canary

Edit the `/api` rule in `route.yaml` and replace it with these two rules (keep the `web` rule last):

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: shop
  namespace: lab-14-shop
spec:
  parentRefs:
    - name: public
      namespace: lab-14-infra
  hostnames: ["shop.kc.local"]
  rules:
    - matches:
        - path: { type: PathPrefix, value: /api }
          headers:
            - { name: x-beta-user, value: "true" }
      filters:
        - type: URLRewrite
          urlRewrite: { path: { type: ReplacePrefixMatch, replacePrefixMatch: / } }
      backendRefs:
        - { name: api-v2, port: 80 }
    - matches:
        - path: { type: PathPrefix, value: /api }
      filters:
        - type: URLRewrite
          urlRewrite: { path: { type: ReplacePrefixMatch, replacePrefixMatch: / } }
      backendRefs:
        - { name: api-v1, port: 80, weight: 80 }
        - { name: api-v2, port: 80, weight: 20 }
    - backendRefs:
        - { name: web, port: 80 }
```

```bash
kubectl apply -f route.yaml
curl -s -H 'Host: shop.kc.local' -H 'x-beta-user: true' localhost:8888/api | grep message
for i in $(seq 1 50); do curl -s -H 'Host: shop.kc.local' localhost:8888/api | grep -o '"api v[12]"'; done | sort | uniq -c
```

✅ **Check:** beta users always get `api v2`; everyone else gets roughly **40 × v1 and 10 × v2**.

## 💥 Step 7 · Break it on purpose

A new team creates a route from a namespace that was not onboarded:

```bash
kubectl create namespace lab-14-rogue
kubectl -n lab-14-rogue create deployment rogue --image=nginx:1.27 --port=80
kubectl -n lab-14-rogue expose deployment rogue --port=80
cat <<'EOF' | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: rogue
  namespace: lab-14-rogue
spec:
  parentRefs:
    - name: public
      namespace: lab-14-infra
  hostnames: ["rogue.kc.local"]
  rules:
    - backendRefs:
        - { name: rogue, port: 8080 }
EOF
curl -s -o /dev/null -w "%{http_code}\n" -H 'Host: rogue.kc.local' localhost:8888/
```

**Diagnosis:** the route returns 404. Use only the route's **status** to find **two** problems.

<details><summary>Show solution</summary>

```bash
kubectl -n lab-14-rogue get httproute rogue -o yaml | yq '.status.parents[].conditions'
```
Both problems show up at the same time:
1. `Accepted: False`, reason **`NotAllowedByListeners`** — the namespace lacks the `gateway-access=true`
   label required by the listener. This is the Gateway API safety model working as designed.
2. `ResolvedRefs: False`, reason **`PortNotFound`** — the Service listens on port **80**, not 8080.

Fix both:
```bash
kubectl label ns lab-14-rogue gateway-access=true
kubectl -n lab-14-rogue patch httproute rogue --type=json \
  -p='[{"op":"replace","path":"/spec/rules/0/backendRefs/0/port","value":80}]'
curl -s -o /dev/null -w "%{http_code}\n" -H 'Host: rogue.kc.local' localhost:8888/    # 200
```
> The route status may not refresh right after you only label the namespace; it is re-evaluated when the
> route itself changes (the port patch).
</details>

## 🧹 Cleanup

```bash
kill %1 2>/dev/null
kubectl config set-context --current --namespace=default
kubectl delete namespace lab-14-shop lab-14-rogue lab-14-infra
# keep Envoy Gateway installed if you continue to module 24; otherwise:
# helm uninstall eg -n envoy-gateway-system
```

➡️ Next: [📝 EXERCISES.md](EXERCISES.md)
