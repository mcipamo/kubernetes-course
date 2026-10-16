# 📝 Exercises 06 · Namespaces, Labels and Selectors

## 🟢 Exercise 1 · Scope check
Classify as namespaced or cluster-scoped: Pod, Node, PersistentVolume, PersistentVolumeClaim, ClusterRole, Role, StorageClass, Service, Namespace.

<details><summary>Show solution</summary>

- **Namespaced:** Pod, PersistentVolumeClaim, Role, Service
- **Cluster-scoped:** Node, PersistentVolume, ClusterRole, StorageClass, Namespace

Verify any type with `kubectl api-resources | grep -i <kind>` (the `NAMESPACED` column).
</details>

## 🟢 Exercise 2 · Label surgery
On Pod `web`: add `tier=frontend`, change it to `tier=edge`, then remove it.

<details><summary>Show solution</summary>

```bash
kubectl label pod web tier=frontend
kubectl label pod web tier=edge --overwrite
kubectl label pod web tier-
```
</details>

## 🟡 Exercise 3 · Selector translation
Translate to a `kubectl -l` expression: "Pods of team payments, in prod or staging, that are not canaries".

<details><summary>Show solution</summary>

```bash
kubectl get pods -A -l 'team=payments,env in (prod,staging),!canary'
```
And as a `matchExpressions` block:
```yaml
selector:
  matchLabels:
    team: payments
  matchExpressions:
    - { key: env, operator: In, values: [prod, staging] }
    - { key: canary, operator: DoesNotExist }
```
</details>

## 🟡 Exercise 4 · Custom columns report
Print a table of all Pods in all namespaces with columns NAMESPACE, NAME, TEAM (label), NODE.

<details><summary>Show solution</summary>

```bash
kubectl get pods -A -o custom-columns='NAMESPACE:.metadata.namespace,NAME:.metadata.name,TEAM:.metadata.labels.team,NODE:.spec.nodeName'
```
</details>

## 🔴 Challenge · Quarantine a Pod
A Pod of a Deployment is misbehaving and you want to keep it for forensics without it receiving traffic
and without the Deployment deleting it. How do you do it with only labels?

<details><summary>Show solution</summary>

Change the label that **both** the Service and the ReplicaSet select on:
```bash
kubectl label pod web-7d4b9c-abcde app=web-quarantine --overwrite
```
- The **Service** stops sending traffic (selector no longer matches).
- The **ReplicaSet** no longer counts it as its own, so it creates a replacement to keep the replica count.
- The quarantined Pod stays alive for `kubectl exec`, `logs` and `debug`. Delete it when you are done.
</details>

---

## ❓ Quiz

1. By default, can a Pod in namespace `a` reach a Pod in namespace `b`?

   - a) Only through Ingress
   - b) No
   - c) Yes
   - d) Only with RBAC

2. Which of these can you use in a selector?

   - a) Names
   - b) Both
   - c) Labels
   - d) Annotations

3. The full DNS name of Service `api` in namespace `shop` is:

   - a) `api.shop.svc.cluster.local`
   - b) `api.cluster.local`
   - c) `api.svc.shop`
   - d) `shop.api.svc`

4. Which field of a Deployment cannot be changed after creation?

   - a) `template`
   - b) `strategy`
   - c) `selector`
   - d) `replicas`

5. `kubectl get pods -l '!canary'` returns Pods that…

   - a) Have `canary=false`
   - b) Fail
   - c) Have any canary value
   - d) Do not have the `canary` key at all

<details><summary>Show answers</summary>

1. **c)** Yes
2. **c)** Labels
3. **a)** `api.shop.svc.cluster.local`
4. **c)** `selector`
5. **d)** Do not have the `canary` key at all
</details>

➡️ Next module: [07 · ReplicaSets and Deployments](../07-deployments/)
