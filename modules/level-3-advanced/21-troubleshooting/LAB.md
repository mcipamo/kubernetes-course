# 🧪 Lab 21 · Incident Day

| Goal | Duration | Environment |
|---|---|---|
| Diagnose and fix six realistic incidents using the method, then write a postmortem | 180 min | `kc` kind cluster |

**Rules of the game:**
- Do **not** read the YAML of the scenarios before solving them. Apply them blind.
- For each incident, fill in the incident card below **before** looking at the hints.
- Time yourself. Target: under 15 minutes per incident.

```text
Incident #__  |  Start: __:__  |  Mitigated: __:__  |  Resolved: __:__
Scope:
Hypotheses (H1, H2, H3):
Evidence (command → finding):
Root cause:
Fix:
Prevention:
```

## Setup: deploy the "production" environment

```bash
kubectl create namespace lab-21
kubectl config set-context --current --namespace=lab-21
kubectl apply -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata: { name: frontend }
spec:
  replicas: 2
  selector: { matchLabels: { app: frontend } }
  template:
    metadata: { labels: { app: frontend } }
    spec:
      containers:
        - name: web
          image: ghcr.io/stefanprodan/podinfo:6.7.0
          ports: [{ name: http, containerPort: 9898 }]
          readinessProbe: { httpGet: { path: /readyz, port: http }, periodSeconds: 3 }
---
apiVersion: v1
kind: Service
metadata: { name: frontend }
spec:
  selector: { app: frontend }
  ports: [{ port: 80, targetPort: http }]
EOF
kubectl rollout status deployment/frontend
```

---

## 🔥 Incident 1 · "The new release never came up"

```bash
kubectl apply -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata: { name: orders }
spec:
  replicas: 2
  selector: { matchLabels: { app: orders } }
  template:
    metadata: { labels: { app: orders } }
    spec:
      containers:
        - name: orders
          image: ghcr.io/stefanprodan/podinfo:6.7.0
          envFrom: [{ secretRef: { name: orders-db } }]
EOF
```

<details><summary>Hint</summary>The Pod status name is a strong clue. Then read the events.</details>
<details><summary>Show solution</summary>

`CreateContainerConfigError` → `describe pod` → `secret "orders-db" not found`.
Fix: `kubectl create secret generic orders-db --from-literal=DB_HOST=db` (or fix the reference).
Prevention: deploy Secrets before workloads (GitOps sync waves), and validate references in CI.
</details>

## 🔥 Incident 2 · "Payments keeps restarting"

```bash
kubectl apply -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata: { name: payments }
spec:
  replicas: 1
  selector: { matchLabels: { app: payments } }
  template:
    metadata: { labels: { app: payments } }
    spec:
      containers:
        - name: payments
          image: polinux/stress:1.0.4
          command: ["stress", "--vm", "1", "--vm-bytes", "180M", "--vm-hang", "0"]
          resources:
            requests: { memory: 64Mi }
            limits: { memory: 128Mi }
EOF
```

<details><summary>Hint</summary>`kubectl get pod -o jsonpath='{.status.containerStatuses[0].lastState}'`</details>
<details><summary>Show solution</summary>

`CrashLoopBackOff`; last state `OOMKilled`, exit code 137. The process needs ~180 MB with a 128 Mi
limit. Fix: raise the limit (e.g., 256Mi) based on measured usage, or fix the memory use.
Prevention: alert on OOMKilled; right-size with VPA recommendations.
</details>

## 🔥 Incident 3 · "The catalog page says 503"

```bash
kubectl apply -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata: { name: catalog }
spec:
  replicas: 2
  selector: { matchLabels: { app: catalog } }
  template:
    metadata: { labels: { app: catalog, version: v2 } }
    spec:
      containers:
        - name: catalog
          image: ghcr.io/stefanprodan/podinfo:6.7.0
          ports: [{ containerPort: 9898 }]
---
apiVersion: v1
kind: Service
metadata: { name: catalog }
spec:
  selector: { app: catalog, version: v1 }
  ports: [{ port: 80, targetPort: 9898 }]
EOF
kubectl run probe --image=curlimages/curl:8.10.1 --rm -it --restart=Never -- curl -s -m 3 catalog || true
```

<details><summary>Hint</summary>Pods are Running and Ready. Follow the Service checklist.</details>
<details><summary>Show solution</summary>

`kubectl get endpointslices -l kubernetes.io/service-name=catalog` → no endpoints. The Service selects
`version: v1`, but Pods are `version: v2`. Fix: select only `app: catalog` (or the right version).
Prevention: avoid version labels in Service selectors unless doing blue/green on purpose; alert on
Services with zero endpoints.
</details>

## 🔥 Incident 4 · "Search is stuck in Pending"

```bash
kubectl apply -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata: { name: search }
spec:
  replicas: 2
  selector: { matchLabels: { app: search } }
  template:
    metadata: { labels: { app: search } }
    spec:
      nodeSelector: { node.kubernetes.io/instance-type: m7i.4xlarge }
      containers:
        - name: search
          image: nginx:1.27
          resources: { requests: { cpu: 100m, memory: 64Mi } }
EOF
```

<details><summary>Hint</summary>Read the full `FailedScheduling` message.</details>
<details><summary>Show solution</summary>

`0/3 nodes are available: … didn't match Pod's node affinity/selector`. No node has that instance-type
label (it was copied from a cloud manifest). Fix: remove/adjust the `nodeSelector`.
Prevention: environment-specific overlays (Kustomize, module 23) and policy checks for node selectors.
</details>

## 🔥 Incident 5 · "Frontend can't reach anything since the security change"

```bash
kubectl apply -f - <<'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: lockdown }
spec:
  podSelector: {}
  policyTypes: [Egress]
  egress:
    - to:
        - podSelector: {}
EOF
kubectl exec deploy/frontend -- sh -c 'curl -s -m 3 catalog || echo FAIL'
kubectl exec deploy/frontend -- sh -c 'curl -s -m 3 http://$(getent hosts catalog | cut -d" " -f1 2>/dev/null) || nslookup catalog'
```

<details><summary>Hint</summary>Does the name resolve? Who answers DNS, and where does it live?</details>
<details><summary>Show solution</summary>

Egress is allowed only to Pods **in this namespace**, so DNS queries to CoreDNS in `kube-system` are
dropped: every name lookup fails (`curl: (6) Could not resolve host`). Connecting by Pod IP still works.
Fix: add the allow-DNS egress policy (module 18).
Prevention: ship default-deny together with allow-DNS; connectivity tests in CI.
</details>

## 🔥 Incident 6 · "A node went bad"

```bash
docker exec kc-worker2 systemctl stop kubelet
kubectl scale deployment frontend --replicas=6
sleep 60; kubectl get nodes; kubectl get pods -o wide -l app=frontend
```

<details><summary>Hint</summary>Node conditions, then the kubelet on that node.</details>
<details><summary>Show solution</summary>

`kc-worker2` turns `NotReady` (`Kubelet stopped posting node status`). Existing Pods there show as
Running but are **unknown**; new Pods go to healthy nodes; after ~5 min, Pods on the node get evicted.
On the node: `docker exec kc-worker2 systemctl status kubelet` → inactive; `journalctl -u kubelet`.
Mitigation in production: `kubectl cordon` + `drain` and let capacity move. Fix:
```bash
docker exec kc-worker2 systemctl start kubelet
kubectl wait --for=condition=Ready node/kc-worker2 --timeout=120s
```
Prevention: node auto-repair (cloud node pools), alert on `NotReady` > 2 min, PDBs and spread so one
node never holds too many replicas.
</details>

---

## 📝 Final task · Write a postmortem

Pick the incident that would have hurt users the most and write a one-page postmortem:

```markdown
# Postmortem: <title>
**Date:** · **Duration:** · **Severity:** · **Author:**

## Impact
Who was affected, how, for how long (use SLI numbers).

## Timeline (UTC)
- hh:mm  Change deployed …
- hh:mm  Alert fired …
- hh:mm  Mitigated by …
- hh:mm  Resolved …

## Root cause
What happened and why (5 whys). No blame.

## What went well / what went wrong / where we got lucky

## Action items
| Action | Type (prevent/detect/mitigate) | Owner | Due |
```

## 🧹 Cleanup

```bash
docker exec kc-worker2 systemctl start kubelet 2>/dev/null
kubectl config set-context --current --namespace=default
kubectl delete namespace lab-21
```

➡️ Next: [📝 EXERCISES.md](EXERCISES.md)
