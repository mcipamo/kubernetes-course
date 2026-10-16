# 🧪 Lab 13 · The Right Controller for the Job

| Goal | Duration | Environment |
|---|---|---|
| Run a StatefulSet, a DaemonSet, an indexed Job and a CronJob | 90 min | `kc` kind cluster |

## Setup

```bash
kubectl create namespace lab-13
kubectl config set-context --current --namespace=lab-13
mkdir -p ~/kc/lab13 && cd ~/kc/lab13
```

## Step 1 · A StatefulSet with stable identity

Create `kv.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kv
spec:
  clusterIP: None                  # headless
  selector: { app: kv }
  ports:
    - { name: redis, port: 6379 }
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: kv
spec:
  serviceName: kv
  replicas: 3
  selector:
    matchLabels: { app: kv }
  template:
    metadata:
      labels: { app: kv }
    spec:
      containers:
        - name: redis
          image: redis:7.4
          args: ["--appendonly", "yes"]
          ports:
            - { name: redis, containerPort: 6379 }
          readinessProbe:
            exec: { command: ["redis-cli", "ping"] }
            periodSeconds: 3
          volumeMounts:
            - { name: data, mountPath: /data }
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: [ReadWriteOnce]
        resources:
          requests:
            storage: 256Mi
```

```bash
kubectl apply -f kv.yaml
kubectl get pods -l app=kv -w     # kv-0 → Ready, then kv-1, then kv-2 (ordered); Ctrl+C
kubectl get pvc
```

✅ **Check:** Pods `kv-0`, `kv-1`, `kv-2` and PVCs `data-kv-0`, `data-kv-1`, `data-kv-2`.

## Step 2 · Stable DNS and stable storage

```bash
kubectl run net --image=nicolaka/netshoot --restart=Never -- sleep infinity
kubectl wait --for=condition=Ready pod/net
kubectl exec net -- dig +short kv-1.kv.lab-13.svc.cluster.local
kubectl exec kv-1 -- redis-cli set owner "I am kv-1"
kubectl delete pod kv-1
kubectl wait --for=condition=Ready pod/kv-1 --timeout=90s
kubectl exec kv-1 -- redis-cli get owner
```

✅ **Check:** the recreated `kv-1` has the **same name**, the **same PVC** and still returns `I am kv-1`.
(Its IP may change — that is why you use the DNS name.)

## Step 3 · Partitioned rolling update (canary for StatefulSets)

```bash
kubectl patch sts kv -p '{"spec":{"updateStrategy":{"type":"RollingUpdate","rollingUpdate":{"partition":2}}}}'
kubectl set image sts/kv redis=redis:7.4-alpine
sleep 30
kubectl get pods -l app=kv -o custom-columns=NAME:.metadata.name,IMAGE:.spec.containers[0].image
```

✅ **Check:** only `kv-2` runs `7.4-alpine`. Set `partition: 0` to finish the rollout.

```bash
kubectl patch sts kv -p '{"spec":{"updateStrategy":{"rollingUpdate":{"partition":0}}}}'
kubectl rollout status sts/kv
```

## Step 4 · A DaemonSet node agent

Create `agent.yaml`:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-agent
spec:
  selector:
    matchLabels: { app: node-agent }
  template:
    metadata:
      labels: { app: node-agent }
    spec:
      tolerations:
        - key: node-role.kubernetes.io/control-plane
          operator: Exists
          effect: NoSchedule
      containers:
        - name: agent
          image: busybox:1.36
          command: ["sh", "-c", "while true; do echo \"$(date) agent on $NODE_NAME\"; sleep 30; done"]
          env:
            - name: NODE_NAME
              valueFrom: { fieldRef: { fieldPath: spec.nodeName } }
          resources:
            requests: { cpu: 10m, memory: 16Mi }
            limits: { memory: 32Mi }
```

```bash
kubectl apply -f agent.yaml
kubectl get ds node-agent
kubectl get pods -l app=node-agent -o wide
```

✅ **Check:** `DESIRED = CURRENT = READY = 3` — one Pod per node, including the control plane (thanks to the toleration).

## Step 5 · An indexed parallel Job

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  name: chunks
spec:
  completions: 6
  parallelism: 2
  completionMode: Indexed
  ttlSecondsAfterFinished: 600
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: worker
          image: busybox:1.36
          command: ["sh", "-c", "echo processing chunk $JOB_COMPLETION_INDEX; sleep 5"]
EOF
kubectl get pods -l job-name=chunks -w     # 2 at a time; Ctrl+C when 6 are Completed
kubectl get job chunks
kubectl logs -l job-name=chunks --prefix | sort
```

✅ **Check:** `COMPLETIONS 6/6`, and the logs show chunks 0 through 5.

## Step 6 · A CronJob, run on demand

```bash
kubectl create cronjob tick --image=busybox:1.36 --schedule="*/1 * * * *" -- sh -c 'date; echo tick'
kubectl create job tick-now --from=cronjob/tick
kubectl wait --for=condition=complete job/tick-now --timeout=60s && kubectl logs job/tick-now
sleep 70; kubectl get jobs
```

✅ **Check:** besides `tick-now`, a Job named `tick-<timestamp>` was created by the schedule.

## 💥 Step 7 · Break it on purpose: a Job that never gives up

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  name: bad-migration
spec:
  backoffLimit: 6
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migrate
          image: busybox:1.36
          command: ["sh", "-c", "echo 'ERROR: schema version 12 incompatible'; exit 42"]
EOF
sleep 60; kubectl get pods -l job-name=bad-migration
```

**Diagnosis:** exit code 42 means "permanent error, retrying is useless". How many Pods were wasted?
Rewrite the Job so it fails immediately on exit code 42.

<details><summary>Show solution</summary>

The Job keeps creating Pods with exponential back-off until `backoffLimit: 6` is reached (up to 7 Pods
and several minutes). Add a `podFailurePolicy`:
```yaml
spec:
  backoffLimit: 6
  podFailurePolicy:
    rules:
      - action: FailJob
        onExitCodes:
          containerName: migrate
          operator: In
          values: [42]
```
```bash
kubectl delete job bad-migration
# re-apply with the policy, then:
kubectl get job bad-migration -o jsonpath='{.status.conditions[*].reason}'; echo   # PodFailurePolicy
```
Only **one** Pod runs, and the Job fails at once with reason `PodFailurePolicy`.
</details>

## 🧹 Cleanup

```bash
kubectl config set-context --current --namespace=default
kubectl delete namespace lab-13       # also deletes the StatefulSet PVCs in this namespace
```

➡️ Next: [📝 EXERCISES.md](EXERCISES.md)
