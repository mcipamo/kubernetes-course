# 📝 Exercises 28 · Reliability Engineering

## 🟢 Exercise 1 · Budget math
A service has a 99.9% availability SLO over 30 days and serves 50 million requests per month. How
many failed requests are allowed? A 20-minute full outage happened — how much budget is left?

<details><summary>Show solution</summary>

0.1% of 50 M = **50,000** failed requests (≈ 43.2 min of full outage). A 20-minute full outage ≈ 46% of
the budget, so **~54%** remains (assuming uniform traffic).
</details>

## 🟢 Exercise 2 · Good SLI?
Which are good SLIs? (a) Node CPU usage; (b) % of checkout requests answered with non-5xx in < 400 ms at
the gateway; (c) number of Pods running; (d) % of daily reports delivered before 07:00.

<details><summary>Show solution</summary>

(b) and (d) — they measure what users experience. (a) and (c) are useful **causes/diagnostics**, not SLIs.
</details>

## 🟡 Exercise 3 · Slow-burn alert
Add the second paging rule to the lab: 6 h and 30 min windows, burn rate 6, for a 99.5% SLO.

<details><summary>Show solution</summary>

```yaml
- record: sli:checkout_errors:ratio_rate30m
  expr: sum(rate(http_requests_total{namespace="lab-28",status=~"5.."}[30m])) / sum(rate(http_requests_total{namespace="lab-28"}[30m]))
- record: sli:checkout_errors:ratio_rate6h
  expr: sum(rate(http_requests_total{namespace="lab-28",status=~"5.."}[6h])) / sum(rate(http_requests_total{namespace="lab-28"}[6h]))
- alert: CheckoutErrorBudgetBurnSlow
  expr: |
    sli:checkout_errors:ratio_rate6h > (6 * 0.005)
    and
    sli:checkout_errors:ratio_rate30m > (6 * 0.005)
  for: 15m
  labels: { severity: page }
```
</details>

## 🟡 Exercise 4 · PDB review
Review these PDBs for a 3-replica Deployment: (a) `minAvailable: 3`; (b) `maxUnavailable: 1`;
(c) `minAvailable: 50%`; (d) none.

<details><summary>Show solution</summary>

(a) ❌ blocks every drain (0 disruptions allowed). (b) ✅ the usual choice. (c) ⚠️ 50% of 3 rounds up to 2 → 1 disruption: fine, but percentages are less obvious to read. (d) ⚠️ a drain or autoscaler can evict all 3 at once.
</details>

## 🔴 Challenge · Production Readiness Review
You are reviewing the launch of a new `payments` service on Kubernetes. Write the 15 questions
you will ask, grouped by area.

<details><summary>Show solution</summary>

**Ownership:** 1) Who is on call, and is there a runbook per alert? 2) What are the service's dependencies and their SLOs?
**SLOs:** 3) What are the SLIs/SLOs and where are they measured? 4) Are burn-rate alerts in place and tested?
**Capacity:** 5) What is the expected peak, and what load test proves it? 6) Requests/limits based on data? 7) HPA configured and tested? What caps it?
**Resilience:** 8) ≥ 3 replicas spread across zones? 9) Probes correct (liveness local only)? 10) Graceful shutdown tested under load? 11) PDB allows drains?
**Delivery:** 12) GitOps, canary with automated analysis, and a one-command rollback?
**Security:** 13) Least-privilege ServiceAccount, PSS restricted, NetworkPolicies, secrets from a manager, signed images?
**Data:** 14) Backups, restore tested, RPO/RTO defined?
**Failure:** 15) Which failure modes were tested in a game day, and what was learned?
</details>

---

## ❓ Quiz

1. An SLO of 99.9% over 30 days allows about how much full downtime?

   - a) 43 min
   - b) 4 min
   - c) 1 day
   - d) 7 h

2. A burn rate of 1 means…

   - a) 1% errors
   - b) You would consume exactly the whole budget by the end of the window
   - c) The service is down
   - d) One alert per hour

3. PodDisruptionBudgets protect against…

   - a) Node hardware failures
   - b) Container crashes
   - c) OOM kills
   - d) Voluntary disruptions like drains and scale-downs

4. Why use two windows in burn-rate alerts?

   - a) For two teams
   - b) Prometheus requires it
   - c) To double the alerts
   - d) Significance (long window) plus fast reset (short window)

5. The first step of a chaos experiment is…

   - a) Notify customers
   - b) Install a tool
   - c) Kill a node
   - d) Define the steady state (SLI) and a hypothesis

<details><summary>Show answers</summary>

1. **a)** 43 min
2. **b)** You would consume exactly the whole budget by the end of the window
3. **d)** Voluntary disruptions like drains and scale-downs
4. **d)** Significance (long window) plus fast reset (short window)
5. **d)** Define the steady state (SLI) and a hypothesis
</details>

➡️ Next module: [29 · What's New in v1.35 – v1.37](../29-whats-new/)
