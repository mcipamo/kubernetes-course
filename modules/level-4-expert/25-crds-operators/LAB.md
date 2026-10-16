# 🧪 Lab 25 · Build Your Own Kubernetes API

| Goal | Duration | Environment |
|---|---|---|
| Create a `Website` CRD with CEL validation, then write a working controller in ~40 lines of bash | 120 min | `kc` kind cluster · `jq` |

## Setup

```bash
kubectl create namespace lab-25
kubectl config set-context --current --namespace=lab-25
mkdir -p ~/kc/lab25 && cd ~/kc/lab25
```

## Step 1 · Define the new resource type

Create `crd.yaml`:

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: websites.kc.dev
spec:
  group: kc.dev
  scope: Namespaced
  names:
    kind: Website
    plural: websites
    singular: website
    shortNames: [web]
  versions:
    - name: v1alpha1
      served: true
      storage: true
      subresources:
        status: {}
      additionalPrinterColumns:
        - { name: Replicas, type: integer, jsonPath: .spec.replicas }
        - { name: Ready, type: integer, jsonPath: .status.readyReplicas }
        - { name: Tier, type: string, jsonPath: .spec.tier }
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              required: [message]
              properties:
                message: { type: string, maxLength: 100 }
                replicas: { type: integer, minimum: 1, maximum: 10, default: 1 }
                tier: { type: string, enum: [free, pro], default: free }
              x-kubernetes-validations:
                - rule: "self.tier != 'free' || self.replicas <= 2"
                  message: "free tier allows at most 2 replicas"
            status:
              type: object
              properties:
                readyReplicas: { type: integer }
```

```bash
kubectl apply -f crd.yaml
kubectl api-resources --api-group=kc.dev
kubectl explain website.spec
```

✅ **Check:** `websites` (short name `web`) appears, and `kubectl explain` documents your fields.

## Step 2 · Validation in action

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: kc.dev/v1alpha1
kind: Website
metadata:
  name: blog
spec:
  message: "Hello from a custom resource"
  replicas: 2
EOF
kubectl get web
kubectl get website blog -o jsonpath='{.spec}'; echo

# Now three invalid ones
kubectl patch website blog --type=merge -p '{"spec":{"replicas":5}}'
kubectl patch website blog --type=merge -p '{"spec":{"tier":"enterprise"}}'
kubectl apply -f - <<'EOF'
apiVersion: kc.dev/v1alpha1
kind: Website
metadata:
  name: nomsg
spec:
  replicas: 1
EOF
```

✅ **Check:** the default `tier: free` was added; the three changes are rejected with `free tier allows
at most 2 replicas`, `Unsupported value: "enterprise"` and `spec.message: Required value`. The API server
enforced all of it — no code written yet.

## Step 3 · Write the controller

Create `controller.sh`:

```bash
#!/usr/bin/env bash
# A tiny, level-triggered controller: Website -> Deployment + Service. For learning only.
set -uo pipefail
NS="${NS:-lab-25}"

reconcile() {
  local name="$1" json uid replicas message ready
  json=$(kubectl -n "$NS" get website "$name" -o json 2>/dev/null) || return 0   # deleted: GC handles children
  uid=$(jq -r .metadata.uid <<<"$json")
  replicas=$(jq -r .spec.replicas <<<"$json")
  message=$(jq -r .spec.message <<<"$json")

  # Desired children, owned by the Website (garbage-collected when it is deleted)
  kubectl -n "$NS" apply --server-side --field-manager=website-controller -f - >/dev/null <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: $name
  ownerReferences: [{ apiVersion: kc.dev/v1alpha1, kind: Website, name: $name, uid: $uid, controller: true }]
spec:
  replicas: $replicas
  selector: { matchLabels: { website: $name } }
  template:
    metadata: { labels: { website: $name } }
    spec:
      containers:
        - name: web
          image: ghcr.io/stefanprodan/podinfo:6.7.0
          env: [{ name: PODINFO_UI_MESSAGE, value: "$message" }]
---
apiVersion: v1
kind: Service
metadata:
  name: $name
  ownerReferences: [{ apiVersion: kc.dev/v1alpha1, kind: Website, name: $name, uid: $uid, controller: true }]
spec:
  selector: { website: $name }
  ports: [{ port: 80, targetPort: 9898 }]
EOF

  # Report status through the /status subresource
  ready=$(kubectl -n "$NS" get deploy "$name" -o jsonpath='{.status.readyReplicas}' 2>/dev/null)
  kubectl -n "$NS" patch website "$name" --subresource=status --type=merge \
    -p "{\"status\":{\"readyReplicas\":${ready:-0}}}" >/dev/null
  echo "$(date +%T) reconciled $name (replicas=$replicas ready=${ready:-0})"
}

echo "website-controller watching namespace $NS"
while true; do                                     # level-triggered: re-check everything periodically
  for w in $(kubectl -n "$NS" get websites -o jsonpath='{.items[*].metadata.name}'); do
    reconcile "$w"
  done
  sleep 5
done
```

```bash
chmod +x controller.sh
./controller.sh          # leave it running in this terminal
```

## Step 4 · Watch it work (second terminal)

```bash
kubectl -n lab-25 get web,deploy,svc,pods
kubectl -n lab-25 run c --image=curlimages/curl:8.10.1 --rm -it --restart=Never -- curl -s blog | grep message
kubectl -n lab-25 patch website blog --type=merge -p '{"spec":{"message":"Updated by the operator"}}'
sleep 15; kubectl -n lab-25 get web
```

✅ **Check:** the controller created a Deployment and Service, the `READY` column fills in, and the
message change rolls out automatically.

## Step 5 · Self-healing and garbage collection

```bash
kubectl -n lab-25 delete deployment blog          # someone deletes a child…
sleep 10; kubectl -n lab-25 get deploy blog        # …the controller recreates it
kubectl -n lab-25 delete website blog              # delete the parent
sleep 5; kubectl -n lab-25 get deploy,svc          # children are garbage-collected
```

✅ **Check:** the deleted Deployment comes back; deleting the Website removes the Deployment and the
Service via **owner references** — the controller did not have to delete anything.

## 💥 Step 6 · Break it on purpose: a finalizer without a controller

```bash
cat <<'EOF' | kubectl -n lab-25 apply -f -
apiVersion: kc.dev/v1alpha1
kind: Website
metadata:
  name: shop
  finalizers: ["kc.dev/cleanup-dns"]
spec:
  message: "has a finalizer"
EOF
# Stop the controller (Ctrl+C in its terminal), then:
kubectl -n lab-25 delete website shop --wait=false
sleep 5; kubectl -n lab-25 get website shop -o jsonpath='{.metadata.deletionTimestamp}{"  "}{.metadata.finalizers}'; echo
kubectl delete namespace lab-25 --wait=false; sleep 10; kubectl get namespace lab-25
```

**Diagnosis:** why is the Website (and now the namespace) stuck in `Terminating`? What is the right
fix, and when is removing the finalizer by hand acceptable?

<details><summary>Show solution</summary>

A finalizer tells the API server "do not delete this until my controller has cleaned up". The object
gets a `deletionTimestamp` but stays until the finalizer list is empty. Nobody removes
`kc.dev/cleanup-dns`, so it hangs — and the namespace cannot finish deleting while it contains objects.

Right fix: run the controller that owns the finalizer so it performs the cleanup and removes it. Manual
removal is acceptable **only** if you have verified the external cleanup is done (or not needed):
```bash
kubectl -n lab-25 patch website shop --type=merge -p '{"metadata":{"finalizers":null}}'
kubectl get namespace lab-25     # disappears shortly
```
</details>

## 🧹 Cleanup

```bash
kubectl config set-context --current --namespace=default
kubectl delete namespace lab-25 --ignore-not-found
kubectl delete crd websites.kc.dev
```

➡️ Next: [📝 EXERCISES.md](EXERCISES.md)
