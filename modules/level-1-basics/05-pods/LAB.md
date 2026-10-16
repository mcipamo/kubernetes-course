# 🧪 Lab 05 · Pods Inside Out

| Goal | Duration | Environment |
|---|---|---|
| Build single and multi-container Pods, and debug three classic failures | 75 min | `kc` kind cluster |

## Setup

```bash
kubectl create namespace lab-05
kubectl config set-context --current --namespace=lab-05
mkdir -p ~/kc/lab05 && cd ~/kc/lab05
```

## Step 1 · Your first declarative Pod

Create `podinfo.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: podinfo
  labels:
    app: podinfo
spec:
  containers:
    - name: app
      image: ghcr.io/stefanprodan/podinfo:6.7.0
      ports:
        - name: http
          containerPort: 9898
      env:
        - name: PODINFO_UI_MESSAGE
          value: "Hello from KC"
        - name: MY_NODE
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
```

```bash
kubectl apply -f podinfo.yaml
kubectl wait --for=condition=Ready pod/podinfo --timeout=60s
kubectl get pod podinfo -o wide
```

✅ **Check:** `kubectl exec podinfo -- printenv MY_NODE` prints the node name shown by `-o wide`.

## Step 2 · Talk to it

```bash
kubectl port-forward pod/podinfo 9898:9898 &
sleep 2; curl -s localhost:9898 | head -12; kill %1
```

**Expected output:** JSON including `"message": "Hello from KC"` and `"hostname": "podinfo"`.

## Step 3 · Pod IP and direct access from another Pod

```bash
POD_IP=$(kubectl get pod podinfo -o jsonpath='{.status.podIP}')
kubectl run curl --image=curlimages/curl:8.10.1 --rm -it --restart=Never -- curl -s $POD_IP:9898/version
```

✅ **Check:** you get `{"commit":"…","version":"6.7.0"}`. Every Pod can reach every other Pod by IP.

## Step 4 · Init container + native sidecar

Create `multi.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi
spec:
  initContainers:
    - name: prepare                       # classic init: runs once, first
      image: busybox:1.36
      command: ["sh", "-c", "echo '<h1>Prepared by init</h1>' > /work/index.html"]
      volumeMounts:
        - { name: web, mountPath: /work }
    - name: log-shipper                   # native sidecar: keeps running
      image: busybox:1.36
      restartPolicy: Always
      command: ["sh", "-c", "touch /logs/access.log; tail -F /logs/access.log"]
      volumeMounts:
        - { name: logs, mountPath: /logs }
  containers:
    - name: nginx
      image: nginx:1.27
      volumeMounts:
        - { name: web, mountPath: /usr/share/nginx/html }
        - { name: logs, mountPath: /var/log/nginx }
  volumes:
    - { name: web,  emptyDir: {} }
    - { name: logs, emptyDir: {} }
```

```bash
kubectl apply -f multi.yaml
kubectl get pod multi -w        # Init:0/2 → Init:1/2 → PodInitializing → 2/2 Running ; Ctrl+C
kubectl exec multi -c nginx -- curl -s localhost
kubectl logs multi -c log-shipper --tail=3
```

✅ **Check:** `curl` returns `Prepared by init`, and the `log-shipper` logs show the access line of that request.

> 💡 Why `2/2` and not `3/3`? The finished `prepare` init container is not counted; the sidecar is.

## Step 5 · Attach an ephemeral debug container

Many production images have no shell. `kubectl debug` adds a temporary container to the running Pod:

```bash
kubectl debug -it podinfo --image=busybox:1.36 --target=app -- ps
```

✅ **Check:** `ps` lists the `podinfo` process. The ephemeral container shares the process namespace
of `app`, and `kubectl get pod podinfo -o jsonpath='{.spec.ephemeralContainers[*].name}'` shows it.

## 💥 Step 6 · Break it on purpose (three times)

Apply all three broken Pods:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: broken-image
spec:
  containers:
    - name: app
      image: nginx:1.27-does-not-exist
---
apiVersion: v1
kind: Pod
metadata:
  name: broken-crash
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "echo 'connecting to db...'; sleep 2; echo 'FATAL: DB_HOST not set'; exit 1"]
---
apiVersion: v1
kind: Pod
metadata:
  name: broken-oom
spec:
  containers:
    - name: app
      image: polinux/stress:1.0.4
      command: ["stress", "--vm", "1", "--vm-bytes", "200M", "--vm-hang", "1"]
      resources:
        limits:
          memory: 64Mi
```

Save it as `broken.yaml`, then:

```bash
kubectl apply -f broken.yaml
sleep 45; kubectl get pods
```

For **each** Pod, find the root cause using only `get`, `describe` and `logs`.

<details><summary>Show solution</summary>

| Pod | Status | Evidence | Root cause |
|---|---|---|---|
| `broken-image` | `ImagePullBackOff` | `describe` → `Failed to pull image … not found` | The tag does not exist |
| `broken-crash` | `CrashLoopBackOff` | `logs --previous` → `FATAL: DB_HOST not set`, exit code 1 | App needs config (module 09) |
| `broken-oom` | `OOMKilled` → `CrashLoopBackOff` | `describe` → `Last State: Terminated, Reason: OOMKilled, Exit Code: 137` | Uses 200 MB with a 64 Mi limit |

```bash
kubectl describe pod broken-image | grep -A3 Events
kubectl logs broken-crash --previous
kubectl get pod broken-oom -o jsonpath='{.status.containerStatuses[0].lastState.terminated.reason}'
```
</details>

## 🧹 Cleanup

```bash
kubectl config set-context --current --namespace=default
kubectl delete namespace lab-05
```

➡️ Next: [📝 EXERCISES.md](EXERCISES.md)
