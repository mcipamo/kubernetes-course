<img src="assets/kubernetes-icon-color.svg" alt="Kubernetes logo" width="64" align="right">

# 📅 Syllabus — Kubernetes Course

| Field | Detail |
|---|---|
| **Program** | Platform Engineering and Reliability |
| **Course codes** | KC-101 to KC-590 |
| **Total credits** | 33 (1 credit ≈ 6 hours of work) |
| **Suggested length** | 20 weeks at ~10 h/week |
| **Format** | Self-paced, hands-on, 100% on a local lab cluster |
| **Reference version** | Kubernetes v1.37 |
| **Primary source** | https://kubernetes.io/docs/home/ |

---

## 🎯 Program learning outcomes

By the end of the program you will be able to:

1. **Explain** the Kubernetes architecture and its declarative reconciliation model.
2. **Deploy** stateless and stateful applications with configuration, storage and networking.
3. **Secure** a cluster with RBAC, Pod Security Standards, NetworkPolicy and admission policies.
4. **Operate** a cluster: upgrades, etcd backups, certificates and node maintenance.
5. **Diagnose** failures with a systematic, evidence-based method.
6. **Design** reliable platforms with SLOs, autoscaling, GitOps and disaster recovery.
7. **Extend** Kubernetes with CRDs, operators and policies.

## 🧮 Academic structure

| Code | Level | Modules | Credits | Hours | Weeks |
|---|---|---|---|---|---|
| KC-101 | 🌱 Foundations | 01–03 | 3 | 12 | 1–2 |
| KC-201 | 🧱 Basics | 04–09 | 6 | 30 | 3–6 |
| KC-301 | ⚙️ Intermediate | 10–15 | 6 | 36 | 7–10 |
| KC-401 | 🛡️ Advanced | 16–22 | 7 | 45 | 11–15 |
| KC-501 | 🚀 Expert | 23–29 | 7 | 45 | 16–19 |
| KC-590 | 🎓 Capstone | — | 4 | 30 | 20 |

## 📏 Assessment

| Component | Weight | How it is assessed |
|---|---|---|
| Labs (29) | 35% | Every ✅ check passes + diagnosis of the 💥 failure |
| Exercises and quizzes | 20% | Self-assessed with solutions; ≥ 80% to move to the next level |
| Mock exams (3) | 15% | End of levels 1, 2 and 3 (CKAD / CKA / CKS style) |
| Capstone | 30% | Production Readiness Review using the [rubric](docs/grading-rubric.md) |

**To pass:** ≥ 70% overall and a passing capstone.

## 📆 Suggested schedule

| Week | Modules | Milestone |
|---|---|---|
| 1 | 01, 02 | — |
| 2 | 03 | ✅ kind cluster up and running |
| 3 | 04, 05 | — |
| 4 | 06, 07 | — |
| 5 | 08 | — |
| 6 | 09 | 📝 Mock exam 1 (CKAD style) |
| 7 | 10, 11 | — |
| 8 | 12, 13 | — |
| 9 | 14 | — |
| 10 | 15 | 📝 Mock exam 2 (CKA style, part 1) |
| 11 | 16, 17 | — |
| 12 | 18 | — |
| 13 | 19, 20 | — |
| 14 | 21 | — |
| 15 | 22 | 📝 Mock exam 3 (CKA/CKS style) |
| 16 | 23, 24 | — |
| 17 | 25, 26 | — |
| 18 | 27, 28 | — |
| 19 | 29 | — |
| 20 | Capstone | 🎓 Production Readiness Review |

## 🧰 Technical requirements

| Resource | Minimum | Recommended |
|---|---|---|
| CPU | 4 cores | 8 cores |
| RAM | 8 GB | 16 GB |
| Free disk | 20 GB | 40 GB |
| OS | macOS, Linux or Windows with WSL2 | — |
| Software | Docker/Podman, `kubectl`, `kind` | + `helm`, `k9s`, `jq`, `yq` |

## 📐 Module format

| File | Contents |
|---|---|
| `README.md` | Analogy · concepts · diagram · commands · SRE mindset · common mistakes · references |
| `LAB.md` | Step-by-step lab with ✅ checks and a 💥 deliberate failure |
| `EXERCISES.md` | 🟢🟡🔴 exercises · quiz · solutions |

## ⚖️ Academic policy

- Do every lab **on your own cluster**. Never on a production cluster.
- Copying commands is fine; **understanding why they work** is the goal.
- Using an AI assistant as a tutor is allowed and encouraged: ask it for analogies and questions, not just answers.
