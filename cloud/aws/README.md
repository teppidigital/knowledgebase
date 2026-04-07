# AWS Cloud Native Patterns

A comprehensive catalog of **AWS Cloud Native patterns** covering compute, storage, networking, security, data, messaging, observability, and platform engineering.

---

## Pattern Index

| # | Pattern | Category | Services |
|---|---------|----------|---------|
| [01](01-serverless-lambda.md) | Serverless & Lambda | Compute | Lambda, Step Functions, SAM |
| [02](02-ecs-fargate.md) | ECS & Fargate Containers | Compute | ECS, Fargate, ECR, ALB |
| [03](03-eks-kubernetes.md) | EKS — Kubernetes on AWS | Compute | EKS, Karpenter, IRSA, ArgoCD |
| [04](04-vpc-networking.md) | VPC & Networking | Networking | VPC, NAT, PrivateLink, Transit Gateway |
| [05](05-iam-least-privilege.md) | IAM — Least Privilege | Security | IAM, STS, SCPs, ABAC |
| [06](06-api-gateway.md) | API Gateway Patterns | API | API Gateway (HTTP/REST/WebSocket), WAF |
| [07](07-rds-aurora.md) | RDS & Aurora | Data | Aurora PostgreSQL, RDS Proxy, Serverless v2 |
| [08](08-dynamodb.md) | DynamoDB Patterns | Data | DynamoDB, Streams, DAX, GSI |
| [09](09-sqs-sns-eventbridge.md) | SQS, SNS & EventBridge | Messaging | SQS, SNS, EventBridge, Scheduler |
| [10](10-s3-storage.md) | S3 Storage Patterns | Storage | S3, Intelligent-Tiering, Pre-signed URLs |
| [11](11-observability-cloudwatch.md) | Observability — CloudWatch & X-Ray | Operations | CloudWatch, X-Ray, ADOT, Powertools |
| [12](12-multi-account-landing-zone.md) | Multi-Account & Landing Zone | Governance | Organizations, Control Tower, SCPs, SSO |
| [13](13-cloudfront-cdn.md) | CloudFront & CDN | Networking | CloudFront, WAF, CF Functions, Lambda@Edge |
| [14](14-disaster-recovery.md) | Disaster Recovery | Resilience | Aurora Global, DynamoDB Global, Route53, Backup |
| [15](15-cost-optimisation.md) | Cost Optimisation | FinOps | Savings Plans, Spot, Karpenter, Budgets |

---

## Patterns by Category

### Compute

| Pattern | Key decisions |
|---------|--------------|
| [Serverless & Lambda](01-serverless-lambda.md) | Event-driven, 15-min limit, cold start mitigation, Step Functions |
| [ECS & Fargate](02-ecs-fargate.md) | No node management, ECS Exec, Blue/Green deploy, Auto Scaling |
| [EKS — Kubernetes](03-eks-kubernetes.md) | Full K8s API, Karpenter, IRSA, ArgoCD GitOps |

**When to use which compute**:
```
Need full K8s ecosystem (Helm, CRDs, operators)?  → EKS
Need containers without K8s complexity?             → ECS + Fargate
Need event-driven, short-lived functions?           → Lambda
Need high-performance batch workloads?             → ECS + Spot or EKS + Karpenter Spot
```

### Networking

| Pattern | Key decisions |
|---------|--------------|
| [VPC & Networking](04-vpc-networking.md) | 3-tier subnets, NAT per AZ, VPC Endpoints, Transit Gateway |
| [CloudFront & CDN](13-cloudfront-cdn.md) | Cache policies, OAC for S3, Lambda@Edge, WAF at edge |

### Security

| Pattern | Key decisions |
|---------|--------------|
| [IAM — Least Privilege](05-iam-least-privilege.md) | ABAC, Permission Boundaries, SCPs, IRSA, no IAM users |

### API

| Pattern | Key decisions |
|---------|--------------|
| [API Gateway](06-api-gateway.md) | HTTP API (cost) vs REST API (features), JWT authoriser, WAF |

### Data

| Pattern | Key decisions |
|---------|--------------|
| [RDS & Aurora](07-rds-aurora.md) | Aurora Serverless v2, RDS Proxy (Lambda), IAM auth, Multi-AZ |
| [DynamoDB](08-dynamodb.md) | Single-Table Design, GSI access patterns, Streams, TTL, PITR |
| [S3 Storage](10-s3-storage.md) | Pre-signed URLs, lifecycle tiering, event notifications, OAC |

### Messaging

| Pattern | Key decisions |
|---------|--------------|
| [SQS, SNS & EventBridge](09-sqs-sns-eventbridge.md) | SQS for queues, SNS for fanout, EventBridge for routing |

### Operations

| Pattern | Key decisions |
|---------|--------------|
| [Observability](11-observability-cloudwatch.md) | Lambda Powertools, EMF metrics, X-Ray traces, Log Insights |
| [Multi-Account & Landing Zone](12-multi-account-landing-zone.md) | Account per environment, SCPs, Control Tower, SSO |
| [Disaster Recovery](14-disaster-recovery.md) | Aurora Global, DynamoDB Global Tables, Route53 failover |
| [Cost Optimisation](15-cost-optimisation.md) | Savings Plans, Spot + Karpenter, Graviton, Budgets |

---

## AWS Service Quick-Reference

| Service | Purpose | Key patterns used |
|---------|---------|-------------------|
| **Lambda** | Serverless functions | Event-driven, Step Functions orchestration |
| **ECS** | Container orchestration | Fargate tasks, Task roles (IRSA-equivalent) |
| **EKS** | Managed Kubernetes | Karpenter, IRSA, ArgoCD |
| **API Gateway** | Managed API | JWT auth, WAF, throttling |
| **Aurora PostgreSQL** | Relational DB | Serverless v2, Global DB, RDS Proxy |
| **DynamoDB** | NoSQL key-value | Single-table design, Streams, DAX |
| **S3** | Object storage | Pre-signed URLs, lifecycle, event triggers |
| **SQS** | Work queues | DLQ, partial batch failure, idempotency |
| **SNS** | Pub/sub fanout | Filter policies, SQS subscribers |
| **EventBridge** | Event routing | Content-based routing, Scheduler, Pipes |
| **CloudFront** | CDN + edge compute | Cache policies, OAC, Functions, Lambda@Edge |
| **VPC** | Network isolation | 3-tier subnets, VPC Endpoints, Security Groups |
| **IAM** | Identity & access | Least privilege, ABAC, SCPs, IRSA |
| **Organizations** | Multi-account | SCPs, account vending, OU hierarchy |
| **Route53** | DNS + health checks | Failover routing, latency routing |
| **CloudWatch** | Metrics + logs | Alarms, Dashboards, Log Insights |
| **X-Ray** | Distributed tracing | Service map, sampling, annotations |
| **AWS Backup** | Centralised backup | Cross-region copy, PITR |
| **Cost Explorer** | Cost analysis | RI/SP recommendations |
| **AWS Budgets** | Spend alerts | Budget actions, anomaly detection |

---

## Architecture Combinations

### "Serverless API Platform"
Fully managed, no infrastructure to maintain, scales to zero:
```
API Gateway (HTTP API) → Lambda → DynamoDB + SQS → Lambda workers
CloudFront → API Gateway (cache static + proxy API)
```

### "Container Platform (Production)"
Container-first, Kubernetes-native, GitOps:
```
EKS + Karpenter → ECS (or EKS) → Aurora + ElastiCache
ArgoCD (GitOps) + IRSA (per-pod IAM) + External Secrets (Vault/SM)
CloudFront → ALB → EKS
```

### "Event-Driven Microservices"
Decoupled, resilient, independently deployable:
```
API Gateway → Lambda → EventBridge → SNS → SQS (per service) → Lambda
DynamoDB Streams → Lambda → EventBridge (change events)
```

### "Data Lake & Analytics"
Scale-out data processing:
```
S3 (Intelligent-Tiering) → Glue ETL → Athena / Redshift
DynamoDB Streams → Kinesis → S3 → Glue
```

### "Secure Multi-Account Enterprise"
Governance-first, compliant:
```
Organizations + Control Tower + SSO
Security OU: GuardDuty + Security Hub + CloudTrail (immutable)
Network Account: Transit Gateway + VPC sharing
Workload accounts: per-environment, per-product
```

---

## AWS Well-Architected Framework Alignment

| Pillar | Key patterns |
|--------|-------------|
| **Operational Excellence** | Observability (11), GitOps (03), CloudFormation/Terraform IaC |
| **Security** | IAM (05), Multi-Account (12), VPC Endpoints (04), WAF (06, 13) |
| **Reliability** | Disaster Recovery (14), Multi-AZ (07), SQS DLQ (09), Health Checks |
| **Performance Efficiency** | DynamoDB (08), CloudFront (13), EKS Karpenter (03), Lambda (01) |
| **Cost Optimisation** | Cost Patterns (15), Spot (03), S3 Lifecycle (10), Serverless (01) |
| **Sustainability** | Graviton ARM (15), Serverless (01), Intelligent-Tiering (10) |

---

## Related Pattern Libraries

- [System Design Patterns](../../system-design/README.md)
- [Distributed Design Patterns](../../distributed-design-pattern/README.md)
- [DevSecOps Patterns](../../devsecops/README.md)
