# 🧪 Lab 26 · Guardrails for the Whole Cluster

| Goal | Duration | Environment |
|---|---|---|
| Roll out CEL policies safely, parameterize them, mutate with CEL, and survive a broken webhook | 120 min | `kc` kind cluster |

## Setup

```bash
kubectl create namespace lab-26
kubectl label namespace lab-26 policy.kc.dev/enforce=true
kubectl config set-context --current --namespace=lab-26
mkdir -p ~/kc/lab26 && cd ~/kc/lab26
kubectl api-resources --api-group=admissionregistration.k8s.io
```

✅ **Check:** the list includes `validatingadmissionpolicies` (and, on recent versions, `mutatingadmissionpolicies`).

## Step 1 · A policy in Warn mode

Create `policy.yaml`:

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: workload-standards
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
      - apiGroups: ["apps"]
        apiVersions: ["v1"]
        operations: ["CREATE", "UPDATE"]
        resources: ["deployments"]
  validations:
    - expression: "has(object.metadata.labels) && 'team' in object.metadata.labels"
      message: "every Deployment needs a 'team' label"
    - expression: >
        object.spec.template.spec.containers.all(c,
          c.image.contains(':') && !c.image.endsWith(':latest'))
      message: "images must use a pinned tag (no ':latest', no missing tag)"
    - expression: >
        object.spec.template.spec.containers.all(c,
          has(c.resources) && has(c.resources.requests) && 'memory' in c.resources.requests)
      message: "every container must request memory"
---
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: workload-standards
spec:
  policyName: workload-standards
  validationActions: [Warn]
  matchResources:
    namespaceSelector:
      matchLabels:
        policy.kc.dev/enforce: "true"
```

```bash
kubectl apply -f policy.yaml
kubectl create deployment sloppy --image=nginx
```

✅ **Check:** the Deployment is **created**, but kubectl prints three `Warning:` lines. Users learn the
rules before they are enforced.

## Step 2 · Switch to Deny

```bash
kubectl patch validatingadmissionpolicybinding workload-standards --type=merge \
  -p '{"spec":{"validationActions":["Deny"]}}'
kubectl create deployment sloppy2 --image=nginx
```

✅ **Check:** rejected with `ValidatingAdmissionPolicy 'workload-standards' with binding … denied request`.

Now a compliant one:

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tidy
  labels: { team: payments }
spec:
  replicas: 1
  selector: { matchLabels: { app: tidy } }
  template:
    metadata: { labels: { app: tidy } }
    spec:
      containers:
        - name: web
          image: nginx:1.27
          resources:
            requests: { memory: 64Mi }
EOF
```

✅ **Check:** `deployment.apps/tidy created`. Other namespaces (without the label) are not affected.

## Step 3 · Parameters: a different replica limit per namespace

Create `params.yaml`:

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: max-replicas
spec:
  failurePolicy: Fail
  paramKind:
    apiVersion: v1
    kind: ConfigMap
  matchConstraints:
    resourceRules:
      - apiGroups: ["apps"]
        apiVersions: ["v1"]
        operations: ["CREATE", "UPDATE"]
        resources: ["deployments"]
  validations:
    - expression: "object.spec.replicas <= int(params.data.maxReplicas)"
      messageExpression: "'replicas must be <= ' + params.data.maxReplicas"
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: replica-limits
  namespace: lab-26
data:
  maxReplicas: "3"
---
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: max-replicas-lab-26
spec:
  policyName: max-replicas
  validationActions: [Deny]
  paramRef:
    name: replica-limits
    namespace: lab-26
    parameterNotFoundAction: Deny
  matchResources:
    namespaceSelector:
      matchLabels:
        kubernetes.io/metadata.name: lab-26
```

```bash
kubectl apply -f params.yaml
kubectl scale deployment tidy --replicas=5
kubectl patch configmap replica-limits --type=merge -p '{"data":{"maxReplicas":"6"}}'
kubectl scale deployment tidy --replicas=5
```

✅ **Check:** the first scale is rejected (`replicas must be <= 3`); after raising the parameter, it succeeds — no policy change.

> `kubectl scale` uses the `/scale` subresource. If it is not blocked on your version, test with
> `kubectl patch deployment tidy -p '{"spec":{"replicas":5}}'`.

## Step 4 · Mutation with CEL (if your cluster serves it)

```bash
MAP_VERSION=$(kubectl api-resources --api-group=admissionregistration.k8s.io -o wide | awk '/mutatingadmissionpolicies /{print $3}')
echo "MutatingAdmissionPolicy served as: ${MAP_VERSION:-not available}"
```

If available, create `mutate.yaml` (adjust `apiVersion` to the served version):

```yaml
apiVersion: admissionregistration.k8s.io/v1beta1
kind: MutatingAdmissionPolicy
metadata:
  name: stamp-pods
spec:
  failurePolicy: Fail
  reinvocationPolicy: Never
  matchConstraints:
    resourceRules:
      - apiGroups: [""]
        apiVersions: ["v1"]
        operations: ["CREATE"]
        resources: ["pods"]
  mutations:
    - patchType: ApplyConfiguration
      applyConfiguration:
        expression: >
          Object{ metadata: Object.metadata{ labels: {"kc.dev/admitted": "true"} } }
---
apiVersion: admissionregistration.k8s.io/v1beta1
kind: MutatingAdmissionPolicyBinding
metadata:
  name: stamp-pods
spec:
  policyName: stamp-pods
  matchResources:
    namespaceSelector:
      matchLabels:
        kubernetes.io/metadata.name: lab-26
```

```bash
kubectl apply -f mutate.yaml
kubectl run stamped --image=nginx:1.27
kubectl get pod stamped --show-labels
```

✅ **Check:** the Pod carries `kc.dev/admitted=true` although you never set it.

## 💥 Step 5 · Break it on purpose: the webhook that took down a namespace

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: imaginary-security-scanner
webhooks:
  - name: scan.security.kc.dev
    admissionReviewVersions: ["v1"]
    sideEffects: None
    failurePolicy: Fail
    timeoutSeconds: 5
    clientConfig:
      service:
        name: scanner            # this Service does not exist
        namespace: lab-26
        path: /validate
    rules:
      - apiGroups: [""]
        apiVersions: ["v1"]
        operations: ["CREATE"]
        resources: ["pods"]
    namespaceSelector:           # scoped to the lab — imagine it were not!
      matchLabels:
        kubernetes.io/metadata.name: lab-26
EOF
kubectl rollout restart deployment tidy
sleep 15; kubectl get pods -l app=tidy
kubectl describe rs -l app=tidy | grep -i webhook | tail -2
```

**Diagnosis:** why can no Pod be created in `lab-26`? What would have happened without the
`namespaceSelector`? How do you recover?

<details><summary>Show solution</summary>

Every Pod CREATE in `lab-26` calls the webhook; the Service does not exist, and `failurePolicy: Fail`
turns "cannot reach the webhook" into "reject the request":
`failed calling webhook "scan.security.kc.dev": … service "scanner" not found`.

Without the selector, **every namespace — including `kube-system`** — would be unable to create Pods:
no CoreDNS restarts, no CNI Pods on new nodes, no recovery of crashed components. That is how admission
webhooks cause full-cluster outages.

Recover by removing or scoping the configuration (you can, because webhooks do not intercept their own
configuration objects):
```bash
kubectl delete validatingwebhookconfiguration imaginary-security-scanner
kubectl get pods -l app=tidy -w
```
Prevention: exclude system namespaces, run webhooks HA with PDBs, short timeouts, alert on webhook
error rate, and prefer in-process CEL policies when possible.
</details>

## 🧹 Cleanup

```bash
kubectl delete validatingadmissionpolicybinding workload-standards max-replicas-lab-26
kubectl delete validatingadmissionpolicy workload-standards max-replicas
kubectl delete -f mutate.yaml --ignore-not-found 2>/dev/null
kubectl config set-context --current --namespace=default
kubectl delete namespace lab-26
```

➡️ Next: [📝 EXERCISES.md](EXERCISES.md)
