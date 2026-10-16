# 📝 Exercises 01 · Containers and Why Kubernetes

## 🟢 Exercise 1 · Tags
Build `kc/hello:1.1` with the message changed to "Hello, Kubernetes". Run versions 1.0 and 1.1 at the
same time on different ports.

<details><summary>Show solution</summary>

```bash
sed -i.bak 's/Hello from/Hello, Kubernetes, from/' app.py
docker build -t kc/hello:1.1 .
docker run -d --name v10 -p 8010:8000 kc/hello:1.0
docker run -d --name v11 -p 8011:8000 kc/hello:1.1
curl localhost:8010; curl localhost:8011
docker rm -f v10 v11
```
</details>

## 🟢 Exercise 2 · Digests
Get the digest of `nginx:1.27` and run it by digest instead of by tag.

<details><summary>Show solution</summary>

```bash
docker pull nginx:1.27
docker inspect nginx:1.27 --format '{{index .RepoDigests 0}}'
# nginx@sha256:...
docker run --rm -d --name pinned -p 8080:80 nginx@sha256:<digest>
docker rm -f pinned
```
A digest guarantees you always run exactly the same bytes.
</details>

## 🟡 Exercise 3 · A smaller image
Rewrite the Dockerfile with a smaller base image (or a multi-stage build) and compare sizes.

<details><summary>Show solution</summary>

```dockerfile
FROM python:3.12-alpine
WORKDIR /app
COPY app.py .
USER 1000
CMD ["python", "app.py"]
```
```bash
docker build -t kc/hello:alpine .
docker images kc/hello
```
Alpine usually ends up below 60 MB. For compiled languages (Go, Rust), a multi-stage build with
`FROM scratch` or `distroless` gives images of just a few MB.
</details>

## 🟡 Exercise 4 · CPU limits
Run `docker run --rm --cpus 0.2 busybox:1.36 sh -c 'while true; do :; done'` and watch `docker stats`.
How much CPU does it use? What limits it?

<details><summary>Show solution</summary>

About 20% of one core. The **CPU cgroup** (CFS quota) limits it. Kubernetes uses the same mechanism for
`resources.limits.cpu`. Unlike memory, going over the CPU limit **does not kill** the process — it gets
*throttled* (slowed down).
</details>

## 🔴 Challenge · A home-made orchestrator
Write a bash script that always keeps 3 `kc/hello:1.0` containers running: if you delete one with
`docker rm -f`, the script must recreate it within 5 seconds. Then answer: what is your script missing
to be like Kubernetes?

<details><summary>Show solution</summary>

```bash
#!/usr/bin/env bash
DESIRED=3
while true; do
  ACTUAL=$(docker ps -q --filter label=app=hello | wc -l | tr -d ' ')
  for ((i=ACTUAL; i<DESIRED; i++)); do
    docker run -d --label app=hello kc/hello:1.0 >/dev/null && echo "➕ recreated"
  done
  sleep 2
done
```
You just wrote a **reconciliation loop**. It is missing: many servers, scheduling, networking and DNS
between containers, zero-downtime updates, storage, security, an API, high availability for the
"orchestrator" itself… in other words, **Kubernetes**.
</details>

---

## ❓ Quiz

1. What do all containers on the same host share?
   a) The IP address b) The root user c) **The kernel** d) The filesystem

2. Which Linux feature limits a container's memory?
   a) chroot b) **cgroups** c) iptables d) namespaces

3. Which runtime does modern Kubernetes **not** use directly?
   a) **Docker Engine (dockershim)** b) CRI-O c) containerd d) None of them

4. What is the safest way to reference an image in production?
   a) No tag b) **`@sha256:<digest>`** c) `:stable` d) `:latest`

5. "I always want 3 replicas" is an example of:
   a) A cron job b) **Declarative desired state** c) Imperative programming d) A deploy script

<details><summary>Show answers</summary>

1-c · 2-b · 3-a · 4-b · 5-b
</details>

➡️ Next module: [02 · Linux, Networking and YAML](../02-linux-networking-yaml/)
