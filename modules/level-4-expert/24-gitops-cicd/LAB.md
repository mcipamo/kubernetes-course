# 🧪 Lab 24 · Git Is the Source of Truth

| Goal | Duration | Environment |
|---|---|---|
| Install Argo CD, deploy from Git, watch it heal drift, and promote a change with a commit | 120 min | `kc` kind cluster · a GitHub account (for Part B) |

## Part A — Argo CD with a public repository

### Step 1 · Install Argo CD

```bash
kubectl create namespace argocd
kubectl apply -n argocd --server-side -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl -n argocd rollout status deployment/argocd-server --timeout=300s
kubectl -n argocd get pods
```

✅ **Check:** `argocd-server`, `argocd-repo-server`, `argocd-application-controller`, `argocd-redis` are `Running`.

### Step 2 · Open the UI

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d; echo
kubectl -n argocd port-forward svc/argocd-server 8443:443 >/dev/null 2>&1 &
```

Open https://localhost:8443 (accept the self-signed certificate), user `admin`, password from above.

### Step 3 · Your first Application

```bash
mkdir -p ~/kc/lab24 && cd ~/kc/lab24
```

Create `guestbook-app.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    targetRevision: HEAD
    path: kustomize-guestbook
  destination:
    server: https://kubernetes.default.svc
    namespace: lab-24
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

```bash
kubectl apply -f guestbook-app.yaml
sleep 30
kubectl -n argocd get application guestbook
kubectl -n lab-24 get deploy,svc
```

✅ **Check:** `SYNC STATUS Synced`, `HEALTH STATUS Healthy`, and the guestbook Deployment exists in `lab-24`.

### Step 4 · Drift: a "quick fix" by hand

```bash
DEPLOY=$(kubectl -n lab-24 get deploy -o jsonpath='{.items[0].metadata.name}')
kubectl -n lab-24 scale deployment $DEPLOY --replicas=5
kubectl -n lab-24 get deploy $DEPLOY -w     # watch it go back to the Git value; Ctrl+C
```

✅ **Check:** within seconds Argo CD reverts the replicas to the value in Git. In the UI, the app history
shows a self-heal sync.

### Step 5 · Drift: deleting a resource

```bash
kubectl -n lab-24 delete svc --all
sleep 15; kubectl -n lab-24 get svc
```

✅ **Check:** the Service is recreated. With GitOps, "fixing" production by hand is not possible —
which is exactly the point.

## Part B — Your own config repository

### Step 6 · Create a config repo

1. On GitHub, create a **public** repository `kc-gitops`.
2. Copy the Kustomize app from module 23 into it:

```bash
cd ~/kc && git clone https://github.com/<your-user>/kc-gitops.git && cd kc-gitops
mkdir -p apps && cp -r ~/kc/lab23/shop apps/shop
git add . && git commit -m "Add shop app (base + dev/prod overlays)" && git push
```

### Step 7 · An Application per environment

Create `~/kc/lab24/shop-apps.yaml` (replace `<your-user>`):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: shop-dev
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/<your-user>/kc-gitops.git
    targetRevision: main
    path: apps/shop/overlays/dev
  destination:
    server: https://kubernetes.default.svc
  syncPolicy:
    automated: { prune: true, selfHeal: true }
    syncOptions: [CreateNamespace=true]
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: shop-prod
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/<your-user>/kc-gitops.git
    targetRevision: main
    path: apps/shop/overlays/prod
  destination:
    server: https://kubernetes.default.svc
  syncPolicy:
    automated: { prune: true, selfHeal: true }
    syncOptions: [CreateNamespace=true]
```

```bash
kubectl apply -f ~/kc/lab24/shop-apps.yaml
kubectl -n argocd get applications
```

✅ **Check:** `shop-dev` and `shop-prod` are `Synced` and `Healthy`.

### Step 8 · Promote with a commit

```bash
cd ~/kc/kc-gitops
sed -i.bak 's/count: 3/count: 4/' apps/shop/overlays/prod/kustomization.yaml && rm apps/shop/overlays/prod/kustomization.yaml.bak
git commit -am "prod: scale shop to 4 replicas" && git push
# Argo CD polls Git every ~3 minutes; to speed it up, click "Refresh" in the UI
sleep 180; kubectl -n lab-23-prod get deploy shop
```

✅ **Check:** prod runs 4 replicas, and the change is a commit with an author, a message and a diff.

## 💥 Step 9 · Break it on purpose: a bad commit

```bash
sed -i.bak 's/newTag: 6.7.0/newTag: 0.0.0-broken/' apps/shop/overlays/prod/kustomization.yaml && rm apps/shop/overlays/prod/kustomization.yaml.bak
git commit -am "prod: bump podinfo" && git push
sleep 180; kubectl -n argocd get application shop-prod; kubectl -n lab-23-prod get pods
```

**Diagnosis:** what do Argo CD's sync and health statuses say? What is the GitOps way to recover?

<details><summary>Show solution</summary>

`Synced` (the cluster matches Git) but **`Degraded`/`Progressing`** health: new Pods are in
`ImagePullBackOff`, old Pods keep serving thanks to the rolling update. Sync ≠ healthy.

Recover the GitOps way — revert the commit:
```bash
git revert --no-edit HEAD && git push
```
Argo CD syncs the revert and the app becomes `Healthy`. (`argocd app rollback` also works, but it
disables auto-sync and leaves Git and cluster out of step — prefer `git revert`.)
Prevent it: CI must verify the tag exists (and is signed) before opening the promotion PR.
</details>

## 🧹 Cleanup

```bash
kubectl -n argocd delete application guestbook shop-dev shop-prod
kubectl delete namespace lab-24 lab-23-dev lab-23-prod --ignore-not-found
# keep Argo CD for the capstone, or remove it:
# kubectl delete namespace argocd
```

➡️ Next: [📝 EXERCISES.md](EXERCISES.md)
