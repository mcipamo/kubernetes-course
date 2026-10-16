# 🧪 Lab 23 · Package It Once, Deploy It Everywhere

| Goal | Duration | Environment |
|---|---|---|
| Build a base + dev/prod overlays with Kustomize, then write and ship a Helm chart | 120 min | `kc` kind cluster + `helm` |

## Part A — Kustomize

### Step 1 · The base

```bash
mkdir -p ~/kc/lab23/shop/{base,overlays/dev,overlays/prod} && cd ~/kc/lab23/shop
```

Create `base/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shop
spec:
  replicas: 1
  selector:
    matchLabels: { app: shop }
  template:
    metadata:
      labels: { app: shop }
    spec:
      containers:
        - name: app
          image: ghcr.io/stefanprodan/podinfo:6.6.0
          ports:
            - { name: http, containerPort: 9898 }
          envFrom:
            - configMapRef: { name: shop-config }
          resources:
            requests: { cpu: 20m, memory: 32Mi }
```

Create `base/service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: shop
spec:
  selector: { app: shop }
  ports:
    - { port: 80, targetPort: http }
```

Create `base/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
configMapGenerator:
  - name: shop-config
    literals:
      - PODINFO_UI_MESSAGE=base
```

```bash
kubectl kustomize base | grep -E 'name: shop-config|kind:'
```

✅ **Check:** the ConfigMap is called `shop-config-<hash>`, and the Deployment's `configMapRef` was rewritten to that hashed name.

### Step 2 · Two overlays

Create `overlays/dev/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: lab-23-dev
namePrefix: dev-
resources:
  - ../../base
configMapGenerator:
  - name: shop-config
    behavior: merge
    literals:
      - PODINFO_UI_MESSAGE=hello from DEV
```

Create `overlays/prod/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: lab-23-prod
resources:
  - ../../base
images:
  - name: ghcr.io/stefanprodan/podinfo
    newTag: 6.7.0
replicas:
  - name: shop
    count: 3
configMapGenerator:
  - name: shop-config
    behavior: merge
    literals:
      - PODINFO_UI_MESSAGE=hello from PROD
patches:
  - target: { kind: Deployment, name: shop }
    patch: |-
      - op: replace
        path: /spec/template/spec/containers/0/resources
        value:
          requests: { cpu: 100m, memory: 64Mi }
          limits:   { memory: 128Mi }
```

```bash
kubectl create namespace lab-23-dev; kubectl create namespace lab-23-prod
diff <(kubectl kustomize overlays/dev) <(kubectl kustomize overlays/prod)
kubectl apply -k overlays/dev
kubectl apply -k overlays/prod
kubectl get deploy -A -l app=shop -o custom-columns=NS:.metadata.namespace,NAME:.metadata.name,REPLICAS:.spec.replicas,IMAGE:.spec.template.spec.containers[0].image
```

✅ **Check:** `dev-shop` (1 replica, 6.6.0) in `lab-23-dev`; `shop` (3 replicas, 6.7.0) in `lab-23-prod`.

### Step 3 · Config change → automatic rollout

```bash
sed -i.bak 's/hello from PROD/hello from PROD v2/' overlays/prod/kustomization.yaml
kubectl diff -k overlays/prod | grep -E '^[+-] .*shop-config' | head -4
kubectl apply -k overlays/prod
kubectl -n lab-23-prod rollout status deployment/shop
```

✅ **Check:** a **new** ConfigMap name was created and the Deployment rolled out by itself.

## Part B — Helm

### Step 4 · Scaffold a chart

```bash
cd ~/kc/lab23 && helm create podinfo-chart
ls podinfo-chart podinfo-chart/templates
```

Edit `podinfo-chart/values.yaml` — change these keys:

```yaml
image:
  repository: ghcr.io/stefanprodan/podinfo
  tag: "6.7.0"
service:
  port: 9898
```

and in `podinfo-chart/templates/deployment.yaml` make sure the container port is `9898`
(`containerPort: {{ .Values.service.port }}`), then:

```bash
helm lint podinfo-chart
helm template demo podinfo-chart | kubectl apply --dry-run=server -f - >/dev/null && echo "renders and validates"
```

### Step 5 · Install, upgrade, roll back

```bash
helm install demo ./podinfo-chart -n lab-23-helm --create-namespace --set replicaCount=2 --wait
helm list -n lab-23-helm
helm upgrade demo ./podinfo-chart -n lab-23-helm --set replicaCount=4 --wait
helm history demo -n lab-23-helm
helm rollback demo 1 -n lab-23-helm --wait
kubectl -n lab-23-helm get deploy
kubectl -n lab-23-helm get secrets -l owner=helm
```

✅ **Check:** history shows revisions 1, 2, 3 (the rollback is a new revision), the Deployment is back
to 2 replicas, and each revision is stored as a Secret `sh.helm.release.v1.demo.vN`.

## 💥 Step 6 · Break it on purpose: a bad upgrade

```bash
helm upgrade demo ./podinfo-chart -n lab-23-helm --set image.tag=does-not-exist --wait --timeout 60s; echo "exit=$?"
helm history demo -n lab-23-helm
kubectl -n lab-23-helm get pods
```

**Diagnosis:** what state is the release in? Are users affected? How do you recover, and how would
`--atomic` have changed the story?

<details><summary>Show solution</summary>

The release revision is marked **`failed`**; the new Pods are in `ImagePullBackOff`, but thanks to the
rolling update the old Pods keep serving. Recover with:
```bash
helm rollback demo -n lab-23-helm --wait
```
With `--atomic` (`helm upgrade --atomic …`), Helm would have **rolled back automatically** when the
wait timed out, leaving the release `deployed` on the previous good revision. In CI/CD, always use
`--atomic` (or GitOps health checks, module 24).
</details>

## 🧹 Cleanup

```bash
helm uninstall demo -n lab-23-helm
kubectl delete namespace lab-23-dev lab-23-prod lab-23-helm
```

➡️ Next: [📝 EXERCISES.md](EXERCISES.md)
