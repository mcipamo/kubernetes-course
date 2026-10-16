# 📝 Exercises 20 · Observability

## 🟢 Exercise 1 · Pick the signal
Which signal answers each question? (a) Why was this Pod evicted? (b) Which service adds 800 ms to
checkout? (c) Is the error rate higher than last week? (d) What did the payment service log for order 123?

<details><summary>Show solution</summary>

(a) Events · (b) Traces · (c) Metrics · (d) Logs
</details>

## 🟢 Exercise 2 · metrics-server vs Prometheus
Name two things metrics-server cannot do that Prometheus can.

<details><summary>Show solution</summary>

metrics-server keeps only the **latest** CPU/memory sample (no history, no trends) and only
**resource** metrics (no app metrics, no object state). It also cannot alert or power dashboards.
</details>

## 🟡 Exercise 3 · PromQL: top memory namespaces
Write a query for the top 5 namespaces by memory working set.

<details><summary>Show solution</summary>

```promql
topk(5, sum by (namespace) (container_memory_working_set_bytes{container!=""}))
```
</details>

## 🟡 Exercise 4 · An alert rule
Write a PrometheusRule that fires when any container restarts more than 3 times in 15 minutes, for 5
minutes, with a runbook link.

<details><summary>Show solution</summary>

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: restarts
  namespace: monitoring
spec:
  groups:
    - name: workloads
      rules:
        - alert: ContainerRestartingOften
          expr: increase(kube_pod_container_status_restarts_total[15m]) > 3
          for: 5m
          labels:
            severity: ticket
          annotations:
            summary: "{{ $labels.namespace }}/{{ $labels.pod }} ({{ $labels.container }}) restarts often"
            runbook_url: https://runbooks.example.com/ContainerRestartingOften
```
Note `severity: ticket`, not `page` — restarts are a cause; page on user-facing symptoms.
</details>

## 🔴 Challenge · Cardinality incident
Prometheus memory doubled overnight. A new deploy added a label `user_id` to `http_requests_total`.
Explain the impact and two ways to fix it without losing the metric.

<details><summary>Show solution</summary>

Each unique label combination is a new **time series**. With 1 M users × status codes × paths × Pods,
series count explodes, and so do memory, CPU and storage — possibly OOMing Prometheus and blinding
everyone.

Fixes:
1. **Remove the label at the source** (put user IDs in logs or traces, not metrics).
2. **Drop it at scrape time** with relabeling:
   ```yaml
   metricRelabelings:
     - action: labeldrop
       regex: user_id
   ```
3. Guard against recurrence: `sample_limit`/`label_limit` per target and a CI check on metric labels.
</details>

---

## ❓ Quiz

1. Which component exposes the *state* of Deployments, Pods and Jobs as metrics?

   - a) cAdvisor
   - b) kube-state-metrics
   - c) node-exporter
   - d) metrics-server

2. Where are container logs stored on a node by default?

   - a) `/var/log/pods/…`
   - b) etcd
   - c) The image
   - d) The API server

3. The RED method stands for…

   - a) Requests, Events, Data
   - b) Reliability, Efficiency, Durability
   - c) Rate, Errors, Duration
   - d) Restarts, Evictions, Deployments

4. Which should page an on-call engineer at 3 a.m.?

   - a) SLO burn rate threatening the error budget
   - b) Disk at 70%
   - c) A new image pushed
   - d) One Pod restarted

5. The CNCF standard for traces, metrics and logs instrumentation is…

   - a) Jaeger
   - b) Fluentd
   - c) OpenTelemetry
   - d) StatsD

<details><summary>Show answers</summary>

1. **b)** kube-state-metrics
2. **a)** `/var/log/pods/…`
3. **c)** Rate, Errors, Duration
4. **a)** SLO burn rate threatening the error budget
5. **c)** OpenTelemetry
</details>

➡️ Next module: [21 · Systematic Troubleshooting](../21-troubleshooting/)
