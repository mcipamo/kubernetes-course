# 📖 Glossary

One-line definitions of the terms used in the course. The number in brackets is the module where the
term is explained. See also the [official glossary](https://kubernetes.io/docs/reference/glossary/).

| Term | Definition |
|---|---|
| **Admission controller** | Code in the API server that mutates or validates requests after authorization [26] |
| **Affinity / anti-affinity** | Rules that attract or repel Pods to/from nodes or other Pods [15] |
| **Annotation** | Non-identifying key/value metadata read by tools; not selectable [06] |
| **API server** (`kube-apiserver`) | The front door of the control plane; the only component that talks to etcd [04] |
| **Burn rate** | How fast an error budget is consumed relative to the SLO window [28] |
| **cAdvisor** | Container metrics collector built into the kubelet [20] |
| **CEL** | Common Expression Language, used in CRD validation and admission policies [25, 26] |
| **cgroups** | Linux feature that limits CPU, memory and I/O for a group of processes [01] |
| **CNI** | Container Network Interface: plugins that give Pods IPs and connectivity [19] |
| **ConfigMap** | Object holding non-sensitive configuration [09] |
| **conntrack** | Kernel table of tracked connections used by Service NAT [19] |
| **Container** | An isolated process created from an image [01] |
| **Context** (kubeconfig) | A cluster + user + default namespace combination [03] |
| **Control plane** | API server, etcd, scheduler and controllers: the cluster's brain [04] |
| **Controller** | A loop that drives actual state toward desired state [04] |
| **CoreDNS** | The cluster DNS server [08] |
| **CRD** | CustomResourceDefinition: adds a new resource type to the API [25] |
| **CRI** | Container Runtime Interface between the kubelet and runtimes like containerd [04] |
| **CronJob** | Creates Jobs on a schedule [13] |
| **CSI** | Container Storage Interface: plugins for storage systems [12] |
| **DaemonSet** | Runs one Pod per (selected) node [13] |
| **Deployment** | Manages ReplicaSets for declarative rolling updates [07] |
| **DeviceClass / ResourceClaim / ResourceSlice** | DRA objects to describe, request and publish devices [27] |
| **DRA** | Dynamic Resource Allocation for GPUs and other devices [27] |
| **Drain** | Evict all Pods from a node safely, respecting PDBs [22] |
| **EndpointSlice** | The list of ready backend addresses of a Service [08] |
| **Error budget** | `1 − SLO`: the amount of unreliability you are allowed [28] |
| **etcd** | Consistent key-value store that holds all cluster state [04, 22] |
| **Eviction** | Removing a Pod from a node (API-initiated or because of node pressure) [10, 22] |
| **Feature gate** | A switch that enables/disables alpha/beta features [29] |
| **Finalizer** | A marker that blocks deletion until a controller cleans up [25] |
| **Gateway API** | Role-oriented API for L4/L7 traffic routing (GatewayClass, Gateway, HTTPRoute…) [14] |
| **GitOps** | Operating model where Git is the source of truth, pulled and reconciled by agents [24] |
| **Headless Service** | Service without a virtual IP; DNS returns Pod IPs [08] |
| **Helm** | Package manager for Kubernetes (charts, values, releases) [23] |
| **HPA** | HorizontalPodAutoscaler: scales replicas from metrics [16] |
| **Image** | Immutable, layered package of an app and its dependencies [01] |
| **Ingress** | Older API for HTTP routing into the cluster (feature-frozen) [14] |
| **Init container** | Runs to completion before app containers start [05] |
| **KEP** | Kubernetes Enhancement Proposal [29] |
| **kind** | Kubernetes IN Docker: local clusters in containers [03] |
| **kube-proxy** | Programs Service rules (iptables/nftables/IPVS) on each node [08, 19] |
| **kubelet** | Node agent that runs Pods and reports status [04] |
| **Kustomize** | Template-free patching of YAML (base + overlays) [23] |
| **Label / selector** | Key/value tags and the queries that match them [06] |
| **Lease** | Lightweight object for heartbeats and leader election [04] |
| **LimitRange** | Default and maximum resources per container in a namespace [10] |
| **Liveness probe** | Restarts a container that is stuck [11] |
| **Namespace** | A scope for names, RBAC, quotas and policies [06] |
| **NetworkPolicy** | Allow-list rules for Pod traffic [18] |
| **Node** | A machine (VM or physical) that runs Pods [04] |
| **OOMKilled** | Container killed for exceeding its memory limit (exit 137) [10] |
| **Operator** | A controller that encodes operational knowledge for an app [25] |
| **Owner reference** | Link from a child object to its parent for garbage collection [25] |
| **PDB** | PodDisruptionBudget: limits voluntary disruptions [22, 28] |
| **PersistentVolume (PV) / Claim (PVC)** | A piece of storage and a request for it [12] |
| **Pod** | The smallest deployable unit: one or more containers sharing network and storage [05] |
| **Pod Security Standards / Admission** | Privileged, baseline, restricted profiles and their built-in enforcement [18] |
| **PriorityClass** | Relative importance of Pods, used for scheduling and preemption [15] |
| **QoS class** | Guaranteed, Burstable or BestEffort, from requests/limits [10] |
| **RBAC** | Role-based access control: Roles, ClusterRoles and bindings [17] |
| **Readiness probe** | Controls whether a Pod receives Service traffic [11] |
| **Reconciliation** | The continuous process of making actual state match desired state [04] |
| **ReplicaSet** | Keeps N identical Pods running [07] |
| **Requests / limits** | Guaranteed share (scheduling) and maximum allowed (enforcement) of resources [10] |
| **ResourceQuota** | Aggregate resource budget per namespace [10] |
| **RPO / RTO** | Maximum data loss / maximum time to restore after a disaster [12, 22] |
| **Runbook** | Step-by-step guide for responding to an alert [21, 28] |
| **Scheduler** (`kube-scheduler`) | Assigns Pods to nodes (filter → score → bind) [04, 15] |
| **Secret** | Object for sensitive data (base64-encoded, not encrypted by default) [09] |
| **Service** | Stable virtual IP and DNS name in front of a set of Pods [08] |
| **ServiceAccount** | Identity for workloads inside the cluster [17] |
| **Sidecar container** | Helper container running alongside the app (native: init container with `restartPolicy: Always`) [05] |
| **SLI / SLO / SLA** | Indicator, objective and agreement for service reliability [28] |
| **StatefulSet** | Controller for Pods with stable identity and per-replica storage [13] |
| **StorageClass** | A "type" of storage with a provisioner and parameters [12] |
| **Taint / toleration** | Node-side repulsion and Pod-side permission for scheduling [15] |
| **Topology spread** | Rules to distribute Pods across zones/nodes [15] |
| **User namespaces** | Map container users to unprivileged host users (`hostUsers: false`) [18, 29] |
| **ValidatingAdmissionPolicy** | In-process CEL validation policies [26] |
| **VPA** | Vertical Pod Autoscaler: recommends/sets requests [16] |
