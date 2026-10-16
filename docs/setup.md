# 🧪 Lab environment setup

You practice the whole course on your own machine. No cloud account needed.

## Tools

| Tool | What for | Required |
|---|---|---|
| Docker Desktop / Podman / OrbStack / Colima | Container engine that runs kind | ✅ |
| `kubectl` | The Kubernetes command-line client | ✅ |
| `kind` | Kubernetes clusters inside containers | ✅ |
| `helm` | Package manager (module 23 onwards) | Level 4 |
| `k9s` | Terminal UI to explore the cluster | Recommended |
| `jq` / `yq` | Process JSON and YAML | Recommended |
| `kubectx` / `kubens` | Switch context and namespace quickly | Optional |

## macOS

```bash
brew install kubectl kind helm k9s jq yq kubectx
# Container engine (pick one)
brew install --cask docker      # or: brew install orbstack  /  brew install colima && colima start --cpu 4 --memory 8
```

## Linux (Debian/Ubuntu)

```bash
# Docker
curl -fsSL https://get.docker.com | sh && sudo usermod -aG docker $USER   # log out and back in

# kubectl (latest stable)
curl -LO "https://dl.k8s.io/release/$(curl -Ls https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -m 0755 kubectl /usr/local/bin/kubectl

# kind (check the latest release at https://github.com/kubernetes-sigs/kind/releases)
go install sigs.k8s.io/kind@latest   # or download the binary from the releases page
```

## Windows

Use **WSL2** with Ubuntu and follow the Linux steps. Docker Desktop must have WSL2 integration enabled.

## ✅ Check

```bash
docker version --format '{{.Server.Version}}'
kubectl version --client
kind version
```

## Alternatives to kind

| Tool | Strength | When to use it |
|---|---|---|
| **kind** (course standard) | Multi-node, fast, disposable | Always, unless told otherwise |
| minikube | Built-in addons (`minikube addons enable ingress`) | If you already know it |
| k3d | Very light (k3s) | Laptops with little RAM |
| Docker Desktop Kubernetes | One click | Quick single-node tests |

> ⚠️ The labs assume the `kc` cluster from [module 03](../modules/level-0-foundations/03-local-lab/LAB.md).
> With other tools, some steps (ports, StorageClass, node names) will differ.

## Autocompletion and aliases

```bash
# zsh
echo 'source <(kubectl completion zsh)' >> ~/.zshrc
echo 'alias k=kubectl' >> ~/.zshrc
echo 'compdef __start_kubectl k' >> ~/.zshrc
# bash
echo 'source <(kubectl completion bash)' >> ~/.bashrc
echo 'alias k=kubectl; complete -o default -F __start_kubectl k' >> ~/.bashrc
```

## 🩹 Troubleshooting your setup

| Symptom | Cause | Fix |
|---|---|---|
| `kubectl` warns about version skew, or a flag like `--subresource resize` is unknown | Your `kubectl` is more than one minor version away from the cluster | Install a `kubectl` within ±1 minor of the server (`kubectl version`) |
| `kind load docker-image` says `image … not present locally` (Podman) | It calls the `docker` binary, and Podman names images `localhost/<name>` | `podman save -o img.tar localhost/<name>:<tag>` then `kind load image-archive img.tar --name kc` |
| `docker exec` not found (Podman) | Podman is your container engine | Use `podman exec <node>` (or `alias docker=podman`) |
| `helm install oci://…` fails with `docker-credential-desktop: executable file not found` | `~/.docker/config.json` still points to Docker Desktop's credential helper | Remove the `"credsStore"` line from that file, or run with `DOCKER_CONFIG=$(mktemp -d)` |
