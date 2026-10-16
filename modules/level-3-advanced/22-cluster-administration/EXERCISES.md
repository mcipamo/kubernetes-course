# 📝 Exercises 22 · Cluster Administration

## 🟢 Exercise 1 · Version skew
The API server is at v1.37. Which are allowed? (a) kubelet v1.34; (b) kubelet v1.33; (c) scheduler
v1.38; (d) kubectl v1.36.

<details><summary>Show solution</summary>

(a) ✅ (3 minors older) · (b) ❌ (4 older) · (c) ❌ (never newer than the API server) · (d) ✅ (±1)
</details>

## 🟢 Exercise 2 · etcd quorum
How many members can fail in etcd clusters of 1, 3, 4 and 5 members?

<details><summary>Show solution</summary>

Quorum = ⌊n/2⌋ + 1 → tolerated failures: 1 → **0**, 3 → **1**, 4 → **1**, 5 → **2**. An even count adds
cost without adding tolerance, which is why you use 3 or 5.
</details>

## 🟡 Exercise 3 · Upgrade runbook
Write the checklist for upgrading a kubeadm cluster from v1.36 to v1.37.

<details><summary>Show solution</summary>

- [ ] Read v1.37 release notes, urgent upgrade notes and deprecations
- [ ] Find removed/deprecated API usage (metrics, `pluto`, `kubent`); migrate manifests and Helm charts
- [ ] Check addon compatibility (CNI, CSI, ingress/gateway, cert-manager, monitoring)
- [ ] Upgrade staging first and run smoke tests
- [ ] Take and verify an etcd snapshot; copy it off-cluster
- [ ] Check PDBs allow ≥ 1 disruption; check spare capacity for drains
- [ ] `kubeadm upgrade plan` → `apply` on the first control plane → `upgrade node` on the others
- [ ] Nodes one at a time: drain → upgrade kubelet/kubectl → restart → uncordon → check SLIs
- [ ] Verify versions, component health, workloads; close the change record
</details>

## 🟡 Exercise 4 · Audit policy
Write an audit policy that: skips events from `system:kube-proxy`; logs only metadata for Secrets and
ConfigMaps; logs full request/response for RBAC changes; logs metadata for everything else.

<details><summary>Show solution</summary>

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
omitStages: ["RequestReceived"]
rules:
  - level: None
    users: ["system:kube-proxy"]
  - level: Metadata
    resources:
      - group: ""
        resources: ["secrets", "configmaps"]
  - level: RequestResponse
    resources:
      - group: rbac.authorization.k8s.io
        resources: ["roles", "rolebindings", "clusterroles", "clusterrolebindings"]
  - level: Metadata
```
Rules are evaluated **in order**; the first match wins.
</details>

## 🔴 Challenge · Disaster recovery plan
Your single production cluster's control plane is lost (all three control-plane VMs deleted). Describe
two recovery strategies and what you need to have prepared **before** the disaster for each.

<details><summary>Show solution</summary>

**A. Restore the control plane from etcd backup**
- Needs: recent off-site etcd snapshots, the cluster **CA and PKI** (`/etc/kubernetes/pki`, including
  etcd and service-account keys), kubeadm config, the same control-plane endpoint (DNS/LB), and the
  encryption-at-rest key.
- Steps: build new control-plane VMs, restore PKI, `etcdutl snapshot restore`, start the control plane,
  let workers reconnect.

**B. Rebuild a fresh cluster and redeploy from Git**
- Needs: everything in GitOps (module 24): manifests, Helm values, cluster addons; secrets in an
  external manager; data in backups (Velero / managed databases / object storage); DNS automation.
- Steps: provision a new cluster (IaC), bootstrap GitOps, restore persistent data, switch DNS.

Mature teams prepare **both**, and practice them with game days. Define and measure RPO/RTO for each.
</details>

---

## ❓ Quiz

1. Which command evicts Pods while respecting PodDisruptionBudgets?
   a) `kubectl cordon` b) `kubectl taint` c) `kubectl delete node` d) **`kubectl drain`**

2. kubeadm leaf certificates are valid by default for…
   a) 90 days b) 10 years c) **1 year** d) 5 years

3. You must upgrade from 1.35 to 1.37. You should…
   a) **Go 1.35 → 1.36 → 1.37** b) Jump directly c) Reinstall d) Upgrade nodes first

4. Where must an etcd restore be written?
   a) Into a ConfigMap b) Into the API server c) **To a new data directory, then point etcd at it** d) Over the current data dir

5. After enabling encryption at rest, existing Secrets are…
   a) Deleted b) **Still unencrypted until rewritten** c) Moved to KMS d) Encrypted automatically

<details><summary>Show answers</summary>

1-d · 2-c · 3-a · 4-c · 5-b
</details>

🎯 **End of Level 3.** Do **Practice round 3** (CKA/CKS style): 12 tasks in 2 hours across modules 16–22,
including one etcd restore and one broken node.

➡️ Next module: [23 · Kustomize and Helm](../../level-4-expert/23-kustomize-helm/)
