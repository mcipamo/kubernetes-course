# 📝 Exercises 24 · GitOps and CI/CD

## 🟢 Exercise 1 · Principles
Which OpenGitOps principle does each practice violate? (a) Engineers fix prod with `kubectl edit`.
(b) CI runs `kubectl apply` with an admin kubeconfig. (c) Manifests live in a shared drive without history.

<details><summary>Show solution</summary>

(a) Continuously reconciled (drift is accepted) · (b) Pulled automatically (it is push-based) ·
(c) Versioned and immutable
</details>

## 🟢 Exercise 2 · Ignore HPA replicas
An Application keeps showing `OutOfSync` because an HPA changes `spec.replicas`. Fix it two ways.

<details><summary>Show solution</summary>

1. Remove `replicas` from the Deployment in Git (best).
2. Tell Argo CD to ignore the field:
```yaml
spec:
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas
```
</details>

## 🟡 Exercise 3 · ApplicationSet per environment
Replace the two Applications from the lab with one ApplicationSet that generates an app for every
directory under `apps/shop/overlays/`.

<details><summary>Show solution</summary>

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: shop
  namespace: argocd
spec:
  goTemplate: true
  generators:
    - git:
        repoURL: https://github.com/<your-user>/kc-gitops.git
        revision: main
        directories:
          - path: apps/shop/overlays/*
  template:
    metadata:
      name: 'shop-{{.path.basename}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/<your-user>/kc-gitops.git
        targetRevision: main
        path: '{{.path.path}}'
      destination:
        server: https://kubernetes.default.svc
      syncPolicy:
        automated: { prune: true, selfHeal: true }
        syncOptions: [CreateNamespace=true]
```
Adding `overlays/staging` now creates `shop-staging` automatically.
</details>

## 🟡 Exercise 4 · CI pipeline
Write the steps (as a GitHub Actions job outline) that build, scan, sign and promote an image to the
config repo.

<details><summary>Show solution</summary>

```yaml
jobs:
  release:
    runs-on: ubuntu-latest
    permissions: { contents: read, packages: write, id-token: write }   # id-token for keyless signing
    steps:
      - uses: actions/checkout@v4
      - run: make test
      - uses: docker/build-push-action@v6
        id: build
        with: { push: true, tags: "ghcr.io/acme/shop:${{ github.sha }}" }
      - uses: aquasecurity/trivy-action@0.28.0
        with: { image-ref: "ghcr.io/acme/shop:${{ github.sha }}", exit-code: "1", severity: "CRITICAL,HIGH" }
      - uses: sigstore/cosign-installer@v3
      - run: cosign sign --yes ghcr.io/acme/shop@${{ steps.build.outputs.digest }}
      - name: Open promotion PR in the config repo
        run: |
          # clone config repo, `kustomize edit set image ghcr.io/acme/shop@<digest>` in overlays/staging, open PR
```
⚠️ Pin action versions you have verified; the ones above are examples.
</details>

## 🔴 Challenge · Canary with automatic analysis
Design a canary for `shop` that sends 10% → 50% → 100% of traffic and aborts if the 5xx ratio is above
1% or p99 latency above 500 ms.

<details><summary>Show solution</summary>

With **Argo Rollouts** + Gateway API traffic routing (or Flagger):
```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: shop-slo
spec:
  metrics:
    - name: error-ratio
      interval: 1m
      failureLimit: 1
      successCondition: result[0] < 0.01
      provider:
        prometheus:
          address: http://kps-kube-prometheus-stack-prometheus.monitoring:9090
          query: |
            sum(rate(http_requests_total{app="shop",status=~"5.."}[2m]))
            / sum(rate(http_requests_total{app="shop"}[2m]))
    - name: p99-latency
      interval: 1m
      failureLimit: 1
      successCondition: result[0] < 0.5
      provider:
        prometheus:
          address: http://kps-kube-prometheus-stack-prometheus.monitoring:9090
          query: |
            histogram_quantile(0.99, sum by (le) (rate(http_request_duration_seconds_bucket{app="shop"}[2m])))
```
Rollout steps: `setWeight: 10` → analysis → `setWeight: 50` → analysis → full promotion. A failed
analysis sets the weight back to 0 and marks the rollout `Degraded` — no human needed at 3 a.m.
</details>

---

## ❓ Quiz

1. In pull-based GitOps, who applies changes to the cluster?
   a) The registry b) **An agent running in the cluster** c) The CI server d) The developer

2. `selfHeal: true` in Argo CD means…
   a) **Manual changes in the cluster are reverted to Git** b) Pods restart on failure c) Git is fixed automatically d) Nodes are repaired

3. An app is `Synced` but `Degraded`. This means…
   a) Argo CD is down b) Git and cluster differ c) **The cluster matches Git, but the workload is unhealthy** d) Pruning failed

4. The preferred GitOps rollback is…
   a) Restarting Argo CD b) **`git revert` of the bad commit** c) Deleting the Application d) `kubectl rollout undo`

5. Which tool keeps secrets in Git **encrypted**?
   a) ConfigMaps b) Kustomize patches c) Annotations d) **Sealed Secrets / SOPS**

<details><summary>Show answers</summary>

1-b · 2-a · 3-c · 4-b · 5-d
</details>

➡️ Next module: [25 · CRDs and Operators](../25-crds-operators/)
