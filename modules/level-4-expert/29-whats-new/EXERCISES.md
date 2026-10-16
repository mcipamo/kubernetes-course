# 📝 Exercises 29 · What's New in v1.35 – v1.37

## 🟢 Exercise 1 · Stages
Classify as alpha, beta or GA in v1.37: in-place Pod resize, user namespaces, HPA scale to zero, Metrics
API, numeric toleration operators.

<details><summary>Show solution</summary>

In-place Pod resize: **GA** (v1.35) · User namespaces: **GA** (v1.36) · HPA scale to zero: **Beta**
(v1.37) · Metrics API: **GA** (v1.37) · Numeric toleration operators: **Alpha** (v1.35)
</details>

## 🟢 Exercise 2 · Deprecation response
Service `externalIPs` is deprecated in v1.36. Write the three steps your platform team should take.

<details><summary>Show solution</summary>

1. **Find usage**: `kubectl get svc -A -o json | jq -r '.items[] | select(.spec.externalIPs) | "\(.metadata.namespace)/\(.metadata.name)"'`.
2. **Block new usage** with a ValidatingAdmissionPolicy (module 26) in Warn → Deny mode.
3. **Migrate** each owner to `LoadBalancer` Services or Gateway API, with deadlines before the removal version.
</details>

## 🟡 Exercise 3 · Read a KEP
Pick one feature from this module, open its KEP in `kubernetes/enhancements`, and summarize: the
problem, the API change, the graduation criteria and the stage per release.

<details><summary>Show solution</summary>

Example for **In-place Pod resize** (KEP-1287): problem — changing resources required recreating Pods;
API — mutable `resources` via the `resize` subresource, `resizePolicy`, resize status conditions;
graduation — e2e tests, metrics, feedback from users and runtimes; stages — alpha 1.27, beta 1.33, GA 1.35.
Your summary should follow the same structure for the feature you chose.
</details>

## 🟡 Exercise 4 · Managed cluster check
Your cloud provider runs v1.37. How do you verify whether a beta feature (e.g., Memory QoS) is enabled
in **your** cluster?

<details><summary>Show solution</summary>

```bash
kubectl get --raw /metrics | grep kubernetes_feature_enabled | grep -i memoryqos
kubectl get --raw "/api/v1/nodes/<node>/proxy/configz" | jq '.kubeletconfig.featureGates'
```
And read the provider's release notes: managed services sometimes disable betas or enable them later.
</details>

## 🔴 Challenge · Adoption plan
Write a one-page plan to adopt **user namespaces** (GA in v1.36) across all application namespaces.

<details><summary>Show solution</summary>

- **Why**: container root no longer maps to host root → container-escape impact drops.
- **Prerequisites**: v1.36+ on all nodes; kernel and runtime support for idmapped mounts (containerd/CRI-O
  versions); filesystem support for volumes in use.
- **Compatibility test**: run the main workloads with `hostUsers: false` in staging; watch for volume
  permission issues, apps needing host namespaces, privileged sidecars.
- **Rollout**: platform base charts default to `hostUsers: false`; exceptions via label + review;
  admission policy in Warn → Deny for app namespaces.
- **Measure**: % of Pods with user namespaces, incidents caused, exceptions remaining.
- **Rollback**: per-workload override; policy back to Warn.
</details>

---

## ❓ Quiz

1. A feature at **alpha** stage is…

   - a) Stable
   - b) Removed
   - c) Off by default, behind a feature gate
   - d) On by default

2. Which v1.36 GA feature maps container root to an unprivileged host user?

   - a) Volume group snapshots
   - b) Declarative validation
   - c) User namespaces
   - d) PSI metrics

3. Where can you see, from inside the cluster, which feature gates are on?

   - a) CoreDNS logs
   - b) etcd keys
   - c) `kubectl get nodes`
   - d) `kubernetes_feature_enabled` metrics

4. What happened to the community ingress-nginx controller?

   - a) It became GA
   - b) It moved into kube-proxy
   - c) Nothing
   - d) It was retired; Gateway API is the recommended path

5. Where are Kubernetes features designed and tracked?

   - a) KEPs in kubernetes/enhancements
   - b) Helm charts
   - c) Blog comments
   - d) Docker Hub

<details><summary>Show answers</summary>

1. **c)** Off by default, behind a feature gate
2. **c)** User namespaces
3. **d)** `kubernetes_feature_enabled` metrics
4. **d)** It was retired; Gateway API is the recommended path
5. **a)** KEPs in kubernetes/enhancements
</details>

🎓 **You finished the curriculum.** Time for the [Capstone](../../../capstone/).
