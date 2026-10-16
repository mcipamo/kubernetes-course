# ⌨️ kubectl cheatsheet

> Golden rule: **`get` to see, `describe` to understand, `logs` to listen, `events` for the story.**

## Context and configuration

| Command | What it does |
|---|---|
| `kubectl config get-contexts` | List available clusters/contexts |
| `kubectl config use-context kind-kc` | Switch to the course cluster |
| `kubectl config set-context --current --namespace=lab-05` | Set the default namespace |
| `kubectl cluster-info` | Show the API server URL |
| `kubectl api-resources` | List every resource type in the cluster |
| `kubectl explain deployment.spec.strategy` | Field docs without leaving the terminal |

## Look

```bash
kubectl get pods                         # Pods in the current namespace
kubectl get pods -A                      # in all namespaces
kubectl get pods -o wide                 # + IP and node
kubectl get pods -l app=web              # filter by label
kubectl get pods --field-selector=status.phase!=Running
kubectl get deploy,svc,pods              # several types at once
kubectl get pod web -o yaml              # the full object
kubectl get pods -o jsonpath='{.items[*].metadata.name}'
kubectl get pods -w                      # watch changes live
kubectl get events --sort-by=.lastTimestamp
kubectl events --for pod/web             # events for one object
```

## Create and change

```bash
kubectl apply -f file.yaml               # declarative (recommended)
kubectl apply -k ./overlay               # with Kustomize
kubectl diff -f file.yaml                # what would change?
kubectl create deployment web --image=nginx --replicas=3
kubectl run tmp --image=busybox:1.36 --rm -it --restart=Never -- sh
kubectl expose deployment web --port=80 --target-port=80
kubectl scale deployment web --replicas=5
kubectl set image deployment/web nginx=nginx:1.27
kubectl edit deployment web
kubectl label pod web tier=frontend
kubectl annotate pod web owner=team-a
kubectl delete -f file.yaml
```

## Generate YAML without typing it (exam trick)

```bash
kubectl create deployment web --image=nginx --dry-run=client -o yaml > deploy.yaml
kubectl run web --image=nginx --dry-run=client -o yaml > pod.yaml
kubectl create service clusterip web --tcp=80:80 --dry-run=client -o yaml
kubectl create configmap app --from-literal=MODE=prod --dry-run=client -o yaml
kubectl create secret generic db --from-literal=pass=s3cr3t --dry-run=client -o yaml
kubectl create job once --image=busybox:1.36 --dry-run=client -o yaml -- echo hello
kubectl create cronjob tick --image=busybox:1.36 --schedule="*/5 * * * *" --dry-run=client -o yaml -- date
```

> Flags go **before** `--`. Everything after `--` is the container command.

## Debug

```bash
kubectl describe pod web                 # state + events (always start here)
kubectl logs web                         # container logs
kubectl logs web -c sidecar              # from a specific container
kubectl logs web --previous              # from the container that restarted
kubectl logs -l app=web --all-containers --tail=50 -f
kubectl exec -it web -- sh               # shell into the container
kubectl debug -it web --image=busybox:1.36 --target=web   # ephemeral container
kubectl debug node/kc-worker -it --image=busybox:1.36    # shell on a node
kubectl port-forward svc/web 8080:80     # reach it from your laptop
kubectl top pods; kubectl top nodes      # needs metrics-server
kubectl auth can-i create pods --as=system:serviceaccount:lab:app
```

## Rollouts

```bash
kubectl rollout status deployment/web
kubectl rollout history deployment/web
kubectl rollout undo deployment/web --to-revision=2
kubectl rollout restart deployment/web
kubectl rollout pause deployment/web; kubectl rollout resume deployment/web
```

## Nodes

```bash
kubectl cordon kc-worker                # stop scheduling Pods here
kubectl drain kc-worker --ignore-daemonsets --delete-emptydir-data
kubectl uncordon kc-worker
kubectl taint nodes kc-worker dedicated=gpu:NoSchedule
kubectl taint nodes kc-worker dedicated=gpu:NoSchedule-   # remove the taint
```

## Pod states and what they mean

| State | Meaning | First command |
|---|---|---|
| `Pending` | Could not be placed on a node | `kubectl describe pod` → Events |
| `ContainerCreating` | Placed; preparing volumes/image | `kubectl describe pod` |
| `ImagePullBackOff` | Cannot pull the image | Check name, tag and credentials |
| `CrashLoopBackOff` | Starts and dies over and over | `kubectl logs --previous` |
| `OOMKilled` | Exceeded its memory limit | Check `resources.limits.memory` |
| `Running` but `0/1 Ready` | The readiness probe fails | `kubectl describe pod` → probes |
| Stuck in `Terminating` | Finalizers or a lost node | Check `metadata.finalizers` |
