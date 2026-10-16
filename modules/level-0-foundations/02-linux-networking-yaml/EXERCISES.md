# 📝 Exercises 02 · Linux, Networking and YAML

## 🟢 Exercise 1 · CIDR math
How many IP addresses are in `10.96.0.0/12`? What is the last one?

<details><summary>Show solution</summary>

`/12` leaves 32 − 12 = 20 bits → 2²⁰ = **1,048,576** addresses, from `10.96.0.0` to **`10.111.255.255`**.
This is the default Service CIDR in many clusters (including kind).
</details>

## 🟢 Exercise 2 · Fix the YAML
What is wrong with this snippet?

```text
metadata:
  name: shop
  labels:
   app: shop
    version: 1.10
```

<details><summary>Show solution</summary>

`app` and `version` are siblings but have different indentation, and `1.10` will be read as `1.1`.

```yaml
metadata:
  name: shop
  labels:
    app: shop
    version: "1.10"
```
Label values must be strings anyway, so quoting is required.
</details>

## 🟡 Exercise 3 · Multi-document file
Write one file `stack.yaml` that contains a Pod `api` and a Pod `worker` (both `busybox:1.36` running
`sleep 3600`). Use `yq` to print only the names.

<details><summary>Show solution</summary>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: api
spec:
  containers:
    - name: main
      image: busybox:1.36
      command: ["sleep", "3600"]
---
apiVersion: v1
kind: Pod
metadata:
  name: worker
spec:
  containers:
    - name: main
      image: busybox:1.36
      command: ["sleep", "3600"]
```
```bash
yq '.metadata.name' stack.yaml   # api, worker
```
</details>

## 🟡 Exercise 4 · Graceful shutdown
Write a shell entrypoint that traps `SIGTERM`, prints "draining…", waits 2 s and exits 0. Test it with
`docker stop`. What exit code do you get now?

<details><summary>Show solution</summary>

```bash
docker run -d --name graceful busybox:1.36 sh -c \
  'trap "echo draining...; sleep 2; exit 0" TERM; while true; do sleep 1; done'
docker stop graceful
docker logs graceful            # draining...
docker inspect graceful --format '{{.State.ExitCode}}'   # 0
docker rm graceful
```
</details>

## 🔴 Challenge · Debug ladder
Inside `netshoot`, a teammate says "`curl http://api:8080` hangs". Write the ordered list of commands
you would run to find out whether the problem is DNS, routing, the port or the app.

<details><summary>Show solution</summary>

1. `nslookup api` → does the name resolve? (DNS)
2. `ping -c1 <ip>` → is the IP reachable? (routing — note ICMP may be blocked)
3. `nc -zv <ip> 8080` → is the port open? (firewall / nothing listening)
4. `curl -v --max-time 5 http://<ip>:8080/` → does the app answer? (app layer)
5. On the server: `ss -tlnp` → is the app listening on `0.0.0.0:8080` or only `127.0.0.1`?

This bottom-up ladder (name → IP → port → app) is the same one you will use for Services in module 08.
</details>

---

## ❓ Quiz

1. Which exit code usually means "killed because of memory"?

   - a) 1
   - b) 0
   - c) 137
   - d) 143

2. Which character must never be used for YAML indentation?

   - a) Tab
   - b) Space
   - c) Dash
   - d) Colon

3. In a Kubernetes object, who writes the `status` field?

   - a) You
   - b) kubectl
   - c) The container
   - d) Kubernetes controllers

4. What protocol and port does DNS normally use?

   - a) TCP 80
   - b) UDP 53
   - c) UDP 443
   - d) TCP 22

5. What separates multiple YAML documents in one file?

   - a) `###`
   - b) A blank line
   - c) `---`
   - d) `===`

<details><summary>Show answers</summary>

1. **c)** 137
2. **a)** Tab
3. **d)** Kubernetes controllers
4. **b)** UDP 53
5. **c)** `---`
</details>

➡️ Next module: [03 · Your Local Lab](../03-local-lab/)
