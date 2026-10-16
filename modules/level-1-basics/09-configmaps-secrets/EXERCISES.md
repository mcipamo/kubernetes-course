# 📝 Exercises 09 · ConfigMaps and Secrets

## 🟢 Exercise 1 · From a file
Create a ConfigMap `nginx-conf` from a local `default.conf` file and mount it at `/etc/nginx/conf.d` in an nginx Pod.

<details><summary>Show solution</summary>

```bash
cat > default.conf <<'EOF'
server { listen 80; location / { return 200 "configured by ConfigMap\n"; } }
EOF
kubectl create configmap nginx-conf --from-file=default.conf
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-custom
spec:
  containers:
    - name: nginx
      image: nginx:1.27
      volumeMounts:
        - { name: conf, mountPath: /etc/nginx/conf.d }
  volumes:
    - name: conf
      configMap:
        name: nginx-conf
```
```bash
kubectl exec nginx-custom -- curl -s localhost   # configured by ConfigMap
```
</details>

## 🟢 Exercise 2 · base64 trap
Create a Secret with `kubectl create secret generic t --from-literal=token=abc` and another from a file
created with `echo abc > token`. Compare the decoded values. What is different?

<details><summary>Show solution</summary>

The file version contains `abc\n` (a trailing newline added by `echo`). Apps comparing tokens will fail.
Use `echo -n abc > token` or `printf abc > token`.
</details>

## 🟡 Exercise 3 · Private registry
Create an image pull Secret for `ghcr.io` and reference it so a Pod can pull a private image.

<details><summary>Show solution</summary>

```bash
kubectl create secret docker-registry ghcr --docker-server=ghcr.io \
  --docker-username=<user> --docker-password=<token>
```
```yaml
spec:
  imagePullSecrets:
    - name: ghcr
  containers:
    - name: app
      image: ghcr.io/<user>/<private-image>:1.0
```
Tip: attach it to the namespace's `default` ServiceAccount so every Pod gets it:
`kubectl patch serviceaccount default -p '{"imagePullSecrets":[{"name":"ghcr"}]}'`.
</details>

## 🟡 Exercise 4 · Config rollout with versioned names
Show how to change `LOG_LEVEL` from `info` to `debug` with a clean rollout and an easy rollback, using
immutable versioned ConfigMaps.

<details><summary>Show solution</summary>

1. Create `app-config-v2` (immutable) with `LOG_LEVEL=debug`.
2. Change the Deployment's `configMapRef.name` from `app-config-v1` to `app-config-v2` → template change → rollout.
3. Rollback = `kubectl rollout undo` → the previous ReplicaSet references `app-config-v1`, which still exists.
4. Garbage-collect old versions after a safe period.

Kustomize's `configMapGenerator` does this automatically by appending a content hash (module 23).
</details>

## 🔴 Challenge · Threat model
List four ways a Secret's value can leak in a typical cluster and one control for each.

<details><summary>Show solution</summary>

| Leak path | Control |
|---|---|
| etcd data or backups read directly | Encryption at rest with KMS v2; protect backups |
| A user/app with `list secrets` in the namespace | Least-privilege RBAC; audit logs |
| Committed to Git | Sealed Secrets / SOPS / External Secrets; secret scanning in CI |
| Printed in logs or env dumps (`/proc/<pid>/environ`, crash reports) | Mount as files, not env; scrub logs |
| Anyone who can create Pods in the namespace (can mount any Secret there) | Restrict Pod creation; namespace per team; admission policies |
</details>

---

## ❓ Quiz

1. Is base64 in a Secret a form of encryption?
   a) Only in etcd b) Only with `stringData` c) **No, it is only encoding** d) Yes

2. A ConfigMap consumed with `envFrom` changes. The running container…
   a) Sees the new value at once b) Gets it within 60 s c) Crashes d) **Keeps the old value until restarted**

3. Which volume option prevents automatic updates of a mounted ConfigMap?
   a) `defaultMode` b) **`subPath`** c) `items` d) `readOnly`

4. Where does the kubelet store mounted Secret files on the node?
   a) The image layer b) The node disk c) etcd d) **tmpfs (memory)**

5. What does `immutable: true` give you?
   a) Encryption b) Base64 encoding c) **Protection from edits and less load on the API server** d) Automatic rotation

<details><summary>Show answers</summary>

1-c · 2-d · 3-b · 4-d · 5-c
</details>

🎯 **End of Level 1.** Before moving on, take **Mock exam 1** (CKAD style): ask your tutor or a
classmate to break 5 things in your `kc` cluster using modules 04–09 and fix them in 30 minutes.

➡️ Next module: [10 · Resources, QoS and Quotas](../../level-2-intermediate/10-resources-qos/)
