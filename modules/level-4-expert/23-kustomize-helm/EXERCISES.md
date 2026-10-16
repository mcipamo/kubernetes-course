# 📝 Exercises 23 · Kustomize and Helm

## 🟢 Exercise 1 · Which tool?
(a) Deploy your own microservice to dev/staging/prod with small differences. (b) Install
kube-prometheus-stack. (c) Patch a third-party chart's output to add a label it does not support.

<details><summary>Show solution</summary>

(a) Kustomize overlays · (b) Helm · (c) Helm + Kustomize post-rendering (`helmCharts` in Kustomize, or `helm --post-renderer`)
</details>

## 🟢 Exercise 2 · Secret generator
Add a `secretGenerator` to the prod overlay that creates `db-credentials` from a local `db.env` file.
Should `db.env` be committed?

<details><summary>Show solution</summary>

```yaml
secretGenerator:
  - name: db-credentials
    envs:
      - db.env
```
**No** — never commit plain secrets. Use SOPS/Sealed Secrets/External Secrets for GitOps (module 24),
and add `db.env` to `.gitignore`.
</details>

## 🟡 Exercise 3 · A Kustomize component
Create a reusable component that adds a `PodDisruptionBudget` and enable it only in prod.

<details><summary>Show solution</summary>

```yaml
# components/pdb/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1alpha1
kind: Component
resources:
  - pdb.yaml
---
# components/pdb/pdb.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: shop
spec:
  maxUnavailable: 1
  selector:
    matchLabels: { app: shop }
```
```yaml
# overlays/prod/kustomization.yaml
components:
  - ../../components/pdb
```
</details>

## 🟡 Exercise 4 · Safer chart values
Write a `values.schema.json` that requires `image.tag` to be a non-empty string and `replicaCount` an
integer between 1 and 20.

<details><summary>Show solution</summary>

```json
{
  "$schema": "https://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["image", "replicaCount"],
  "properties": {
    "replicaCount": { "type": "integer", "minimum": 1, "maximum": 20 },
    "image": {
      "type": "object",
      "required": ["tag"],
      "properties": { "tag": { "type": "string", "minLength": 1 } }
    }
  }
}
```
`helm install/upgrade/lint/template` fail fast with a clear error when values do not match.
</details>

## 🔴 Challenge · CI pipeline for manifests
Design a CI job that validates every overlay and chart before merge.

<details><summary>Show solution</summary>

```bash
set -euo pipefail
for o in overlays/*/; do
  kubectl kustomize "$o" > "/tmp/$(basename $o).yaml"
done
helm lint charts/*
for c in charts/*/; do helm template ci "$c" -f "$c/ci-values.yaml" > "/tmp/chart-$(basename $c).yaml"; done
kubeconform -strict -summary -kubernetes-version 1.37.0 -ignore-missing-schemas /tmp/*.yaml
# policy checks (module 26), e.g.: kyverno apply policies/ --resource /tmp/*.yaml
# show the diff against the live cluster/GitOps target as a PR comment
```
</details>

---

## ❓ Quiz

1. What does `configMapGenerator` add to the ConfigMap name?
   a) A timestamp b) **A content hash** c) The namespace d) A version number

2. Where does Helm 3 store release history?
   a) **Secrets in the release namespace** b) Tiller c) The chart repository d) etcd directly

3. Which flag makes `helm upgrade` roll back automatically on failure?
   a) **`--atomic`** b) `--force` c) `--wait` d) `--rollback`

4. Kustomize is…
   a) A package registry b) A GitOps server c) A template engine d) **A template-free patching tool built into kubectl**

5. `helm template` …
   a) **Renders manifests locally without installing** b) Creates a chart c) Installs a release d) Lints values

<details><summary>Show answers</summary>

1-b · 2-a · 3-a · 4-d · 5-a
</details>

➡️ Next module: [24 · GitOps and CI/CD](../24-gitops-cicd/)
