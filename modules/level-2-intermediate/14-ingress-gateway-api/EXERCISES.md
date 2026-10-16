# 📝 Exercises 14 · Ingress and Gateway API

## 🟢 Exercise 1 · Who owns what?
Assign each object to a persona (infrastructure provider, cluster operator, app developer):
GatewayClass, Gateway, HTTPRoute, ReferenceGrant.

<details><summary>Show solution</summary>

GatewayClass → infrastructure provider · Gateway → cluster operator · HTTPRoute → app developer ·
ReferenceGrant → owner of the namespace being referenced (usually the app or platform team).
</details>

## 🟢 Exercise 2 · Redirect HTTP to HTTPS
Write an HTTPRoute attached to the `http` listener that redirects everything to HTTPS with a 301.

<details><summary>Show solution</summary>

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: https-redirect
spec:
  parentRefs:
    - name: public
      namespace: infra
      sectionName: http          # attach only to the HTTP listener
  rules:
    - filters:
        - type: RequestRedirect
          requestRedirect:
            scheme: https
            statusCode: 301
```
</details>

## 🟡 Exercise 3 · Translate an Ingress
Convert this to Gateway API: host `blog.example.com`, path `/` → Service `blog:80`, TLS with Secret `blog-tls`.

<details><summary>Show solution</summary>

TLS moves to the **Gateway** listener; routing stays in the **HTTPRoute**:
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: blog
spec:
  gatewayClassName: eg
  listeners:
    - name: https
      protocol: HTTPS
      port: 443
      hostname: blog.example.com
      tls:
        certificateRefs:
          - name: blog-tls
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: blog
spec:
  parentRefs:
    - name: blog
  hostnames: ["blog.example.com"]
  rules:
    - backendRefs:
        - { name: blog, port: 80 }
```
The `ingress2gateway` tool automates this for many controllers.
</details>

## 🟡 Exercise 4 · Mirror traffic
Send a copy of all `/api` traffic to `api-shadow` without affecting responses.

<details><summary>Show solution</summary>

```yaml
rules:
  - matches:
      - path: { type: PathPrefix, value: /api }
    filters:
      - type: RequestMirror
        requestMirror:
          backendRef: { name: api-shadow, port: 80 }
    backendRefs:
      - { name: api-v1, port: 80 }
```
Responses from the mirror are discarded. Great for testing a rewrite with production traffic.
</details>

## 🔴 Challenge · Cross-namespace backend
An HTTPRoute in namespace `frontend` must send `/auth` to Service `auth` in namespace `identity`.
Write everything required.

<details><summary>Show solution</summary>

```yaml
# In namespace frontend
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: app
  namespace: frontend
spec:
  parentRefs: [{ name: public, namespace: infra }]
  rules:
    - matches: [{ path: { type: PathPrefix, value: /auth } }]
      backendRefs:
        - { name: auth, namespace: identity, port: 80 }
---
# In namespace identity: the target namespace must explicitly allow it
apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
  name: allow-frontend-routes
  namespace: identity
spec:
  from:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      namespace: frontend
  to:
    - group: ""
      kind: Service
      name: auth
```
Without the ReferenceGrant, the route shows `ResolvedRefs=False, RefNotPermitted`.
</details>

---

## ❓ Quiz

1. Which object defines ports, protocols and TLS certificates in Gateway API?
   a) **Gateway** b) HTTPRoute c) GatewayClass d) ReferenceGrant

2. In Ingress, advanced features like canaries are usually configured with…
   a) Spec fields b) ConfigMaps only c) **Controller-specific annotations** d) They are impossible

3. An HTTPRoute shows `Accepted=False, NotAllowedByListeners`. The likely cause is…
   a) **The Gateway listener does not allow routes from that namespace** b) Bad TLS c) DNS d) The Service does not exist

4. A weighted split `weight: 90` / `weight: 10` sends to the second backend about…
   a) 90% b) 1% c) **10%** d) 50%

5. Which tool automates TLS certificates for Gateways and Ingresses?
   a) **cert-manager** b) CoreDNS c) kube-proxy d) Kustomize

<details><summary>Show answers</summary>

1-a · 2-c · 3-a · 4-c · 5-a
</details>

➡️ Next module: [15 · Scheduling](../15-scheduling/)
