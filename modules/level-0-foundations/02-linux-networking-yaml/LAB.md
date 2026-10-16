# 🧪 Lab 02 · Networking and YAML Toolkit

| Goal | Duration | Environment |
|---|---|---|
| Practice network debugging and YAML with containers | 60 min | Docker + `yq` |

## Step 1 · A private network with two containers

```bash
docker network create kc-net
docker run -d --name web --network kc-net nginx:1.27
docker run -d --name toolbox --network kc-net nicolaka/netshoot sleep infinity
```

`netshoot` is an image full of network tools. You will use it again in Kubernetes.

✅ **Check:** `docker ps --filter network=kc-net` shows both containers.

## Step 2 · DNS between containers

```bash
docker exec toolbox nslookup web
docker exec toolbox dig +short web
```

**Expected output:** an IP like `172.18.0.2`. Docker's built-in DNS resolves the container name, just
like CoreDNS resolves Service names in Kubernetes.

## Step 3 · Ports and HTTP

```bash
docker exec toolbox nc -zv web 80        # is the port open?
docker exec toolbox curl -sI http://web  # headers only
docker exec web sh -c 'cat /proc/net/tcp | head -3'   # listening sockets (hex)
```

✅ **Check:** `nc` says `succeeded` / `open`, and `curl` shows `HTTP/1.1 200 OK`.

## Step 4 · Signals and exit codes

```bash
docker run -d --name sig busybox:1.36 sleep 3600
docker stop -t 5 sig                      # sends SIGTERM, then SIGKILL after 5 s
docker inspect sig --format 'ExitCode={{.State.ExitCode}}'
```

**Question:** the exit code is `137`. Why not `143`?

<details><summary>Show explanation</summary>

`sleep` running as PID 1 in a container ignores `SIGTERM` (PID 1 has no default signal handlers), so
Docker waited 5 s and sent `SIGKILL` → `128 + 9 = 137`. A well-written app handles `SIGTERM` and exits
with `143` or `0` quickly. This is exactly what happens on every Kubernetes rolling update.
</details>

## Step 5 · Write your first manifest by hand

```bash
mkdir -p ~/kc/lab02 && cd ~/kc/lab02
```

Create `pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hello
  labels:
    app: hello
    tier: frontend
spec:
  containers:
    - name: web
      image: nginx:1.27
      ports:
        - containerPort: 80
```

```bash
yq '.kind, .metadata.name, .spec.containers[0].image' pod.yaml
```

✅ **Check:** the output is `Pod`, `hello`, `nginx:1.27`.

## 💥 Step 6 · Break it on purpose

Create `broken.yaml` with a tab and a missing space (copy it exactly, the `→` means press Tab):

```bash
printf 'apiVersion: v1\nkind: Pod\nmetadata:\n\tname: broken\nspec:\n  containers:\n    - name:web\n      image: nginx:1.27\n' > broken.yaml
yq '.' broken.yaml
```

Find and fix both errors until `yq` parses the file and `.spec.containers[0].name` returns `web`.

<details><summary>Show solution</summary>

1. Line 4 uses a tab before `name:` → replace it with two spaces.
2. `- name:web` has no space after the colon, so YAML reads the string `"name:web"` instead of a map → `- name: web`.

```bash
sed -i.bak $'s/\t/  /; s/name:web/name: web/' broken.yaml
yq '.spec.containers[0].name' broken.yaml   # web
```
</details>

## 🧹 Cleanup

```bash
docker rm -f web toolbox sig
docker network rm kc-net
```

➡️ Next: [📝 EXERCISES.md](EXERCISES.md)
