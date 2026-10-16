# 📝 Exercises 03 · Your Local Lab

## 🟢 Exercise 1 · A second cluster
Create a one-node cluster called `scratch`, switch between `kind-kc` and `kind-scratch`, and delete `scratch`.

<details><summary>Show solution</summary>

```bash
kind create cluster --name scratch
kubectl config get-contexts
kubectl config use-context kind-kc
kubectl get nodes          # 3 nodes
kubectl config use-context kind-scratch
kubectl get nodes          # 1 node
kind delete cluster --name scratch
kubectl config use-context kind-kc
```
</details>

## 🟢 Exercise 2 · Default namespace
Make `lab-03` the default namespace of your current context, then go back to `default`.

<details><summary>Show solution</summary>

```bash
kubectl config set-context --current --namespace=lab-03
kubectl config view --minify -o jsonpath='{..namespace}'; echo
kubectl config set-context --current --namespace=default
```
</details>

## 🟡 Exercise 3 · Inside a node
A kind node is a container. Open a shell in `kc-worker2` and list the running containers with `crictl`.

<details><summary>Show solution</summary>

```bash
docker exec -it kc-worker2 bash
crictl ps              # containers managed by containerd on this node
crictl images | head   # images pulled on this node
exit
```
`crictl` talks to the container runtime through the CRI — the same interface the kubelet uses.
</details>

## 🟡 Exercise 4 · Which API resources exist?
How many resource types does your cluster serve? Which ones belong to the `apps` group?

<details><summary>Show solution</summary>

```bash
kubectl api-resources | wc -l
kubectl api-resources --api-group=apps
# controllerrevisions, daemonsets, deployments, replicasets, statefulsets
```
</details>

## 🔴 Challenge · Match production
Your company runs Kubernetes v1.36 in production. Create a cluster `kc-136` that runs exactly that
minor version, and prove it.

<details><summary>Show solution</summary>

Find the matching `kindest/node` image for your kind version in the kind release notes
(https://github.com/kubernetes-sigs/kind/releases), then:
```bash
kind create cluster --name kc-136 --image kindest/node:v1.36.<patch>
kubectl --context kind-kc-136 version
kubectl --context kind-kc-136 get nodes -o jsonpath='{.items[0].status.nodeInfo.kubeletVersion}'
kind delete cluster --name kc-136
```
⚠️ Each kind release supports a specific set of node images — always use the digest listed in its release notes.
</details>

---

## ❓ Quiz

1. What is a kind "node" physically?
   a) A process on your laptop b) A cloud instance c) **A Docker container** d) A VM

2. Which kubeconfig element combines a cluster, a user and a namespace?
   a) cluster b) **context** c) preferences d) user

3. Why did the bare Pod not come back after the node failed?
   a) **No controller owns it** b) It was too big c) kind does not support Pods d) The image was missing

4. How do you make an image built on your laptop available to kind nodes?
   a) `docker push` b) **`kind load docker-image`** c) It works automatically d) `kubectl cp`

5. What should you always check before a destructive `kubectl` command?
   a) Node count b) **The current context** c) The image tag d) Docker version

<details><summary>Show answers</summary>

1-c · 2-b · 3-a · 4-b · 5-b
</details>

➡️ Next module: [04 · Kubernetes Architecture](../../level-1-basics/04-architecture/)
