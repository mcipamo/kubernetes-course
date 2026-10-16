# 📝 Exercises 26 · Admission Policies

## 🟢 Exercise 1 · Mutating or validating?
Classify: (a) inject a service-mesh sidecar; (b) reject privileged containers; (c) add default
resource requests; (d) reject images not signed by the company key.

<details><summary>Show solution</summary>

(a) Mutating · (b) Validating · (c) Mutating (e.g., LimitRanger or a mutation policy) · (d) Validating
(Kyverno `verifyImages` also mutates tags to digests)
</details>

## 🟢 Exercise 2 · Allowed registries in CEL
Write a validation expression that only allows images from `ghcr.io/acme/` and `registry.k8s.io/`.

<details><summary>Show solution</summary>

```yaml
validations:
  - expression: >
      object.spec.template.spec.containers.all(c,
        c.image.startsWith('ghcr.io/acme/') || c.image.startsWith('registry.k8s.io/'))
    message: "images must come from approved registries"
```
Remember `initContainers` and `ephemeralContainers` too (use `has()` guards).
</details>

## 🟡 Exercise 3 · Skip system users
Add a `matchConditions` entry so the policy ignores requests from `system:` users and controllers in `kube-system`.

<details><summary>Show solution</summary>

```yaml
spec:
  matchConditions:
    - name: not-system
      expression: "!request.userInfo.username.startsWith('system:')"
```
Match conditions are evaluated before validations; non-matching requests skip the policy entirely.
</details>

## 🟡 Exercise 4 · Kyverno equivalent
Write the Kyverno `ClusterPolicy` that requires a `team` label on Deployments, in `Audit` mode.

<details><summary>Show solution</summary>

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-team-label
spec:
  validationFailureAction: Audit
  background: true
  rules:
    - name: check-team
      match:
        any:
          - resources:
              kinds: ["Deployment"]
      validate:
        message: "every Deployment needs a 'team' label"
        pattern:
          metadata:
            labels:
              team: "?*"
```
`background: true` also reports existing violations in PolicyReports.
</details>

## 🔴 Challenge · A platform policy pack
Design the baseline policy pack for a multi-tenant platform: list 8 rules, the engine you would use for
each, and the rollout mode.

<details><summary>Show solution</summary>

| # | Rule | Engine | Initial mode |
|---|---|---|---|
| 1 | Pod Security `restricted` (exceptions per namespace) | Pod Security Admission | warn + audit → enforce |
| 2 | Required labels: `team`, `app.kubernetes.io/name` | VAP (CEL) | Warn → Deny |
| 3 | No `:latest` / tag required | VAP | Warn → Deny |
| 4 | Memory requests required, CPU requests required | VAP | Warn → Deny |
| 5 | Only approved registries | VAP | Audit → Deny |
| 6 | Signed images (cosign keyless, company identity) | Kyverno `verifyImages` / sigstore policy-controller | Audit → Enforce |
| 7 | `LoadBalancer` Services and `hostPath` only in approved namespaces | VAP with params | Deny |
| 8 | Default NetworkPolicy and ResourceQuota in every new namespace | Kyverno `generate` | Enforce |

Every rule: exclude `kube-system`/platform namespaces, documented exception process, tests in CI,
violation dashboards per team.
</details>

---

## ❓ Quiz

1. Which runs first in the admission chain?
   a) Validating webhooks b) **Mutating admission** c) etcd write d) ValidatingAdmissionPolicy

2. ValidatingAdmissionPolicies are written in…
   a) Rego b) Go c) **CEL** d) Lua

3. A `ValidatingAdmissionPolicy` without a binding…
   a) Applies cluster-wide b) **Has no effect** c) Blocks everything d) Applies only to Pods

4. A webhook with `failurePolicy: Fail` is unreachable. Matching requests are…
   a) **Rejected** b) Retried forever c) Queued d) Allowed

5. Which `validationActions` lets users see violations without being blocked?
   a) **Warn** b) Deny c) Ignore d) Skip

<details><summary>Show answers</summary>

1-b · 2-c · 3-b · 4-a · 5-a
</details>

➡️ Next module: [27 · DRA, GPUs and AI Workloads](../27-dra-gpus-ai/)
