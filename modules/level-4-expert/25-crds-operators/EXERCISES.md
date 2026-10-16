# 📝 Exercises 25 · CRDs and Operators

## 🟢 Exercise 1 · CRD naming
For a CRD with group `db.acme.io`, kind `PostgresCluster`, what must `metadata.name` be? How do you list the objects?

<details><summary>Show solution</summary>

`postgresclusters.db.acme.io` (`<plural>.<group>`). List with `kubectl get postgresclusters` (or a short name if defined).
</details>

## 🟢 Exercise 2 · More CEL
Add CEL rules: (a) `message` must not contain the word "password"; (b) `replicas` cannot be decreased
by more than 2 in a single update.

<details><summary>Show solution</summary>

```yaml
x-kubernetes-validations:
  - rule: "!self.message.contains('password')"
    message: "message must not contain secrets"
  - rule: "!has(oldSelf.replicas) || self.replicas >= oldSelf.replicas - 2"
    message: "scale down by at most 2 replicas at a time"
```
Rules that use `oldSelf` are **transition rules**: they only run on updates.
</details>

## 🟡 Exercise 3 · RBAC for the controller
The bash controller ran with your admin credentials. Write the minimal ClusterRole a real controller
for `Website` would need.

<details><summary>Show solution</summary>

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: website-controller
rules:
  - apiGroups: ["kc.dev"]
    resources: ["websites"]
    verbs: ["get", "list", "watch", "update", "patch"]
  - apiGroups: ["kc.dev"]
    resources: ["websites/status"]
    verbs: ["get", "update", "patch"]
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  - apiGroups: [""]
    resources: ["services"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  - apiGroups: ["coordination.k8s.io"]
    resources: ["leases"]
    verbs: ["get", "create", "update"]          # leader election
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "patch"]
```
</details>

## 🟡 Exercise 4 · Improve the controller
List four weaknesses of the bash controller compared with a controller-runtime operator.

<details><summary>Show solution</summary>

1. **Polling** every 5 s instead of **watching** (wasteful, slow to react, no informer cache).
2. No **work queue** with rate-limited retries and exponential back-off.
3. No **leader election** — two copies would fight.
4. No **conditions** / `observedGeneration`, no events, no metrics.
5. No handling of **finalizers** for external cleanup, no conversion webhooks, no tests.
</details>

## 🔴 Challenge · Should we build an operator?
A team wants to write an operator for their stateless web service "because operators are best practice".
How do you evaluate the request?

<details><summary>Show solution</summary>

Ask what **operational knowledge** the operator would automate that existing tools do not:
- Stateless web service → Deployment + HPA + PDB + Helm/Kustomize + GitOps already cover install,
  upgrade and scaling. An operator adds code to maintain, an extra failure domain and RBAC surface.
- Operators shine when there is **complex, stateful lifecycle**: failover, backups, schema migrations,
  rebalancing, version-specific upgrade steps (databases, queues, ML training jobs), or when you want a
  **platform API** (e.g., `Website`, `Database`) that hides many objects behind one abstraction.

Decision record: if the value is mostly "a nicer API", consider a Helm chart or a Crossplane/KRO-style
composition first. Build an operator only with a clear owner, tests and an on-call plan.
</details>

---

## ❓ Quiz

1. What does a CRD add to the cluster?
   a) **A new resource type served by the API server** b) A new namespace c) A new node d) A new controller

2. Which mechanism deletes a Deployment automatically when its parent custom resource is deleted?
   a) **Owner references (garbage collection)** b) Labels c) Finalizers d) Admission webhooks

3. CEL rules in a CRD schema go in…
   a) a ConfigMap b) **`x-kubernetes-validations`** c) annotations d) the controller

4. An object with a `deletionTimestamp` that never disappears most likely has…
   a) **A pending finalizer** b) A wrong label c) A bad schema d) No owner

5. Why use the `status` subresource?
   a) To encrypt status b) To make status public c) **To separate spec (user) from status (controller) updates and permissions** d) It is required for printer columns

<details><summary>Show answers</summary>

1-a · 2-a · 3-b · 4-a · 5-c
</details>

➡️ Next module: [26 · Admission Policies](../26-admission-policies/)
