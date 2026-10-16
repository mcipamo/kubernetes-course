# 🧪 Lab 01 · Your First Container

| Goal | Duration | Environment |
|---|---|---|
| Build, run and inspect your own image | 45 min | Docker (or Podman with `alias docker=podman`) |

## Step 1 · Check your container engine

```bash
docker version --format 'Client {{.Client.Version}} · Server {{.Server.Version}}'
```

**Expected output:** two versions. If you see `Cannot connect to the Docker daemon`, start Docker Desktop.

✅ **Check:** `docker run --rm hello-world` prints *Hello from Docker!*

## Step 2 · Write a tiny app

```bash
mkdir -p ~/kc/lab01 && cd ~/kc/lab01

cat > app.py <<'EOF'
from http.server import BaseHTTPRequestHandler, HTTPServer
import os, socket

class H(BaseHTTPRequestHandler):
    def do_GET(self):
        msg = f"Hello from {socket.gethostname()} · version {os.getenv('VERSION','1.0')}\n"
        self.send_response(200); self.end_headers(); self.wfile.write(msg.encode())

HTTPServer(("", 8000), H).serve_forever()
EOF
```

## Step 3 · Write the Dockerfile

```bash
cat > Dockerfile <<'EOF'
# Base image: official Python, slim variant
FROM python:3.12-slim
# Working directory inside the image
WORKDIR /app
# Copy the code
COPY app.py .
# Default environment variable
ENV VERSION=1.0
# Documents the port (it does not open it by itself)
EXPOSE 8000
# Run as an unprivileged user (good practice)
USER 1000
# Main process (PID 1)
CMD ["python", "app.py"]
EOF
```

## Step 4 · Build the image

```bash
docker build -t kc/hello:1.0 .
docker images kc/hello
```

✅ **Check:** `kc/hello` appears with tag `1.0` and a size around 130–150 MB.

## Step 5 · Run two containers from the same image

```bash
docker run -d --name hello-a -p 8001:8000 kc/hello:1.0
docker run -d --name hello-b -p 8002:8000 -e VERSION=2.0 kc/hello:1.0
curl localhost:8001
curl localhost:8002
```

**Expected output:**
```text
Hello from 3f2a1b9c0d1e · version 1.0
Hello from 9e8d7c6b5a4f · version 2.0
```

✅ **Check:** each answer has a different *hostname*. They are two isolated processes from one image.

## Step 6 · Look inside

```bash
docker exec hello-a ls /proc | grep -E '^[0-9]+$'   # the container only sees its own processes
docker exec hello-a id                              # runs as UID 1000
docker stats --no-stream hello-a hello-b            # CPU/memory usage
```

## 💥 Step 7 · Break it on purpose

Run the container with an absurdly low memory limit:

```bash
docker run -d --name hello-oom -m 6m kc/hello:1.0
sleep 3; docker ps -a --filter name=hello-oom
docker inspect hello-oom --format 'OOMKilled={{.State.OOMKilled}} ExitCode={{.State.ExitCode}}'
```

**Diagnosis question:** why did it die? Which kernel feature killed it?

<details><summary>Show explanation</summary>

The memory **cgroup** capped the container at 6 MB. Python needs more, so the kernel called the *OOM
killer* (`OOMKilled=true`, `ExitCode=137` = 128 + signal 9). In Kubernetes you will see exactly the same
thing as the `OOMKilled` state (module 10).
</details>

## Step 8 · Immutability

```bash
docker exec hello-a sh -c 'echo change > /tmp/note.txt && cat /tmp/note.txt'
docker rm -f hello-a
docker run -d --name hello-a -p 8001:8000 kc/hello:1.0
docker exec hello-a cat /tmp/note.txt
```

✅ **Check:** the last command fails with `No such file`. The change died with the container.

## 🧹 Cleanup

```bash
docker rm -f hello-a hello-b hello-oom
```

➡️ Next: [📝 EXERCISES.md](EXERCISES.md)
