# 📝 Exercises 05 · Pods

## 🟢 Exercise 1 · Generate, don't type
Generate the YAML for a Pod `redis` using `redis:7.4` with label `tier=cache` without writing it by hand, then create it.

<details><summary>Show solution</summary>

```bash
kubectl run redis --image=redis:7.4 --labels=tier=cache --dry-run=client -o yaml > redis.yaml
kubectl apply -f redis.yaml
```
</details>

## 🟢 Exercise 2 · One-shot task
Run a Pod that prints the date once and stops, and confirm its phase is `Succeeded`.

<details><summary>Show solution</summary>

```bash
kubectl run once --image=busybox:1.36 --restart=Never -- date
kubectl get pod once -o jsonpath='{.status.phase}'; echo   # Succeeded
kubectl logs once
```
</details>

## 🟡 Exercise 3 · Shared localhost
Create a Pod with two containers: `nginx:1.27` and `curlimages/curl:8.10.1` running
`sh -c 'while true; do curl -s localhost >/dev/null && echo ok; sleep 5; done'`. Prove they share the network.

<details><summary>Show solution</summary>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: shared-net
spec:
  containers:
    - name: web
      image: nginx:1.27
    - name: probe
      image: curlimages/curl:8.10.1
      command: ["sh", "-c", "while true; do curl -s localhost >/dev/null && echo ok; sleep 5; done"]
```
```bash
kubectl logs shared-net -c probe -f    # ok, ok, ok…
```
</details>

## 🟡 Exercise 4 · Wait for a dependency
Write a Pod whose init container waits until the DNS name `db` resolves (`nslookup db`) before the app starts.
Then create a Service named `db` and watch the Pod start.

<details><summary>Show solution</summary>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: waits-for-db
spec:
  initContainers:
    - name: wait-db
      image: busybox:1.36
      command: ["sh", "-c", "until nslookup db; do echo waiting for db; sleep 2; done"]
  containers:
    - name: app
      image: nginx:1.27
```
```bash
kubectl apply -f waits.yaml; kubectl get pod waits-for-db     # Init:0/1
kubectl create service clusterip db --tcp=5432:5432
kubectl get pod waits-for-db -w                                # → Running
```
</details>

## 🔴 Challenge · Sidecar in a Job
Explain why a classic sidecar (a regular container) made Jobs never finish, and show the manifest
change that fixes it with native sidecars.

<details><summary>Show solution</summary>

A Job completes when **all** its containers exit. A log-shipper running `tail -F` never exits, so the
Pod stays `Running` forever. As a native sidecar (`initContainers` + `restartPolicy: Always`), the
kubelet ignores it for completion and stops it after the main container finishes.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: report
spec:
  template:
    spec:
      restartPolicy: Never
      initContainers:
        - name: shipper
          image: busybox:1.36
          restartPolicy: Always
          command: ["sh", "-c", "tail -F /logs/out.log"]
          volumeMounts: [{ name: logs, mountPath: /logs }]
      containers:
        - name: main
          image: busybox:1.36
          command: ["sh", "-c", "for i in 1 2 3; do echo line $i >> /logs/out.log; sleep 1; done"]
          volumeMounts: [{ name: logs, mountPath: /logs }]
      volumes:
        - { name: logs, emptyDir: {} }
```
</details>

---

## ❓ Quiz

1. Containers in the same Pod share…

   - a) The image
   - b) The network namespace (IP and ports)
   - c) Nothing
   - d) The filesystem

2. `CrashLoopBackOff` is…

   - a) An event type
   - b) A Pod phase
   - c) A reason for a waiting container
   - d) A node condition

3. How do you declare a native sidecar?

   - a) An annotation
   - b) `sidecar: true`
   - c) An init container with `restartPolicy: Always`
   - d) A second `containers` entry

4. Exit code 137 with reason `OOMKilled` means:

   - a) The image is missing
   - b) The app returned an error
   - c) The container exceeded its memory limit
   - d) The node rebooted

5. The best command to see why a container crashed **before** its current restart:

   - a) `kubectl logs`
   - b) `kubectl logs --previous`
   - c) `kubectl top`
   - d) `kubectl get events`

<details><summary>Show answers</summary>

1. **b)** The network namespace (IP and ports)
2. **c)** A reason for a waiting container
3. **c)** An init container with `restartPolicy: Always`
4. **c)** The container exceeded its memory limit
5. **b)** `kubectl logs --previous`
</details>

➡️ Next module: [06 · Namespaces, Labels and Selectors](../06-namespaces-labels/)
