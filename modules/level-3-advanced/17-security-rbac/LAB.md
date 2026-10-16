# 🧪 Lab 17 · Least Privilege in Practice

| Goal | Duration | Environment |
|---|---|---|
| Create a real user with a certificate, grant scoped access, and lock down a workload identity | 90 min | `kc` kind cluster + `openssl` |

## Setup

```bash
kubectl create namespace lab-17
mkdir -p ~/kc/lab17 && cd ~/kc/lab17
kubectl auth whoami
```

**Expected output:** you are `kubernetes-admin` in group `kubeadm:cluster-admins` (or `system:masters`).

## Step 1 · Create a user "ana" with a client certificate

```bash
openssl genrsa -out ana.key 2048
openssl req -new -key ana.key -out ana.csr -subj "/CN=ana/O=shop-devs"

cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: ana
spec:
  request: $(base64 < ana.csr | tr -d '\n')
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 86400            # 1 day
  usages: ["client auth"]
EOF

kubectl certificate approve ana
kubectl get csr ana -o jsonpath='{.status.certificate}' | base64 -d > ana.crt
openssl x509 -in ana.crt -noout -subject -enddate
```

✅ **Check:** subject `CN=ana, O=shop-devs`, valid for one day.

## Step 2 · Give ana her own kubeconfig context

```bash
kubectl config set-credentials ana --client-certificate=ana.crt --client-key=ana.key --embed-certs=true
kubectl config set-context ana@kc --cluster=kind-kc --user=ana --namespace=lab-17
kubectl --context ana@kc auth whoami
kubectl --context ana@kc get pods
```

✅ **Check:** `whoami` shows `ana` with group `shop-devs`; `get pods` is **Forbidden**. Authentication
works, authorization does not (yet).

## Step 3 · A namespaced Role for the group

Create `rbac.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
  namespace: lab-17
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log", "services", "configmaps"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  - apiGroups: ["apps"]
    resources: ["deployments/scale"]
    verbs: ["update", "patch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: shop-devs-developer
  namespace: lab-17
subjects:
  - kind: Group
    name: shop-devs
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl apply -f rbac.yaml
kubectl --context ana@kc create deployment web --image=nginx:1.27
kubectl --context ana@kc scale deployment web --replicas=2
kubectl --context ana@kc get pods
kubectl --context ana@kc delete deployment web          # not allowed
kubectl --context ana@kc get secrets                    # not allowed
kubectl --context ana@kc get pods -n kube-system        # not allowed
```

✅ **Check:** create/scale/get work; delete, secrets and other namespaces are **Forbidden**.

## Step 4 · Audit with `can-i`

```bash
kubectl auth can-i --list -n lab-17 --as=ana --as-group=shop-devs
kubectl auth can-i exec pods -n lab-17 --as=ana --as-group=shop-devs     # no
```

## Step 5 · A workload identity

Create `sa.yaml`:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: pod-watcher
  namespace: lab-17
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-watcher-view
  namespace: lab-17
subjects:
  - kind: ServiceAccount
    name: pod-watcher
    namespace: lab-17
roleRef:
  kind: ClusterRole
  name: view                      # built-in, limited to this namespace by the RoleBinding
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: v1
kind: Pod
metadata:
  name: watcher
  namespace: lab-17
spec:
  serviceAccountName: pod-watcher
  containers:
    - name: client
      image: curlimages/curl:8.10.1
      command: ["sleep", "3600"]
```

```bash
kubectl apply -f sa.yaml
kubectl -n lab-17 wait --for=condition=Ready pod/watcher
kubectl -n lab-17 exec -it watcher -- sh
```

Inside the Pod, call the API with the mounted token — exactly what client libraries do:

```bash
SA=/var/run/secrets/kubernetes.io/serviceaccount
API="https://kubernetes.default.svc/api/v1/namespaces/lab-17/pods"
AUTH="Authorization: Bearer $(cat $SA/token)"
curl -s --cacert $SA/ca.crt -H "$AUTH" $API | grep '"name"' | head -3          # allowed (view)
curl -s --cacert $SA/ca.crt -H "$AUTH" -X DELETE $API/watcher | grep -E 'reason|message'   # Forbidden
cut -d. -f2 $SA/token | base64 -d 2>/dev/null; echo                           # token payload
exit
```

✅ **Check:** listing Pods works, `DELETE` returns `"reason": "Forbidden"`, and the decoded token payload shows an `exp`
(expiry) about one hour ahead and `kubernetes.io` claims binding it to this Pod.

## Step 6 · Opt out of tokens

```bash
kubectl -n lab-17 run no-token --image=busybox:1.36 --overrides='{"spec":{"automountServiceAccountToken":false}}' -- sleep 3600
kubectl -n lab-17 wait --for=condition=Ready pod/no-token
kubectl -n lab-17 exec no-token -- ls /var/run/secrets/kubernetes.io/serviceaccount 2>&1 | head -1
```

✅ **Check:** `No such file or directory` — this Pod cannot talk to the API at all.

## 💥 Step 7 · Break it on purpose: a "harmless" permission

A teammate adds this rule to `developer` "so devs can debug":

```bash
kubectl -n lab-17 patch role developer --type=json -p='[{"op":"add","path":"/rules/-","value":{"apiGroups":[""],"resources":["pods"],"verbs":["create"]}}]'
kubectl -n lab-17 create secret generic db-root --from-literal=password=TopSecret!
kubectl --context ana@kc get secret db-root                  # still forbidden… right?
```

**Diagnosis:** ana cannot `get secrets`. Show how the new permission lets her read `db-root` anyway, then fix it.

<details><summary>Show solution</summary>

Creating a Pod lets you **mount any Secret** in the namespace:
```bash
cat <<'EOF' | kubectl --context ana@kc apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: sneaky
spec:
  restartPolicy: Never
  containers:
    - name: c
      image: busybox:1.36
      command: ["cat", "/s/password"]
      volumeMounts: [{ name: s, mountPath: /s }]
  volumes:
    - name: s
      secret: { secretName: db-root }
EOF
sleep 5; kubectl --context ana@kc logs sneaky      # TopSecret!
```
`create pods` is effectively "read every Secret and use every ServiceAccount in this namespace".
Fixes: keep secrets in namespaces where only the owning app can create Pods; restrict with admission
policies (module 26); or use external secret stores. Revert the rule:
```bash
kubectl apply -f rbac.yaml
```
</details>

## 🧹 Cleanup

```bash
kubectl delete namespace lab-17
kubectl delete csr ana
kubectl config delete-context ana@kc; kubectl config delete-user ana
```

➡️ Next: [📝 EXERCISES.md](EXERCISES.md)
