# 📝 Exercises 21 · Systematic Troubleshooting

## 🟢 Exercise 1 · First command
What is your **first** command for each symptom? (a) `ImagePullBackOff`; (b) `CrashLoopBackOff`;
(c) `Pending`; (d) Service returns connection refused.

<details><summary>Show solution</summary>

(a) `kubectl describe pod` (events show the pull error) · (b) `kubectl logs <pod> --previous` ·
(c) `kubectl describe pod` (FailedScheduling) · (d) `kubectl get endpointslices -l kubernetes.io/service-name=<svc>` and compare `targetPort` with the container port
</details>

## 🟢 Exercise 2 · Exit codes
Interpret: 0, 1, 126, 127, 137, 139, 143.

<details><summary>Show solution</summary>

0 success · 1 generic app error · 126 command not executable (permissions) · 127 command not found
(wrong `command`/missing binary) · 137 SIGKILL (often OOM) · 139 segfault (SIGSEGV) · 143 SIGTERM (graceful stop)
</details>

## 🟡 Exercise 3 · Debug a distroless crash
A distroless container crashes at start; you need to inspect its filesystem and try other commands.
How?

<details><summary>Show solution</summary>

Create a **copy** of the Pod with a different command and a debug container sharing processes:
```bash
kubectl debug <pod> -it --copy-to=<pod>-debug --container=<c> --image=busybox:1.36 --share-processes -- sh
```
Or keep the image and override only the command:
```bash
kubectl debug <pod> --copy-to=<pod>-debug --container=<c> -- sleep 3600
kubectl exec -it <pod>-debug -c <c> -- /busybox/sh   # if the image has a debug shell
```
</details>

## 🟡 Exercise 4 · Stuck namespace
A namespace has been `Terminating` for an hour. Describe how you investigate safely.

<details><summary>Show solution</summary>

```bash
kubectl get ns <ns> -o jsonpath='{.status.conditions}' | jq
kubectl api-resources --verbs=list --namespaced -o name | xargs -n1 kubectl get -n <ns> --ignore-not-found --show-kind
```
Conditions tell you whether some API is unavailable (e.g., a broken APIService/webhook) or which
resources still have **finalizers**. Fix the controller that owns the finalizer (or restore the
APIService). Removing finalizers by hand is a last resort: it can orphan external resources (cloud
disks, load balancers, DNS records).
</details>

## 🔴 Challenge · Intermittent 502s
Users see ~2% 502 errors, only during deploys, only on the busiest service. Build the hypothesis list
and the experiment for each.

<details><summary>Show solution</summary>

| Hypothesis | Experiment |
|---|---|
| Pods receive traffic before they are ready | Check readiness probe exists and reflects warm-up; correlate 502s with new Pod start times |
| Pods are killed while still in endpoints / LB targets | Add `preStop` sleep; compare 502 timing with Pod termination; check LB deregistration delay |
| App ignores SIGTERM, drops in-flight requests | Send SIGTERM in a test and watch in-flight requests; check `terminationGracePeriodSeconds` |
| Keep-alive connections to a removed Pod | Gateway/LB idle timeout vs app keep-alive timeout (app's must be longer) |
| Surge exhausts capacity | Watch CPU/throttling during rollout; `maxUnavailable: 0`, `maxSurge` tuning |

Test one at a time in staging with a load generator, measuring 502s per rollout.
</details>

---

## ❓ Quiz

1. During a customer-facing outage right after a deploy, the first action should usually be…

   - a) Restart the cluster
   - b) Read all logs
   - c) Roll back
   - d) Open a ticket

2. A Pod is `Running` and Ready, but the Service has no endpoints. Most likely cause?

   - a) Quota
   - b) Image pull
   - c) Selector/label mismatch
   - d) Node failure

3. Which command shows the logs of the container instance that crashed?

   - a) `kubectl events`
   - b) `kubectl logs -f`
   - c) `kubectl top`
   - d) `kubectl logs --previous`

4. Exit code 127 suggests…

   - a) OOM
   - b) Segfault
   - c) Command not found
   - d) Graceful stop

5. "Blameless" in a postmortem means…

   - a) We skip root cause
   - b) Nobody is responsible for action items
   - c) Only managers write it
   - d) We fix the system instead of blaming individuals

<details><summary>Show answers</summary>

1. **c)** Roll back
2. **c)** Selector/label mismatch
3. **d)** `kubectl logs --previous`
4. **c)** Command not found
5. **d)** We fix the system instead of blaming individuals
</details>

➡️ Next module: [22 · Cluster Administration](../22-cluster-administration/)
