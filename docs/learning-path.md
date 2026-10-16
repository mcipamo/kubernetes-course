# 🧭 Learning paths

## Module dependencies

```mermaid
flowchart TD
    M01[01 Containers] --> M02[02 Linux/Networking/YAML] --> M03[03 Local lab]
    M03 --> M04[04 Architecture] --> M05[05 Pods]
    M05 --> M06[06 Namespaces/Labels] --> M07[07 Deployments] --> M08[08 Services/DNS]
    M05 --> M09[09 ConfigMaps/Secrets]
    M07 --> M10[10 Resources/QoS] --> M11[11 Probes]
    M09 --> M12[12 Storage] --> M13[13 StatefulSets/Jobs]
    M08 --> M14[14 Ingress/Gateway]
    M10 --> M15[15 Scheduling]
    M10 --> M16[16 Autoscaling]
    M06 --> M17[17 RBAC] --> M18[18 Pod Security/NetPol]
    M14 --> M19[19 Networking deep dive]
    M11 --> M20[20 Observability] --> M21[21 Troubleshooting]
    M04 --> M22[22 Cluster admin]
    M07 --> M23[23 Kustomize/Helm] --> M24[24 GitOps]
    M17 --> M25[25 CRDs/Operators] --> M26[26 Policies]
    M15 --> M27[27 DRA/GPUs/AI]
    M16 & M20 & M21 --> M28[28 Reliability]
    M28 --> M29[29 What's new] --> CAP[🎓 Capstone]
```

## Paths by role

| Role | Recommended path | Length |
|---|---|---|
| 👩‍💻 Developer | 01–14, 16, 20, 23, 24 | ~10 weeks |
| 🛠️ Administrator / Ops | All of levels 0–3, then 23, 28 | ~15 weeks |
| 🧯 Platform Engineer | Full course + capstone | 20 weeks |
| 🔐 Security | 01–09, 17, 18, 22, 25, 26 | ~10 weeks |
| 🤖 ML/AI Engineer | 01–13, 15, 16, 27 | ~10 weeks |

## Paths by certification (CNCF / Linux Foundation)

| Certification | Key modules | Tip |
|---|---|---|
| **KCNA** | 01–09, 20, 24 | Theory and vocabulary; the quizzes are good practice |
| **CKAD** | 05–14, 16, 23 | Speed with `--dry-run=client -o yaml` |
| **CKA** | 04–22 | Master `drain`, etcd backups and node troubleshooting |
| **CKS** | 17, 18, 22, 26 + supply-chain security | Requires an active CKA |

> Always check the current official curriculum at https://github.com/cncf/curriculum — it changes with each release.
