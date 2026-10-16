# Runbook: <AlertName>

| Field | Value |
|---|---|
| Severity | page / ticket |
| Service | |
| Dashboard | <link> |
| Owner / escalation | |

## What it means
One paragraph: which SLI/condition fired and what users experience.

## Impact
Who is affected and how badly (e.g., "checkout fails for ~X% of users").

## Diagnose
```bash
# 1. Scope: is it one Pod, one node, one zone, everything?
kubectl -n <ns> get pods -o wide
kubectl -n <ns> events --types=Warning | tail
# 2. Recent changes
kubectl -n <ns> rollout history deploy/<name>
# 3. Specific checks for this alert
```

## Mitigate
- [ ] If a deploy happened in the last hour: `kubectl -n <ns> rollout undo deploy/<name>` (or `git revert`)
- [ ] If one node is bad: `kubectl cordon <node> && kubectl drain <node> --ignore-daemonsets`
- [ ] If overloaded: scale up / enable load shedding

## Verify
The SLI is back within SLO for 15 minutes; the alert resolved.

## Follow-up
Open a postmortem if the error budget burn was > X% or users were affected.
