# FinOps — Cloud Cost Discipline

> **15 patterns** for engineering cloud cost efficiency: from visibility and alerting to commitment-based discounts, auto-scaling, and FinOps culture maturity.

---

## Patterns

| # | Pattern | Key Tools | Typical Saving |
|---|---------|-----------|---------------|
| 01 | [Cost Monitoring & Alerting](01-cost-monitoring-alerting.md) | AWS Budgets, Cost Anomaly Detection, CUR | Prevents bill surprises |
| 02 | [Resource Rightsizing](02-resource-rightsizing.md) | Compute Optimizer, CloudWatch | 20–40% compute spend |
| 03 | [Reserved Instances & Savings Plans](03-reserved-instances-savings-plans.md) | Savings Plans, RIs | 30–72% baseline compute |
| 04 | [Spot & Preemptible Instances](04-spot-preemptible-instances.md) | EC2 Spot, Node Termination Handler | 60–90% batch compute |
| 05 | [Auto-Scaling & Scale-to-Zero](05-auto-scaling-scale-to-zero.md) | ASG, KEDA, ECS | 60–70% dev environment |
| 06 | [Storage Tiering & Lifecycle](06-storage-tiering-lifecycle.md) | S3 Lifecycle, Glacier, EBS GP3 | 40–80% storage spend |
| 07 | [Data Transfer Cost Optimisation](07-data-transfer-cost-optimisation.md) | VPC Endpoints, CloudFront, PrivateLink | 20–60% egress cost |
| 08 | [Kubernetes Cost Allocation](08-kubernetes-cost-allocation.md) | Kubecost, OpenCost, LimitRange | Cost visibility + 20% efficiency |
| 09 | [Serverless Cost Optimisation](09-serverless-cost-optimisation.md) | Lambda Power Tuning, ARM64, Batch SQS | 20–70% Lambda spend |
| 10 | [Database Cost Optimisation](10-database-cost-optimisation.md) | Aurora Serverless v2, RDS stop/start | 40–65% DB spend |
| 11 | [CDN & Caching Cost Reduction](11-cdn-caching-cost.md) | CloudFront, Origin Shield, Cache-Control | 60–90% egress from CDN |
| 12 | [Tagging & Cost Allocation](12-tagging-cost-allocation.md) | Tag Policies, AWS Config, SCP | Enables all attribution |
| 13 | [Showback & Chargeback](13-showback-chargeback.md) | CUR, Athena, Grafana | Drives team accountability |
| 14 | [Cost Forecasting & Budgeting](14-cost-forecasting-budgeting.md) | CE Forecast API, Unit Economics | Proactive cost control |
| 15 | [FinOps Culture & Maturity](15-finops-culture-maturity.md) | FinOps Foundation, Cost Champions | Sustainable reduction |

---

## Decision Guide

### Where to start?

```
Is your bill visible by team?
  No → Start with 01 (Monitoring) and 12 (Tagging)
  Yes ↓

Is your compute over-provisioned?
  Yes → 02 (Rightsizing) + 04 (Spot)
  No ↓

Is your baseline running 24/7?
  Yes → 03 (Reserved Instances / Savings Plans)
  No ↓

Do you have dev/test environments running at night?
  Yes → 05 (Scale-to-Zero) + 10 (RDS stop/start)
  No ↓

Is egress cost high?
  Yes → 07 (Data Transfer) + 11 (CDN)
  No ↓

Do you have Kubernetes workloads?
  Yes → 08 (K8s Cost Allocation)

Do you have Lambda workloads?
  Yes → 09 (Serverless Cost Optimisation)

To drive sustained reduction → 13 (Showback) + 14 (Forecasting) + 15 (Culture)
```

---

## FinOps Maturity Quick Reference

| Crawl | Walk | Run |
|-------|------|-----|
| CUR enabled | Per-team showback | Chargeback |
| Basic tagging | Rightsizing process | Unit economics KPIs |
| Budget alerts | RI/SP coverage > 70% | Automated tag enforcement |
| Monthly review | Anomaly detection | Cost Champions in every team |
| Untagged < 20% | Dev scale-to-zero | Forecast within 10% |

---

## Tool Ecosystem

| Category | AWS Native | Open Source / Third-Party |
|----------|-----------|--------------------------|
| Cost visibility | Cost Explorer, CUR | Grafana + Athena |
| Anomaly detection | Cost Anomaly Detection | Custom CloudWatch + Lambda |
| Rightsizing | Compute Optimizer | Goldilocks (K8s) |
| K8s cost | — | Kubecost, OpenCost |
| Lambda tuning | — | Lambda Power Tuning (SAR) |
| Forecasting | CE Forecast API | statsmodels, Prophet |
| Tagging governance | Tag Policies, Config | Cloud Custodian |
| Commitment analytics | Savings Plans Console | Spot.io, Apptio Cloudability |

---

## Code Assets in This Section

| Language | Files |
|---------|-------|
| HCL (Terraform) | AWS Budgets, Savings Plans, Spot ASG, S3 Lifecycle, VPC Endpoints, Aurora Serverless, CloudFront, Tag Policy, Budget Forecast |
| Python | Compute Optimizer fetch, RI coverage report, Spot simulation, ECS scale-to-zero, GP2→GP3 migration, Kubecost API, Lambda cost estimator, Athena egress analyser, CUR showback digest, CE forecast, Holt-Winters model, Unit economics, Tag auditor, FinOps KPI publisher |
| YAML | GitHub Actions workflows (cost report, rightsizing, RI review, spot report, Kubecost digest, Lambda power tuning, FinOps KPI), KEDA ScaledObjects, LimitRange, ResourceQuota, Kubecost Helm values, Node Termination Handler, AWS Config rules |
| SQL | Athena CUR queries (team cost, shared cost allocation) |
