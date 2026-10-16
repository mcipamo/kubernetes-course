# 🧪 Lab 09 · Configure an App Without Rebuilding It

| Goal | Duration | Environment |
|---|---|---|
| Inject config as env and files, observe live updates and see where Secrets really live | 75 min | `kc` kind cluster |

## Setup

```bash
kubectl create namespace lab-09
kubectl config set-context --current --namespace=lab-09
mkdir -p ~/kc/lab09 && cd ~/kc/lab09
```

## Step 1 · Create the configuration

Create `config.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  PODINFO_UI_MESSAGE: "Configured by a ConfigMap"
  PODINFO_UI_COLOR: "#34577c"
  settings.yaml: |
    cache:
      ttl: 300
    features:
      newCheckout: false
---
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
stringData:
  DB_USER: shop
  DB_PASSWORD: s3cr3t-change-me
```

```bash
kubectl apply -f config.yaml
kubectl get secret db-credentials -o yaml | grep -A2 '^data'
```

✅ **Check:** the Secret shows `DB_PASSWORD: czNjcjN0LWNoYW5nZS1tZQ==`. Decode it:
`kubectl get secret db-credentials -o jsonpath='{.data.DB_PASSWORD}' | base64 -d`. That is how "secret" base64 is.

## Step 2 · Consume it

Create `app.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  replicas: 1
  selector:
    matchLabels: { app: app }
  template:
    metadata:
      labels: { app: app }
    spec:
      containers:
        - name: app
          image: ghcr.io/stefanprodan/podinfo:6.7.0
          envFrom:
            - configMapRef:
                name: app-config        # every key becomes an env var (PODINFO_UI_MESSAGE, PODINFO_UI_COLOR…)
          env:
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef: { name: db-credentials, key: DB_PASSWORD }
          volumeMounts:
            - name: settings
              mountPath: /etc/app
              readOnly: true
            - name: db
              mountPath: /var/run/secrets/db
              readOnly: true
      volumes:
        - name: settings
          configMap:
            name: app-config
            items:
              - { key: settings.yaml, path: settings.yaml }
        - name: db
          secret:
            secretName: db-credentials
            defaultMode: 0400
```

```bash
kubectl apply -f app.yaml
kubectl rollout status deployment/app
POD=$(kubectl get pod -l app=app -o jsonpath='{.items[0].metadata.name}')
kubectl exec $POD -- printenv PODINFO_UI_MESSAGE DB_PASSWORD
kubectl exec $POD -- cat /etc/app/settings.yaml
kubectl exec $POD -- ls -l /var/run/secrets/db
```

✅ **Check:** env vars are set, `settings.yaml` is a file, and each Secret key is a file with mode `-r--------`.

## Step 3 · Live update: files vs env

```bash
kubectl patch configmap app-config --type=merge -p '{"data":{"PODINFO_UI_MESSAGE":"Changed!","settings.yaml":"cache:\n  ttl: 60\n"}}'
sleep 70
kubectl exec $POD -- cat /etc/app/settings.yaml      # updated
kubectl exec $POD -- printenv PODINFO_UI_MESSAGE     # still the OLD value
```

✅ **Check:** the file shows `ttl: 60`, the env var still says `Configured by a ConfigMap`.

```bash
kubectl rollout restart deployment/app && kubectl rollout status deployment/app
kubectl exec deploy/app -- printenv PODINFO_UI_MESSAGE   # Changed!
```

## Step 4 · Where Secrets live on the node

```bash
NODE=$(kubectl get pod -l app=app -o jsonpath='{.items[0].spec.nodeName}')
docker exec $NODE sh -c 'mount | grep "secrets/db" | head -2'
```

✅ **Check:** the Secret volume is a **tmpfs** (memory), not a disk.

## Step 5 · Where Secrets live in etcd (unencrypted!)

```bash
kubectl -n kube-system exec etcd-kc-control-plane -- etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/secrets/lab-09/db-credentials | strings | grep -i s3cr3t
```

**Expected output:** `s3cr3t-change-me` in clear text. Anyone with access to etcd or its backups can
read every Secret. Module 22 shows how to enable encryption at rest.

## Step 6 · Immutable config

```bash
kubectl create configmap app-config-v2 --from-literal=PODINFO_UI_MESSAGE="v2 config" --dry-run=client -o yaml \
  | yq '.immutable = true' | kubectl apply -f -
kubectl patch configmap app-config-v2 --type=merge -p '{"data":{"PODINFO_UI_MESSAGE":"try to change"}}'
```

✅ **Check:** the patch is rejected: `field is immutable when immutable is set`.

## 💥 Step 7 · Break it on purpose

```bash
kubectl set env deployment/app --from=configmap/app-config-v3
kubectl get pods -l app=app
```

**Diagnosis:** what is the Pod status and why? How would you make the reference non-fatal?

<details><summary>Show solution</summary>

`kubectl set env --from` fails right away because `app-config-v3` does not exist. Simulate what a bad
manifest would do instead:
```bash
kubectl patch deployment app --type=json -p='[{"op":"add","path":"/spec/template/spec/containers/0/envFrom/-","value":{"configMapRef":{"name":"app-config-v3"}}}]'
kubectl get pods -l app=app        # new Pod: CreateContainerConfigError
kubectl describe pod -l app=app | grep -i 'not found'
```
The new Pod is stuck in **`CreateContainerConfigError`**: `configmap "app-config-v3" not found`.
Thanks to the rolling update, the old Pod keeps running. Fix: create the ConfigMap, roll back
(`kubectl rollout undo deployment/app`), or mark it optional: `{"configMapRef":{"name":"app-config-v3","optional":true}}`.
</details>

## 🧹 Cleanup

```bash
kubectl config set-context --current --namespace=default
kubectl delete namespace lab-09
```

➡️ Next: [📝 EXERCISES.md](EXERCISES.md)
