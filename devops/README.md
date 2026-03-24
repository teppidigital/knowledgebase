# DevOps Knowledge Base

A comprehensive reference for DevOps patterns, practices, and tooling covering the full software delivery lifecycle — from CI/CD pipeline design through to operational excellence and continuous improvement.

> **Relationship to DevSecOps**: The [`devsecops/`](../devsecops/README.md) section covers security controls embedded in the delivery pipeline (SAST, SCA, DAST, secret scanning, container security, SBOM, policy-as-code). This section covers the broader operational and delivery patterns.

---

## Patterns Index

| # | Pattern | Key Topics |
|---|---------|-----------|
| [01](01-cicd-pipeline-design.md) | **CI/CD Pipeline Design** | Pipeline stages, branching strategies, artifact versioning, GitHub Actions, parallel jobs, semantic versioning, immutable artifacts |
| [02](02-gitops.md) | **GitOps** | Pull-based deployments, ArgoCD, Flux, reconciliation loops, ApplicationSet, HelmRelease, ImageUpdateAutomation, repo promotion |
| [03](03-infrastructure-as-code.md) | **Infrastructure as Code** | Terraform, AKS modules, remote state, OIDC auth, speculative plans on PR, drift detection, HCL module composition |
| [04](04-deployment-strategies.md) | **Deployment Strategies** | Rolling update, Blue-Green, Canary, Argo Rollouts, traffic splitting, AnalysisTemplate, Nginx ingress, Prometheus-based analysis |
| [05](05-feature-flags.md) | **Feature Flags & Progressive Delivery** | OpenFeature SDK, LaunchDarkly, Flagd, fractional targeting, kill switches, stale flag detection |
| [06](06-observability-opentelemetry.md) | **Observability & OpenTelemetry** | Three pillars, OTel NodeSDK, OTLP exporters, RED method, tail sampling, OTel Collector, Tempo, Loki, Prometheus |
| [07](07-sre-slos.md) | **SRE Practices & SLOs** | SLI/SLO/SLA, error budgets, burn-rate alerts, multi-window alerting, toil reduction, golden signals |
| [08](08-chaos-engineering.md) | **Chaos Engineering** | LitmusChaos, Chaos Toolkit, game days, pod-network-latency, pod-delete, ChaosSchedule, steady-state hypothesis |
| [09](09-database-devops.md) | **Database DevOps** | Expand-Contract pattern, Flyway, Atlas, online DDL, batched backfill migrations, init containers, zero-downtime schema changes |
| [10](10-alerting-oncall.md) | **Alerting & On-Call Management** | Alertmanager, PagerDuty, Slack, inhibit rules, priority severity model, business-hours routing, alert quality principles |
| [11](11-platform-engineering.md) | **Platform Engineering & IDP** | Backstage, Software Templates, Crossplane XRDs, golden paths, paved-road CI/CD, reusable GitHub Actions workflows |
| [12](12-artifact-release-management.md) | **Artifact & Release Management** | OCI registries, Cosign signing, SBOM attestation, SLSA provenance, Kyverno image policy, semantic versioning automation |
| [13](13-configuration-management.md) | **Configuration Management** | Helm values hierarchy, Kustomize overlays, External Secrets Operator, Azure Key Vault, Ansible, config drift detection |
| [14](14-disaster-recovery.md) | **Disaster Recovery & Business Continuity** | RTO/RPO, Velero, cross-region failover, Traffic Manager, PostgreSQL replica promotion, DR runbook automation |
| [15](15-devops-metrics.md) | **DevOps Metrics & Continuous Improvement** | DORA four key metrics, deployment frequency, lead time, change failure rate, MTTR, Prometheus/Grafana dashboards |

---

## Patterns by Category

### Pipeline & Delivery

- [01 — CI/CD Pipeline Design](01-cicd-pipeline-design.md) — Build, test, package, deploy pipeline architecture
- [02 — GitOps](02-gitops.md) — Pull-based deployments with ArgoCD and Flux
- [04 — Deployment Strategies](04-deployment-strategies.md) — Safe, progressive production releases
- [05 — Feature Flags](05-feature-flags.md) — Decouple deploy from release

### Infrastructure

- [03 — Infrastructure as Code](03-infrastructure-as-code.md) — Terraform, Kubernetes, cloud resources
- [13 — Configuration Management](13-configuration-management.md) — Helm, Kustomize, Ansible, secrets
- [12 — Artifact & Release Management](12-artifact-release-management.md) — OCI registries, signing, provenance

### Reliability & Operations

- [06 — Observability & OpenTelemetry](06-observability-opentelemetry.md) — Traces, metrics, logs pipeline
- [07 — SRE Practices & SLOs](07-sre-slos.md) — Error budgets, burn-rate alerts, toil reduction
- [08 — Chaos Engineering](08-chaos-engineering.md) — Fault injection, game days, ChaosEngine
- [10 — Alerting & On-Call](10-alerting-oncall.md) — Alert routing, severity, on-call runbooks
- [14 — Disaster Recovery](14-disaster-recovery.md) — Cross-region failover, backup, RTO/RPO

### Data

- [09 — Database DevOps](09-database-devops.md) — Schema migrations, expand-contract, online DDL

### Platform & Measurement

- [11 — Platform Engineering](11-platform-engineering.md) — IDP, Backstage, Crossplane, paved roads
- [15 — DevOps Metrics](15-devops-metrics.md) — DORA, continuous improvement loop

---

## Tool Ecosystem Reference

| Tool | Category | What it does |
|------|----------|-------------|
| **GitHub Actions** | CI/CD | Workflow automation for build, test, deploy |
| **ArgoCD** | GitOps | Pull-based K8s deployment reconciler |
| **Flux** | GitOps | CNCF GitOps toolkit for K8s |
| **Terraform** | IaC | Declarative cloud infrastructure provisioning |
| **Helm** | Package/Config | Kubernetes application packaging and deployment |
| **Kustomize** | Config | Kubernetes manifest patching without templating |
| **Argo Rollouts** | Deployment | Canary and blue-green deployments |
| **OpenFeature** | Feature Flags | Vendor-agnostic feature flag SDK |
| **Flagd** | Feature Flags | Self-hosted OpenFeature provider |
| **OpenTelemetry** | Observability | Vendor-neutral instrumentation SDK and collector |
| **Prometheus** | Metrics | Time-series metrics collection and alerting rules |
| **Grafana** | Dashboards | Metrics, logs, and trace visualisation |
| **Alertmanager** | Alerting | Alert routing, grouping, and deduplication |
| **LitmusChaos** | Chaos | Kubernetes chaos experiment framework |
| **Flyway / Atlas** | DB migrations | Schema migration tooling with version control |
| **Velero** | DR / Backup | Kubernetes backup, restore, and disaster recovery |
| **External Secrets Operator** | Config/Secrets | Sync secrets from Vault/Key Vault into K8s |
| **Cosign / Sigstore** | Supply Chain | Container image signing and verification |
| **Kyverno** | Policy | K8s admission controller and policy engine |
| **Backstage** | Platform | Developer portal and software catalogue |
| **Crossplane** | Platform | K8s-native cloud resource provisioning |
| **PagerDuty** | On-Call | Incident management and routing |

---

## DORA Metric Reference

| Metric | Elite threshold | Primary tool for collection |
|--------|----------------|-----------------------------|
| Deployment Frequency | Multiple per day | GitHub Deployments API |
| Lead Time for Changes | < 1 hour | GitHub PRs + Deployments API |
| Change Failure Rate | 0–5% | PagerDuty incidents tagged `deployment-caused` |
| MTTR | < 1 hour | PagerDuty incident resolved time |

See [15 — DevOps Metrics](15-devops-metrics.md) for the full collector implementation.

---

## Related Sections

| Section | Focus |
|---------|-------|
| [`devsecops/`](../devsecops/README.md) | Security controls in CI/CD: SAST, SCA, DAST, secret scanning, SBOM, policy-as-code |
| [`cloud-native/aws/`](../cloud-native/aws/README.md) | AWS-specific cloud-native patterns (EKS, S3, Lambda, RDS) |
| [`cloud-native/azure/`](../cloud-native/azure/README.md) | Azure-specific cloud-native patterns (AKS, Azure Functions, Cosmos, APIM) |
| [`system-design/`](../system-design/README.md) | Distributed system architecture patterns |
| [`security/`](../security/README.md) | Application and infrastructure security patterns |
