# 📝 Exercises 11 · Probes and Pod Lifecycle

## 🟢 Exercise 1 · Which probe?
Pick the probe for each need: (a) don't send traffic while the cache warms up; (b) restart if the
event loop deadlocks; (c) a legacy app takes 3 minutes to boot.

<details><summary>Show solution</summary>

(a) **readinessProbe** · (b) **livenessProbe** · (c) **startupProbe** with `failureThreshold × periodSeconds ≥ 180 s`
</details>

## 🟢 Exercise 2 · Time to restart
`livenessProbe: periodSeconds: 10, timeoutSeconds: 2, failureThreshold: 3`. The app hangs. Roughly
how long until the container is restarted?

<details><summary>Show solution</summary>

About **30 seconds** (3 consecutive failures, one every 10 s; each attempt may take up to 2 s).
</details>

## 🟡 Exercise 3 · A TCP and a gRPC probe
Write a readiness probe for PostgreSQL on 5432 and a liveness probe for a gRPC server on 9090.

<details><summary>Show solution</summary>

```yaml
readinessProbe:
  tcpSocket: { port: 5432 }
  periodSeconds: 5
livenessProbe:
  grpc: { port: 9090 }
  periodSeconds: 10
```
The gRPC server must implement the standard `grpc.health.v1.Health` service.
</details>

## 🟡 Exercise 4 · Shell as PID 1
An image's entrypoint is `sh -c "python app.py"`. Pods always take 30 s to terminate. Why, and how do you fix it?

<details><summary>Show solution</summary>

`sh` is PID 1 and does not forward `SIGTERM` to `python`, so the app never hears it and gets `SIGKILL`
after the grace period. Fix: `exec` the process (`sh -c "exec python app.py"`), use the exec form
`CMD ["python","app.py"]`, or use an init like `tini`. Then handle `SIGTERM` in the app.
</details>

## 🔴 Challenge · Long-running requests
Your API serves report downloads that can take up to 2 minutes. Design the termination settings so
deploys never cut a download.

<details><summary>Show solution</summary>

```yaml
spec:
  terminationGracePeriodSeconds: 150      # > longest request + preStop
  containers:
    - name: api
      lifecycle:
        preStop:
          sleep: { seconds: 10 }          # let endpoints/LBs stop sending new requests
      readinessProbe:
        httpGet: { path: /readyz, port: http }
        periodSeconds: 2
```
And in the app: on `SIGTERM`, fail `/readyz`, stop accepting new connections, wait for in-flight
downloads, then exit. Use `maxUnavailable: 0` and a PodDisruptionBudget (module 28) so drains and
rollouts keep enough capacity.
</details>

---

## ❓ Quiz

1. A failing readiness probe causes…

   - a) Pod deletion
   - b) Eviction
   - c) A restart
   - d) Removal from Service endpoints

2. A failing liveness probe causes…

   - a) Removal from endpoints only
   - b) A container restart
   - c) Nothing
   - d) Node drain

3. While the startup probe has not succeeded, liveness and readiness probes are…

   - a) Running normally
   - b) Doubled
   - c) Ignored after 10 s
   - d) Disabled

4. Default `terminationGracePeriodSeconds` is:

   - a) 30
   - b) 60
   - c) 10
   - d) 300

5. Which is a dangerous liveness probe?

   - a) A gRPC health check
   - b) `GET /livez` on localhost
   - c) A TCP check on the app port
   - d) A check that the database is reachable

<details><summary>Show answers</summary>

1. **d)** Removal from Service endpoints
2. **b)** A container restart
3. **d)** Disabled
4. **a)** 30
5. **d)** A check that the database is reachable
</details>

➡️ Next module: [12 · Storage](../12-storage/)
