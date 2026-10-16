# 📏 Grading rubric

## Labs (per lab)

| Criterion | Excellent (4) | Good (3) | Fair (2) | Poor (1) |
|---|---|---|---|---|
| Execution | Every ✅ check passes | 1 check missing | 2–3 missing | More than 3 |
| 💥 Diagnosis | Root cause + evidence + prevention | Root cause + evidence | Cause only | Not found |
| Understanding | Explains *why* for every step | Explains most | Explains some | Only copies commands |
| Cleanup | Namespace deleted, nothing left | Minor leftovers | Leftovers | No cleanup |

## Capstone — Production Readiness Review

| Dimension | Reviewer's question | Weight |
|---|---|---|
| 🏗️ Architecture | Is it documented with a diagram and decisions (ADRs)? | 10% |
| 🚀 Delivery | Can it be deployed from scratch with GitOps in < 15 min? | 15% |
| 🔐 Security | Least-privilege RBAC, PSS `restricted`, default-deny NetworkPolicy, no secrets in Git? | 20% |
| 📈 Scalability | Is HPA configured and load-tested? | 10% |
| 🩺 Health | Correct probes, PDBs, requests/limits everywhere? | 15% |
| 👀 Observability | Metrics, logs, a dashboard and SLO-based alerts? | 15% |
| 🔥 Resilience | Survives losing a node, and backup/restore is tested? | 10% |
| 📓 Operations | Runbooks for the top 3 alerts? | 5% |

**Capstone pass mark:** ≥ 70% and no dimension at 0.
