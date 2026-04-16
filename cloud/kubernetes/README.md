# Kubernetes — Knowledge Base

A reference library covering Kubernetes patterns, primitives, and operational practices from workload fundamentals through production hardening.

---

## Topic Index

| #   | Topic                                                                         | Category                              | Description                                                         |
| --- | ----------------------------------------------------------------------------- | ------------------------------------- | ------------------------------------------------------------------- |
| 01  | [Workloads](./01-workloads.md)                                                | Core, Scheduling                      | Deployment, StatefulSet, DaemonSet, Job, CronJob                    |
| 02  | [Networking & Services](./02-networking-services.md)                          | Networking                            | Services, Ingress, Gateway API, NetworkPolicy                       |
| 03  | [Storage](./03-storage.md)                                                    | Storage, Stateful                     | PV, PVC, StorageClass, CSI drivers, ReadWriteMany                   |
| 04  | [Configuration & Secrets](./04-configuration-secrets.md)                     | Configuration, Security               | ConfigMap, Secret, External Secrets Operator, Sealed Secrets        |
| 05  | [RBAC & Security](./05-rbac-security.md)                                      | Security, IAM                         | RBAC, ServiceAccount, Pod Security Admission, OPA/Gatekeeper        |
| 06  | [Autoscaling](./06-autoscaling.md)                                            | Scalability, Cost                     | HPA, VPA, KEDA, Karpenter, Cluster Autoscaler                       |
| 07  | [Helm & Kustomize](./07-helm-kustomize.md)                                    | Packaging, GitOps                     | Helm charts, Kustomize overlays, OCI registries                     |
| 08  | [GitOps — ArgoCD & Flux](./08-gitops.md)                                      | GitOps, Delivery                      | ArgoCD app-of-apps, FluxCD, image automation, progressive delivery  |
| 09  | [Observability](./09-observability.md)                                        | Observability, Monitoring             | Prometheus Operator, Grafana, Loki, OpenTelemetry Collector         |
| 10  | [Service Mesh](./10-service-mesh.md)                                          | Networking, Security                  | Istio, Linkerd, Cilium, mTLS, traffic management                    |
| 11  | [Multi-tenancy](./11-multi-tenancy.md)                                        | Security, Multi-tenancy               | Namespaces, ResourceQuota, LimitRange, vCluster, HNC                |
| 12  | [Operators & CRDs](./12-operators-crds.md)                                    | Extensibility, Platform               | CRDs, Operator pattern, controller-runtime, kopf                    |
| 13  | [Pod Reliability](./13-pod-reliability.md)                                    | Resilience, Availability              | PodDisruptionBudget, topology spread, priority classes, probes      |
| 14  | [Supply Chain Security](./14-supply-chain-security.md)                        | Security, Compliance                  | Cosign, SBOM, Sigstore, OPA Gatekeeper, admission webhooks          |
| 15  | [Cost Optimisation](./15-cost-optimisation.md)                                | FinOps, Cost                          | Spot nodes, right-sizing, Goldilocks, Kubecost, bin packing         |
| 16  | [Platform Engineering & IDP](./16-platform-engineering.md)                    | Platform, Developer Experience        | Backstage, Crossplane, Kyverno, golden paths, service catalog       |
| 17  | [Cluster Upgrades & Node Maintenance](./17-cluster-upgrades.md)               | Operations, Reliability               | Rolling upgrades, Pluto, PDB, Karpenter drift, drain runbook        |
| 18  | [Kubernetes Best Practices](./18-kubernetes-best-practices.md)                | Learning, Production Excellence       | Opinionated checklist for reliability, security, operations, scaling |
| 19  | [Kubernetes Anti-Patterns](./19-kubernetes-anti-patterns.md)                  | Learning, Risk Reduction              | Common failure modes, why they fail, and safer alternatives          |

---

## Decision Guide

### Which workload type should I use?

| Situation | Resource |
|-----------|----------|
| Stateless web service / API | `Deployment` |
| Database, message broker, ordered pods with stable identity | `StatefulSet` |
| Log agent, node-level monitoring on every node | `DaemonSet` |
| One-off batch task | `Job` |
| Recurring scheduled batch | `CronJob` |
| Long-running stateless task from a queue | `Deployment` + KEDA |

### Which service type should I use?

| Situation | Type |
|-----------|------|
| Internal cluster traffic only | `ClusterIP` |
| Expose directly on node port (dev/debug) | `NodePort` |
| Cloud load balancer (production ingress) | `LoadBalancer` |
| HTTP routing, TLS termination, path-based routing | `Ingress` + Ingress Controller |
| Advanced traffic management, header routing, gRPC | `Gateway API` (`HTTPRoute`) |

### When should I use what scaling approach?

| Situation | Approach |
|-----------|----------|
| Scale pods on CPU/memory | HPA |
| Right-size resource requests | VPA (in recommendation mode) |
| Scale on external metrics (queue depth, Kafka lag) | KEDA |
| Right-size and replace nodes cost-optimally | Karpenter |
| Simple cluster node scaling (existing infra) | Cluster Autoscaler |

### GitOps tool choice

| Situation | Tool |
|-----------|------|
| UI-first, rich app overview, progressive delivery with Argo Rollouts | ArgoCD |
| Git-native, multi-tenancy, image automation, Helm + Kustomize natively | FluxCD |
| Both are fine — follow team/org preference | Either |

### Learning Path: Best Practices vs Anti-Patterns

| Goal | Start here |
|------|------------|
| Build a production baseline from scratch | [Kubernetes Best Practices](./18-kubernetes-best-practices.md) |
| Audit an existing cluster for hidden risks | [Kubernetes Anti-Patterns](./19-kubernetes-anti-patterns.md) |
| Prepare teams for upgrade and maintenance reliability | [Cluster Upgrades & Node Maintenance](./17-cluster-upgrades.md) |
| Improve day-2 platform consistency | [Platform Engineering & IDP](./16-platform-engineering.md) |

---

## Frequently Combined Patterns

| Combination | Why |
|-------------|-----|
| **Deployment + HPA + PDB** | Scalable service with availability guarantees during rollouts |
| **StatefulSet + StorageClass + PVC** | Stateful database workload with persistent, dynamic storage |
| **KEDA + Spot nodes (Karpenter)** | Cost-optimised event-driven scaling |
| **ArgoCD + Kustomize overlays** | Environment-specific config with GitOps delivery |
| **Istio + OPA Gatekeeper** | Traffic policy + admission policy for zero-trust |
| **Backstage + Crossplane + Kyverno** | Self-service IDP with governance guardrails |
| **Pluto + PDB + kubectl drain** | Safe zero-downtime cluster upgrades |
| **External Secrets Operator + Vault/AWS SM** | Secret lifecycle managed outside the cluster |
| **Prometheus + Loki + Tempo + Grafana** | Full observability stack (metrics + logs + traces) |
| **Cosign + OPA Gatekeeper** | Only signed, policy-compliant images run in production |
