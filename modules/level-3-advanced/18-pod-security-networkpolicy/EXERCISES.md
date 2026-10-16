# 📝 Exercises 18 · Security II: Pod Security and NetworkPolicy

## 🟢 Exercise 1 · Fix it for `restricted`
Make this container compliant with the `restricted` standard:
`image: my-api:1.0` with no securityContext.

<details><summary>Show solution</summary>

```yaml
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 10001
    seccompProfile: { type: RuntimeDefault }
  containers:
    - name: api
      image: my-api:1.0
      securityContext:
        allowPrivilegeEscalation: false
        capabilities: { drop: ["ALL"] }
```
`readOnlyRootFilesystem` is not required by `restricted`, but strongly recommended.
</details>

## 🟢 Exercise 2 · Port 80 as non-root
A non-root nginx must listen on port 80. What is the minimum change?

<details><summary>Show solution</summary>

Either listen on 8080 and map `Service port: 80 → targetPort: 8080` (best), or add back one capability:
```yaml
capabilities:
  drop: ["ALL"]
  add: ["NET_BIND_SERVICE"]
```
(`restricted` allows adding only `NET_BIND_SERVICE`.)
</details>

## 🟡 Exercise 3 · Allow from another namespace
Allow Pods with `app=ingress` in namespace `edge` to reach Pods `app=web` in namespace `shop` on TCP 8080.

<details><summary>Show solution</summary>

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: web-from-edge
  namespace: shop
spec:
  podSelector:
    matchLabels: { app: web }
  policyTypes: [Ingress]
  ingress:
    - from:
        - namespaceSelector:
            matchLabels: { kubernetes.io/metadata.name: edge }
          podSelector:                       # same item → AND
            matchLabels: { app: ingress }
      ports:
        - { protocol: TCP, port: 8080 }
```
If `edge` also has default-deny egress, it needs a matching egress rule.
</details>

## 🟡 Exercise 4 · Block the metadata service
In the cloud, Pods can reach the instance metadata endpoint `169.254.169.254` and steal node
credentials. Write an egress policy that allows all egress except that IP.

<details><summary>Show solution</summary>

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: block-metadata
spec:
  podSelector: {}
  policyTypes: [Egress]
  egress:
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
            except: ["169.254.169.254/32"]
```
</details>

## 🔴 Challenge · Rollout plan
Plan the rollout of `restricted` PSS and default-deny NetworkPolicies across 40 namespaces owned by 10
teams, without causing outages.

<details><summary>Show solution</summary>

1. **Inventory**: `kubectl label --dry-run=server` per namespace to list PSS violations; map traffic
   flows (CNI flow logs, mesh telemetry, Hubble).
2. **Warn + audit everywhere** (`warn=restricted`, `audit=restricted`); publish a dashboard of violations per team.
3. **Fix by team**, with templates (Helm/Kustomize base) that are already compliant.
4. **NetworkPolicies first in `audit`/log mode** if your CNI supports it; otherwise apply allow rules
   *before* default-deny.
5. **Enforce gradually**: non-prod → low-risk prod → critical prod; one team at a time; with a fast
   rollback (remove the label / policy via GitOps).
6. **Prevent regressions** with admission policies and CI checks (kubeconform + policy tests).
</details>

---

## ❓ Quiz

1. Which PSS level requires `runAsNonRoot` and dropping ALL capabilities?
   a) privileged b) baseline c) none d) **restricted**

2. By default (no policies), Pods in different namespaces…
   a) Cannot communicate b) Need RBAC c) Need a Service d) **Can communicate freely**

3. In a NetworkPolicy `from` list, a `namespaceSelector` and `podSelector` in the **same item** mean…
   a) OR b) Nothing c) **AND** d) NOT

4. After applying default-deny egress, what usually breaks first?
   a) **DNS resolution** b) Volumes c) Probes d) Logs

5. `hostUsers: false` enables…
   a) Read-only root b) Privileged mode c) Host networking d) **User namespaces**

<details><summary>Show answers</summary>

1-d · 2-d · 3-c · 4-a · 5-d
</details>

➡️ Next module: [19 · Networking Deep Dive](../19-networking-deep-dive/)
