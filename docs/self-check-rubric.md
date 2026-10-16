# 📏 Self-check Rubric

Use this as a checklist to review your own work (or a friend's). There is no score to reach —
the goal is to spot what is missing.

## Labs (per lab)

| Criterion | ✅ Solid | 🟡 Almost | 🟠 Partial | 🔴 Not yet |
|---|---|---|---|---|
| Execution | Every ✅ check passes | 1 check missing | 2–3 missing | More than 3 |
| 💥 Diagnosis | Root cause + evidence + prevention | Root cause + evidence | Cause only | Not found |
| Understanding | Explains *why* for every step | Explains most | Explains some | Only copies commands |
| Cleanup | Namespace deleted, nothing left | Minor leftovers | Leftovers | No cleanup |

## Capstone — Production Readiness Review

| Dimension | Question to ask yourself |
|---|---|
| 🏗️ Architecture | Is it documented with a diagram and decisions (ADRs)? |
| 🚀 Delivery | Can it be deployed from scratch with GitOps in < 15 min? |
| 🔐 Security | Least-privilege RBAC, PSS `restricted`, default-deny NetworkPolicy, no secrets in Git? |
| 📈 Scalability | Is HPA configured and load-tested? |
| 🩺 Health | Correct probes, PDBs, requests/limits everywhere? |
| 👀 Observability | Metrics, logs, a dashboard and SLO-based alerts? |
| 🔥 Resilience | Survives losing a node, and backup/restore is tested? |
| 📓 Operations | Runbooks for the top 3 alerts? |

**Done when** you can answer "yes" to every question and show the evidence.
