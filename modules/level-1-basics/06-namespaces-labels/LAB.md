# 🧪 Lab 06 · Organize a Multi-Team Cluster

| Goal | Duration | Environment |
|---|---|---|
| Model two teams and three environments with namespaces and labels | 60 min | `kc` kind cluster |

## Step 1 · Create the namespaces

Create `namespaces.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: lab-06-payments
  labels:
    team: payments
---
apiVersion: v1
kind: Namespace
metadata:
  name: lab-06-catalog
  labels:
    team: catalog
```

```bash
kubectl apply -f namespaces.yaml
kubectl get ns -l team --show-labels
```

✅ **Check:** two namespaces with their `team` label.

## Step 2 · Deploy labeled Pods

```bash
for env in dev qa prod; do
  kubectl -n lab-06-payments run pay-$env --image=nginx:1.27 \
    --labels="app.kubernetes.io/name=payments,env=$env,tier=backend"
  kubectl -n lab-06-catalog run cat-$env --image=nginx:1.27 \
    --labels="app.kubernetes.io/name=catalog,env=$env,tier=frontend"
done
kubectl get pods -A -l app.kubernetes.io/name -L env,tier
```

✅ **Check:** six Pods; the `ENV` and `TIER` columns are filled in.

## Step 3 · Ask questions with selectors

```bash
kubectl get pods -A -l env=prod                              # every prod Pod
kubectl get pods -A -l 'env in (dev,qa)'                     # non-prod
kubectl get pods -A -l 'env!=prod,tier=frontend'             # non-prod frontends
kubectl get pods -A -l '!canary'                             # no canary label
kubectl get pods -A --field-selector spec.nodeName=kc-worker -l app.kubernetes.io/name
```

✅ **Check:** the first query returns exactly 2 Pods (`pay-prod`, `cat-prod`).

## Step 4 · Labels drive Services

```bash
kubectl -n lab-06-payments expose pod pay-prod --name=payments --port=80 --selector='env=prod'
kubectl -n lab-06-payments get endpointslices -l kubernetes.io/service-name=payments
kubectl -n lab-06-payments label pod pay-qa env=prod --overwrite
kubectl -n lab-06-payments get endpointslices -l kubernetes.io/service-name=payments -o wide
```

✅ **Check:** after relabeling `pay-qa`, the Service has **two** endpoints. You changed traffic routing
with a label — no restart, no redeploy.

## Step 5 · Annotations

```bash
kubectl -n lab-06-payments annotate pod pay-prod \
  owner=ana@example.com runbook=https://wiki.example.com/payments
kubectl -n lab-06-payments get pod pay-prod -o jsonpath='{.metadata.annotations}'; echo
kubectl -n lab-06-payments get pods -l owner=ana@example.com   # → No resources found
```

That last command finds nothing because you **cannot select by annotations**.

## Step 6 · Cross-namespace DNS

```bash
kubectl -n lab-06-catalog run dns --image=busybox:1.36 --rm -it --restart=Never -- \
  nslookup payments.lab-06-payments.svc.cluster.local
```

✅ **Check:** the name resolves to the Service IP. Namespaces do not block networking by default.

## 💥 Step 7 · Break it on purpose

A teammate "cleans up" labels:

```bash
kubectl -n lab-06-payments label pod pay-prod pay-qa env-
kubectl -n lab-06-payments run probe --image=curlimages/curl:8.10.1 --rm -it --restart=Never -- \
  curl -s --max-time 3 payments || echo "REQUEST FAILED"
```

**Diagnosis:** the Service still exists and has an IP. Why does the request fail? Prove it with one command.

<details><summary>Show solution</summary>

```bash
kubectl -n lab-06-payments get endpointslices -l kubernetes.io/service-name=payments
# ENDPOINTS: <unset>
kubectl -n lab-06-payments describe svc payments | grep -E 'Selector|Endpoints'
```
The Service selector `env=prod` matches **no Pods**, so it has no endpoints and connections fail.
This is the #1 cause of "my Service doesn't work". Fix:
```bash
kubectl -n lab-06-payments label pod pay-prod env=prod
```
</details>

## 🧹 Cleanup

```bash
kubectl delete -f namespaces.yaml     # deleting a namespace deletes everything inside it
```

➡️ Next: [📝 EXERCISES.md](EXERCISES.md)
