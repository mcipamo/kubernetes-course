# 📝 Exercises 17 · Security I: AuthN, AuthZ and RBAC

## 🟢 Exercise 1 · 401 or 403?
For each case say 401 or 403: (a) expired client certificate; (b) valid user without permissions; (c)
denied by an admission policy; (d) token signed by an unknown issuer.

<details><summary>Show solution</summary>

(a) 401 · (b) 403 · (c) 403 (with an admission message) · (d) 401
</details>

## 🟢 Exercise 2 · Read-only on-call
Give the group `oncall` read-only access to everything in **all** namespaces using built-in roles.

<details><summary>Show solution</summary>

```bash
kubectl create clusterrolebinding oncall-view --clusterrole=view --group=oncall
```
Note: `view` intentionally excludes Secrets.
</details>

## 🟡 Exercise 3 · CI deployer
Create a ServiceAccount `deployer` in namespace `ci` that can only `get`, `patch` and `update`
Deployments in namespace `shop`. Mint a 15-minute token for it.

<details><summary>Show solution</summary>

```bash
kubectl create namespace ci
kubectl create serviceaccount deployer -n ci
kubectl create role deployer --verb=get,patch,update --resource=deployments.apps -n shop
kubectl create rolebinding deployer --role=deployer --serviceaccount=ci:deployer -n shop
kubectl create token deployer -n ci --duration=15m
kubectl auth can-i patch deployments -n shop --as=system:serviceaccount:ci:deployer   # yes
kubectl auth can-i delete deployments -n shop --as=system:serviceaccount:ci:deployer  # no
```
</details>

## 🟡 Exercise 4 · Aggregated ClusterRole
You installed a CRD `backups.example.com`. Make every user with the built-in `view` role able to read backups.

<details><summary>Show solution</summary>

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: backups-view
  labels:
    rbac.authorization.k8s.io/aggregate-to-view: "true"
rules:
  - apiGroups: ["example.com"]
    resources: ["backups"]
    verbs: ["get", "list", "watch"]
```
The RBAC controller merges these rules into `view` automatically.
</details>

## 🔴 Challenge · RBAC review
List five checks you would include in a quarterly RBAC review, with the command or tool for each.

<details><summary>Show solution</summary>

| Check | How |
|---|---|
| Who has `cluster-admin`? | `kubectl get clusterrolebindings -o json \| jq '.items[] \| select(.roleRef.name=="cluster-admin") \| .subjects'` |
| Who can read Secrets? | `kubectl-who-can get secrets -A` / `rbac-tool who-can get secrets` |
| Who can create Pods / exec? | `kubectl-who-can create pods`, `who-can create pods/exec` |
| Wildcard rules (`*`) | `kubectl get clusterroles,roles -A -o yaml \| grep -n '"\*"'` |
| Unused ServiceAccounts / stale bindings to deleted users | Compare bindings against the IdP and audit logs |
</details>

---

## ❓ Quiz

1. Does Kubernetes store User objects?
   a) Only for admins b) **No — users come from external authenticators** c) Only with OIDC d) Yes, in etcd

2. A ClusterRole bound with a **RoleBinding** grants permissions…
   a) **Only in the RoleBinding's namespace** b) Nowhere c) Cluster-wide d) Only on cluster-scoped resources

3. Which permission effectively exposes every Secret in a namespace?
   a) `watch events` b) `list services` c) **`create pods`** d) `get configmaps`

4. How do you prevent a Pod from receiving a ServiceAccount token?
   a) It is impossible b) Use `default` c) Delete the SA d) **`automountServiceAccountToken: false`**

5. RBAC rules can…
   a) Allow and deny b) **Only allow** c) Allow per IP d) Only deny

<details><summary>Show answers</summary>

1-b · 2-a · 3-c · 4-d · 5-b
</details>

➡️ Next module: [18 · Security II: Pod Security and NetworkPolicy](../18-pod-security-networkpolicy/)
