# ✅ Production Readiness Review — Checklist

**Service:** · **Owner team:** · **On-call rotation:** · **Review date:** · **Reviewers:**

| Area | Question | Status | Evidence / link |
|---|---|---|---|
| Ownership | Is there a named owner and an on-call rotation? | ⬜ | |
| Ownership | Are dependencies listed with their SLOs? | ⬜ | |
| SLOs | Are SLIs measured at the edge, with SLOs and an error budget policy? | ⬜ | |
| SLOs | Are burn-rate alerts configured and tested? | ⬜ | |
| Capacity | Are requests/limits based on measurements? | ⬜ | |
| Capacity | Is autoscaling configured and load-tested? What is the ceiling? | ⬜ | |
| Resilience | ≥ 3 replicas spread across zones and nodes? | ⬜ | |
| Resilience | Probes correct (liveness local only), graceful shutdown tested? | ⬜ | |
| Resilience | PDB present and allows ≥ 1 disruption? | ⬜ | |
| Resilience | Game day results documented? | ⬜ | |
| Delivery | GitOps-managed; canary or blue/green; one-step rollback? | ⬜ | |
| Security | Least-privilege SA/RBAC; PSS restricted; default-deny NetworkPolicy? | ⬜ | |
| Security | Secrets from a manager; images pinned and signed? | ⬜ | |
| Data | Backups automated, restore tested, RPO/RTO measured? | ⬜ | |
| Observability | Dashboards (golden signals), logs, traces available? | ⬜ | |
| Operations | Runbooks for every paging alert? | ⬜ | |

**Decision:** ⬜ Ready · ⬜ Ready with follow-ups · ⬜ Not ready

**Follow-ups:**
| Action | Owner | Due |
|---|---|---|
