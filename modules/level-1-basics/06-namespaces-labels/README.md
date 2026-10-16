<img src="../../../assets/kubernetes-icon-color.svg" alt="Kubernetes logo" width="56" align="right">

# 06 · Namespaces, Labels and Selectors

| Level | Duration | Prerequisites |
|---|---|---|
| 🧱 Basics | 4 h | Module 05 |

**By the end you will be able to:**
- **Organize** a cluster into namespaces and explain what they do and do not isolate.
- **Apply** labels following the recommended conventions.
- **Query** resources with equality-based and set-based selectors.
- **Distinguish** labels from annotations and know when to use each.

---

## 🧠 The analogy: an office building and colored stickers

A **namespace** is a **floor** of an office building. Each team gets its own floor: two teams can both
have a meeting room called "Room A" without confusion, and the building manager can give each floor a
budget (quotas) and access badges (RBAC). But the floors share the same elevators, plumbing and
electricity — they are not separate buildings.

**Labels** are **colored stickers** you put on things: 🔴 `env=prod`, 🔵 `team=payments`, 🟢 `app=checkout`.
A **selector** is the question "give me everything with a red *and* blue sticker". Kubernetes uses that
question everywhere: a Service finds its Pods, a Deployment counts its replicas, a NetworkPolicy picks
its targets — all by stickers, never by name.

**Annotations** are **sticky notes**: longer text for humans and tools ("owner: ana@example.com",
"last deployed by CI run #4521"). You cannot select by sticky notes.

---

## 📖 Concepts

### 1. Namespaces

A namespace is a scope for names. Inside one namespace, names must be unique per resource type.

| Built-in namespace | Contains |
|---|---|
| `default` | Where things go if you do not specify a namespace (avoid using it for real work) |
| `kube-system` | Control plane and core addons |
| `kube-public` | Readable by everyone, rarely used |
| `kube-node-lease` | Node heartbeat Leases |

**Namespaced vs cluster-scoped:** Pods, Services, Deployments, ConfigMaps, Secrets are namespaced.
Nodes, PersistentVolumes, StorageClasses, ClusterRoles, Namespaces themselves are cluster-scoped.

```bash
kubectl api-resources --namespaced=true  | head
kubectl api-resources --namespaced=false | head
```

**What a namespace gives you:** name scoping, RBAC boundaries, ResourceQuotas/LimitRanges, a target
for NetworkPolicies and Pod Security labels, and a convenient unit for deletion.

**What it does NOT give you:** network isolation (by default every Pod can reach every Pod), node
isolation, or protection against a noisy neighbor without quotas.

### 2. DNS and namespaces

A Service `api` in namespace `shop` is reachable as `api` from inside `shop`, and as
`api.shop` or `api.shop.svc.cluster.local` from anywhere else.

### 3. Labels

Labels are key/value pairs in `metadata.labels`. Keys can have an optional prefix: `app.kubernetes.io/name`.
Values are strings of up to 63 characters.

**Recommended labels** (used by tools like Helm, Argo CD and dashboards):

| Label | Example |
|---|---|
| `app.kubernetes.io/name` | `checkout` |
| `app.kubernetes.io/instance` | `checkout-eu` |
| `app.kubernetes.io/version` | `"2.4.1"` |
| `app.kubernetes.io/component` | `api` |
| `app.kubernetes.io/part-of` | `webshop` |
| `app.kubernetes.io/managed-by` | `helm` |

Add your own organizational labels too: `team`, `env`, `cost-center`, `tier`.

### 4. Selectors

| Type | Syntax | Meaning |
|---|---|---|
| Equality | `app=web` · `env!=prod` | Exactly / not equal |
| Set | `env in (dev,qa)` · `tier notin (cache)` | One of / none of |
| Existence | `team` · `!team` | Has the key / does not have it |

Combine with commas (logical AND): `-l 'app=web,env in (prod,staging)'`.

In manifests you will see two styles:

```yaml
# Services: only equality, as a simple map
selector:
  app: web

# Deployments, Jobs, NetworkPolicies…: matchLabels + matchExpressions
selector:
  matchLabels:
    app: web
  matchExpressions:
    - { key: env, operator: In, values: [prod, staging] }
    - { key: canary, operator: DoesNotExist }
```

### 5. Annotations

Annotations hold non-identifying metadata. They can be large (up to 256 KB in total per object) and
are read by tools: `kubectl.kubernetes.io/last-applied-configuration`,
`prometheus.io/scrape`, `deployment.kubernetes.io/revision`, Ingress controller settings, and so on.

### 6. Field selectors

Select by object fields, not labels: `--field-selector status.phase=Running,spec.nodeName=kc-worker`.

---

## 🗺️ Diagram: selectors glue everything together

```mermaid
flowchart LR
    subgraph NS["Namespace: shop"]
      D["Deployment checkout<br/>selector: app=checkout"]
      S["Service checkout<br/>selector: app=checkout"]
      NP["NetworkPolicy<br/>podSelector: app=checkout"]
      P1["Pod · app=checkout · version=v1"]
      P2["Pod · app=checkout · version=v1"]
      P3["Pod · app=checkout · version=v2"]
      X["Pod · app=cart"]
    end
    D -.manages.-> P1 & P2 & P3
    S -.routes to.-> P1 & P2 & P3
    NP -.protects.-> P1 & P2 & P3
    S -.x.- X
```

---

## ⌨️ Essential commands

| Command | What it does |
|---|---|
| `kubectl create namespace shop` | Create a namespace |
| `kubectl get pods -n shop` / `-A` | List in one / all namespaces |
| `kubectl get pods --show-labels` | Show all labels |
| `kubectl get pods -L app,env` | Labels as columns |
| `kubectl get pods -l 'env in (prod,qa),team=payments'` | Filter by selector |
| `kubectl label pod web env=prod` | Add a label |
| `kubectl label pod web env=qa --overwrite` | Change a label |
| `kubectl label pod web env-` | Remove a label |
| `kubectl annotate deploy web owner=ana@example.com` | Add an annotation |
| `kubectl delete pods -l app=old` | Delete by selector |

---

## 🧯 In production

- **Labels are your query language during incidents.** "Show me all `team=payments` Pods not Ready on
  zone-b" is one command — if your labels are consistent.
- **Enforce labels with policy** (module 26): reject workloads without `team` and `app.kubernetes.io/name`.
- **One namespace per app per environment** (e.g., `checkout-prod`) is a simple, effective convention.
- **Deleting a namespace deletes everything inside.** Protect important namespaces with RBAC and GitOps.
- **Never change a Deployment's `selector`**: it is immutable, and mismatched selectors orphan Pods.

## ⚠️ Common mistakes

| Symptom | Cause | Fix |
|---|---|---|
| Service has no endpoints | Selector does not match Pod labels (typo, case) | Compare `kubectl get svc -o wide` with `--show-labels` |
| `selector does not match template labels` | Deployment `selector` ≠ `template.metadata.labels` | Make them identical |
| Namespace stuck `Terminating` | A finalizer on some resource or an unavailable API | `kubectl get ns X -o yaml`, check conditions |
| Resource "disappears" | It was created in another namespace | `kubectl get <kind> -A \| grep name` |

## 🎓 Official tutorial companion

After the lab, run the upstream tutorial to see the same idea from the official docs:

- [Namespaces walkthrough](https://kubernetes.io/docs/tutorials/cluster-management/namespaces-walkthrough/)

> Tutorials that use minikube work on `kc` too; replace `minikube service` with `kubectl port-forward`.

## 📚 Official references

- [Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)
- [Labels and selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/)
- [Recommended labels](https://kubernetes.io/docs/concepts/overview/working-with-objects/common-labels/)
- [Annotations](https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/)
- [Field selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/field-selectors/)
- [Well-known labels, annotations and taints](https://kubernetes.io/docs/reference/labels-annotations-taints/)

➡️ Next: [🧪 LAB.md](LAB.md)
