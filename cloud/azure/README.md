# Cloud Native — Azure

A comprehensive reference for building cloud-native workloads on Microsoft Azure. Each document covers a core domain with context, trade-offs, architecture diagrams (Mermaid), and production-grade code samples in TypeScript and Bicep.

---

## Pattern Index

| # | File | Pattern | Key Azure Services |
|---|------|---------|-------------------|
| 01 | [01-azure-functions.md](01-azure-functions.md) | Azure Functions & Durable Functions | Functions, Durable Functions, Service Bus, Key Vault |
| 02 | [02-container-apps.md](02-container-apps.md) | Azure Container Apps | ACA, KEDA, Dapr, ACR |
| 03 | [03-aks-kubernetes.md](03-aks-kubernetes.md) | AKS — Kubernetes on Azure | AKS, KEDA, Flux, Workload Identity, Cilium |
| 04 | [04-vnet-networking.md](04-vnet-networking.md) | Virtual Network & Hub-Spoke Networking | VNet, NSG, Azure Firewall, Private Endpoints, Private DNS |
| 05 | [05-entra-id-rbac.md](05-entra-id-rbac.md) | Entra ID & Azure RBAC | Managed Identity, Workload Identity Federation, PIM, Conditional Access |
| 06 | [06-api-management.md](06-api-management.md) | Azure API Management (APIM) | APIM, JWT validation, Rate Limiting, Developer Portal |
| 07 | [07-azure-sql-cosmos.md](07-azure-sql-cosmos.md) | Azure SQL & Cosmos DB | SQL Database, Elastic Pool, Cosmos DB, Change Feed |
| 08 | [08-service-bus-event-grid.md](08-service-bus-event-grid.md) | Service Bus, Event Grid & Event Hubs | Service Bus, Event Grid, Event Hubs, KEDA |
| 09 | [09-blob-storage.md](09-blob-storage.md) | Blob Storage & ADLS Gen2 | Storage Account, Lifecycle Management, Private Endpoints, Event Grid |
| 10 | [10-azure-monitor.md](10-azure-monitor.md) | Azure Monitor, Application Insights & Log Analytics | Monitor, App Insights, Log Analytics, KQL, Alerts |
| 11 | [11-azure-devops-pipelines.md](11-azure-devops-pipelines.md) | Azure DevOps & Pipelines | Azure Pipelines, OIDC, Environments, GitHub Actions |
| 12 | [12-landing-zone.md](12-landing-zone.md) | Landing Zone & Management Groups | Management Groups, Azure Policy, Subscription Vending |
| 13 | [13-front-door-cdn.md](13-front-door-cdn.md) | Azure Front Door & CDN | Front Door Premium, WAF, Private Link Origins |
| 14 | [14-disaster-recovery.md](14-disaster-recovery.md) | Disaster Recovery & Business Continuity | ASR, SQL Failover Groups, Cosmos DB Multi-Region, Traffic Manager |
| 15 | [15-cost-management.md](15-cost-management.md) | Cost Management & Optimisation | Cost Management, Budgets, Reservations, Savings Plans, Spot VMs |
| 16 | [16-key-vault.md](16-key-vault.md) | Azure Key Vault | Key Vault, Secrets, Keys, Certificates, CMK, Rotation |
| 17 | [17-cache-for-redis.md](17-cache-for-redis.md) | Azure Cache for Redis | Redis, Cache-Aside, Rate Limiting, Sessions, Distributed Lock |
| 18 | [18-durable-functions.md](18-durable-functions.md) | Azure Durable Functions | Durable Functions, Orchestrator, Entity, Fan-out, Human Interaction |
| 19 | [19-security-best-practices.md](19-security-best-practices.md) | Security Best Practices | Managed Identity, Conditional Access, PIM, Key Vault, Private Endpoints, Defender for Cloud, Sentinel, WAF |
| 20 | [20-azure-openai-ai-foundry.md](20-azure-openai-ai-foundry.md) | Azure OpenAI & AI Foundry | Azure OpenAI, GPT-4o, AI Foundry, AI Search (RAG), Semantic Kernel, Content Safety |
| 21 | [21-azure-arc.md](21-azure-arc.md) | Azure Arc — Hybrid & Multi-Cloud | Azure Arc, Arc-enabled K8s, Arc-enabled Servers, GitOps, Defender for Cloud, Azure Policy |

---

## Patterns by Category

### Compute

| Pattern | Recommended when |
|---------|-----------------|
| [Azure Functions](01-azure-functions.md) | Event-driven, short-lived workloads; Durable workflows; serverless-first preference |
| [Azure Container Apps](02-container-apps.md) | Microservices with KEDA scaling + Dapr; scale-to-zero; no K8s management overhead |
| [AKS — Kubernetes](03-aks-kubernetes.md) | Full Kubernetes API access; custom operators; GPU; complex scheduling requirements |

### Networking

| Pattern | Recommended when |
|---------|-----------------|
| [VNet & Hub-Spoke](04-vnet-networking.md) | Any workload requiring private networking, Private Endpoints, or on-premises connectivity |
| [Front Door & CDN](13-front-door-cdn.md) | Global HTTP traffic entry; WAF; static asset caching; multi-region failover |

### Security & Identity

| Pattern | Recommended when |
|---------|-----------------|
| [Entra ID & RBAC](05-entra-id-rbac.md) | All workloads — Managed Identity, Workload Identity Federation, PIM |
| [API Management](06-api-management.md) | Publishing APIs to external consumers; centralised auth/rate-limit/CORS || [Key Vault](16-key-vault.md) | All workloads — secrets, keys, certificates; CMK encryption; rotation automation |
### Data & Storage

| Pattern | Recommended when |
|---------|-----------------|
| [Azure SQL & Cosmos DB](07-azure-sql-cosmos.md) | Relational (SQL DB) or globally distributed low-latency NoSQL (Cosmos DB) |
| [Blob Storage & ADLS Gen2](09-blob-storage.md) | Object storage, data lake, analytics landing zone, WORM compliance || [Azure Cache for Redis](17-cache-for-redis.md) | Read-heavy workloads; session storage; rate limiting; leaderboards; distributed locks |
### Messaging & Eventing

| Pattern | Recommended when |
|---------|-----------------|
| [Service Bus, Event Grid & Event Hubs](08-service-bus-event-grid.md) | Decoupled microservices (SB), reactive resource events (EG), streaming ingestion (EH) || [Durable Functions](18-durable-functions.md) | Long-running workflows; fan-out/fan-in; human approval; stateful actor patterns |
### Observability & Operations

| Pattern | Recommended when |
|---------|-----------------|
| [Azure Monitor & Application Insights](10-azure-monitor.md) | All workloads — unified logs, traces, metrics, alerting |
| [Azure DevOps & Pipelines](11-azure-devops-pipelines.md) | CI/CD automation with approval gates, OIDC auth, and IaC deployment |

### Platform & Governance

| Pattern | Recommended when |
|---------|-----------------|
| [Landing Zone & Management Groups](12-landing-zone.md) | Multi-subscription enterprise Azure — policy, RBAC, subscription vending |
| [Disaster Recovery](14-disaster-recovery.md) | Workloads with defined RTO/RPO SLAs; regulated industries |
| [Cost Management](15-cost-management.md) | Controlling cloud spend; FinOps programme; multi-team chargeback || [Azure Arc — Hybrid & Multi-Cloud](21-azure-arc.md) | Multi-cloud K8s governance; on-prem SQL MI; unified Defender posture |

### AI & Machine Learning

| Pattern | Recommended when |
|---------|------------------|
| [Azure OpenAI & AI Foundry](20-azure-openai-ai-foundry.md) | LLM-powered features with data residency; RAG via AI Search; Semantic Kernel orchestration |
---

## Azure vs AWS Service Equivalence

| Domain | Azure | AWS |
|--------|-------|-----|
| Serverless functions | Azure Functions | AWS Lambda |
| Managed containers (serverless) | Azure Container Apps | AWS App Runner / Fargate |
| Managed Kubernetes | AKS | EKS |
| Object storage | Blob Storage | S3 |
| Data lake | ADLS Gen2 | S3 + Lake Formation |
| Relational DB | Azure SQL / SQL MI | RDS / Aurora |
| Global NoSQL | Cosmos DB | DynamoDB |
| Queue messaging | Service Bus Queues | SQS |
| Pub/sub topics | Service Bus Topics | SNS |
| Event streaming | Event Hubs | Kinesis Data Streams |
| Event routing | Event Grid | EventBridge |
| API gateway | API Management | API Gateway |
| Global CDN + load balancer | Azure Front Door | CloudFront + ALB |
| WAF | Azure WAF (on Front Door / App GW) | AWS WAF |
| Identity / IAM | Entra ID + Azure RBAC | IAM + STS |
| Workload identity (K8s) | Workload Identity Federation | EKS IRSA |
| Secrets management | Key Vault | Secrets Manager / SSM Parameter Store |
| In-memory cache | Azure Cache for Redis | ElastiCache (Redis / Memcached) |
| Workflow orchestration | Durable Functions / Logic Apps | Step Functions |
| Managed DNS (private) | Private DNS Zones | Route 53 Private Hosted Zones |
| Network firewall | Azure Firewall | AWS Network Firewall |
| Hub-spoke networking | Hub VNet + Peering | AWS Transit Gateway |
| Private connectivity to PaaS | Private Endpoints | VPC Endpoints (Interface) |
| VM disaster recovery | Azure Site Recovery | AWS DRS / CloudEndure |
| Monitoring / APM | Azure Monitor + Application Insights | CloudWatch + X-Ray |
| Log aggregation | Log Analytics (KQL) | CloudWatch Logs Insights |
| IaC (native) | Bicep | CloudFormation |
| IaC (community) | Terraform | Terraform |
| CI/CD | Azure Pipelines / GitHub Actions | CodePipeline / GitHub Actions |
| Policy enforcement | Azure Policy | AWS Config + Service Control Policies |
| Multi-account / subscription governance | Management Groups + Azure Policy | AWS Organizations + SCPs |
| Landing zone | Azure Landing Zone (CAF) | AWS Control Tower |
| Cost management | Azure Cost Management + Billing | AWS Cost Explorer + Budgets |
| Commitment discounts | Reservations + Savings Plans | Reserved Instances + Savings Plans |

---

## Azure Well-Architected Framework Alignment

The Azure Well-Architected Framework defines **six pillars**:

| Pillar | Key Patterns in this Section |
|--------|------------------------------|
| **Reliability** | AKS multi-AZ node pools (03), SQL Failover Groups (14), Cosmos DB multi-region (07), Front Door health probes (13) |
| **Security** | Entra ID + Managed Identity (05), Private Endpoints everywhere (04), WAF on Front Door (13), APIM JWT validation (06) |
| **Cost Optimisation** | Reservations + Savings Plans (15), Spot Node Pools (03), Scale-to-Zero ACA (02), Storage Lifecycle (09) |
| **Operational Excellence** | Azure Pipelines OIDC CI/CD (11), Landing Zone Policy (12), Azure Monitor + Alerts (10), GitOps Flux (03) |
| **Performance Efficiency** | KEDA autoscaling (02, 03), Cosmos DB autoscale RU/s (07), Front Door edge caching (13), Event Hubs partitioning (08) |
| **Sustainability** | Scale-to-zero (01, 02), Spot evictable compute (15), Storage tier transitions (09), Reserved capacity (15) |

---

## Key Conventions Used in This Section

| Convention | Rationale |
|-----------|-----------|
| `DefaultAzureCredential` throughout | Works identically in dev (az login), AKS (Workload Identity), ACA/Functions (Managed Identity) — no code changes across environments |
| Bicep (not ARM JSON) | Readable, type-safe, native Azure IaC — Bicep compiles to ARM; preferred over raw JSON templates |
| `disableLocalAuth: true` on Service Bus / Cosmos DB | Enforce RBAC-only access — SAS keys and primary keys are permanently disabled |
| `publicNetworkAccess: 'Disabled'` on PaaS | All PaaS services accessed via Private Endpoints only — no public internet exposure |
| OIDC for CI/CD | GitHub Actions and Azure Pipelines authenticate to Azure with federated identity — zero stored service principal secrets |
| Zone-redundant SKUs in prod | AKS node pools, Azure SQL, Service Bus Premium, Storage GZRS — HA without manual failover |
| Managed rules before custom rules in WAF | Microsoft DRS 2.1 covers OWASP top 10; add custom rules only for application-specific requirements |

---

## Related Sections

- [Cloud Native AWS](../aws/README.md) — equivalent patterns on AWS
- [System Design Patterns](../../system-design/) — architecture patterns applicable to both clouds
- [Distributed Design Patterns](../../distributed-design-pattern/) — messaging, saga, CQRS patterns
- [DevSecOps](../../devsecops/) — security pipeline and supply chain patterns
