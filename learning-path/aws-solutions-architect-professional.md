# AWS Certified Solutions Architect – Professional (SAP-C02)
## Expert Learning Path

---

## Exam at a Glance

| Property | Detail |
|----------|--------|
| Exam code | SAP-C02 |
| Duration | 180 minutes |
| Questions | 75 (multiple choice + multiple response) |
| Cost | USD 300 |
| Passing score | 750 / 1000 |
| Validity | 3 years |
| Prerequisite | No formal prerequisite, but associate-level experience expected |
| Recommended experience | 2+ years architecting on AWS at scale |

### Exam Domains and Weighting

| Domain | Title | Weight |
|--------|-------|--------|
| 1 | Design Solutions for Organisational Complexity | 26% |
| 2 | Design for New Solutions | 29% |
| 3 | Continuous Improvement for Existing Solutions | 25% |
| 4 | Accelerate Workload Migration and Modernisation | 20% |

> **Total study time estimate:** 14–18 weeks at 8–12 hours/week (faster if you hold SAA-C03 already).

---

## Prerequisites Self-Assessment

Complete this before starting. Score yourself 1–5 on each area.

| Area | Topics to Test Yourself On | Target Score |
|------|----------------------------|--------------|
| AWS core services | EC2, S3, VPC, IAM, RDS, Lambda basics | 4+ |
| Networking | Subnets, route tables, security groups, NACLs | 4+ |
| IAM | Policies, roles, trust relationships | 3+ |
| High availability | Multi-AZ, load balancing, Auto Scaling | 3+ |
| Cost awareness | Reserved vs On-Demand vs Spot | 3+ |

If any area scores below 3, complete the [AWS Cloud Practitioner](https://aws.amazon.com/certification/certified-cloud-practitioner/) and [SAA-C03](https://aws.amazon.com/certification/certified-solutions-architect-associate/) material first.

---

## Phase Overview

```
Phase 0 (Week 1)        → Baseline and exam orientation
Phase 1 (Weeks 2–4)     → Domain 1: Organisational Complexity
Phase 2 (Weeks 5–7)     → Domain 2: New Solutions
Phase 3 (Weeks 8–10)    → Domain 3: Continuous Improvement
Phase 4 (Weeks 11–12)   → Domain 4: Migration and Modernisation
Phase 5 (Weeks 13–14)   → Consolidation and weak-area remediation
Phase 6 (Weeks 15–16)   → Full mock exams and exam-day readiness
```

---

## Phase 0 — Baseline and Exam Orientation (Week 1)

### Goal
Understand the exam format, identify your starting knowledge gaps, and build your study system.

### Actions

- [ ] Download the official [SAP-C02 Exam Guide](https://aws.amazon.com/certification/certified-solutions-architect-professional/) and read it fully — one sitting.
- [ ] Take the **AWS Certification Official Practice Question Set** (free on AWS Skill Builder) — 20 questions. Record your score per domain.
- [ ] Map your score to the domain weightings table above — the domain where you score lowest relative to its weight is your highest-priority gap.
- [ ] Create a study log (spreadsheet or markdown file) tracking: date, topic, time spent, practice score.
- [ ] Sign up for [AWS Skill Builder](https://skillbuilder.aws) (free tier is sufficient to start).
- [ ] Set your target exam date: book it now (12–16 weeks out). An anchored date creates accountability.

### Validation
You can describe every domain in two sentences and rank them from strongest to weakest based on your practice results.

---

## Phase 1 — Domain 1: Design for Organisational Complexity (Weeks 2–4)

**Exam weight: 26% — approximately 19–20 questions**

This domain is where most candidates lose points. It requires deep understanding of multi-account governance, hybrid connectivity, and identity federation — topics rarely covered at associate level.

### Week 2 — Multi-Account Governance

**Core services:** AWS Organizations, Control Tower, Service Control Policies (SCPs), AWS Config, CloudTrail, Security Hub, GuardDuty

**Topics to master:**

- [ ] AWS Organizations: OUs, consolidated billing, account vending patterns
- [ ] Control Tower: Landing Zone setup, account factory, guardrails (preventive vs detective)
- [ ] SCPs: inheritance rules, deny vs allow strategies, `NotAction` patterns
- [ ] Delegated administrator pattern — when to use it, how it differs from root account management
- [ ] AWS Config aggregators — cross-account compliance views
- [ ] CloudTrail: organisation trail vs per-account trail, S3 log aggregation to security account

**Knowledge portal cross-reference:**
- [`cloud/aws/12-multi-account-landing-zone.md`](../cloud/aws/12-multi-account-landing-zone.md) — read in full
- [`cloud/aws/05-iam-least-privilege.md`](../cloud/aws/05-iam-least-privilege.md) — SCPs and permission boundaries section
- [`devsecops/01-shift-left-security.md`](../devsecops/01-shift-left-security.md) — governance principles

**Practice scenario:** Design a 15-account AWS organisation for a financial services firm. Define the OU structure, write 3 meaningful SCPs, and describe how you would enforce tagging compliance across all accounts.

### Week 3 — Hybrid Connectivity

**Core services:** AWS Direct Connect, Site-to-Site VPN, Transit Gateway, Route 53 Resolver, PrivateLink, VPC Peering

**Topics to master:**

- [ ] Direct Connect: dedicated vs hosted connection, virtual interfaces (private, public, transit), MACsec
- [ ] Direct Connect Gateway: ability to connect to multiple regions and VPCs
- [ ] Transit Gateway: attachments (VPC, VPN, DX, peering), route tables, blackhole routes, multicast
- [ ] Transit Gateway Network Manager — global network visibility
- [ ] When to use VPC Peering vs Transit Gateway (cost, routing, transitive routing limitations)
- [ ] Route 53 Resolver: Inbound/Outbound endpoints for on-premises DNS resolution
- [ ] VPN redundancy: dual-tunnel, BGP failover, accelerated VPN (Global Accelerator path)
- [ ] PrivateLink: endpoint services, interface endpoints, Gateway endpoints (S3/DynamoDB)

**Knowledge portal cross-reference:**
- [`cloud/aws/04-vpc-networking.md`](../cloud/aws/04-vpc-networking.md) — full read, focus on Transit Gateway section

**Practice scenario:** Design hybrid connectivity for an enterprise with 3 on-premises datacentres (EU, US, APAC), requiring < 10 ms latency to nearest AWS region, HA for network paths, and private DNS resolution between on-premises and AWS.

### Week 4 — Identity Federation and Cross-Account Access

**Core services:** IAM Identity Center (SSO), AWS STS, Cognito, SAML 2.0, OIDC, Resource Access Manager (RAM)

**Topics to master:**

- [ ] IAM Identity Center: permission sets, assignment model, SCIM provisioning, external IdP integration
- [ ] SAML 2.0 federation: SP-initiated vs IdP-initiated flow, role mapping
- [ ] OIDC federation: web identity federation, GitHub Actions OIDC, EKS IRSA deep dive
- [ ] STS: `AssumeRole`, `AssumeRoleWithSAML`, `AssumeRoleWithWebIdentity` — when to use each
- [ ] Cross-account role chaining — trust policies, `sts:ExternalId`
- [ ] Resource Access Manager: sharing VPCs, subnets, Transit Gateway, Route 53 Resolver rules
- [ ] Cognito: User Pools vs Identity Pools, federation into identity pools, fine-grained DynamoDB access

**Knowledge portal cross-reference:**
- [`cloud/aws/05-iam-least-privilege.md`](../cloud/aws/05-iam-least-privilege.md) — ABAC and IRSA sections
- [`security/`](../security/) — identity architecture patterns

**Phase 1 Deliverable:** Architecture diagram for a fully federated multi-account AWS environment: external IdP → IAM Identity Center → permission sets → cross-account roles → resource access. Written as a 1-page design doc.

**Domain 1 Validation Questions:**
- What is the difference between a preventive and a detective guardrail in Control Tower?
- A SCP `Deny` at the root OU — can the management account override it?
- Why can't you use VPC Peering for transitive routing? What must you use instead?
- How does `sts:ExternalId` prevent confused deputy attacks in cross-account scenarios?
- What is the difference between a Direct Connect private VIF and a transit VIF?

---

## Phase 2 — Domain 2: Design for New Solutions (Weeks 5–7)

**Exam weight: 29% — approximately 21–22 questions**

The heaviest domain. Covers the full breadth of AWS architectural decision-making: compute, data, messaging, resilience, and security for greenfield systems.

### Week 5 — Resilience and Multi-Region Architecture

**Core services:** Route 53, Global Accelerator, Aurora Global Database, DynamoDB Global Tables, S3 Replication, CloudFront

**Topics to master:**

- [ ] Disaster recovery tiers: Backup & Restore → Pilot Light → Warm Standby → Multi-Site Active/Active — RTO/RPO for each
- [ ] Route 53 health checks, routing policies (latency, geolocation, geoproximity, weighted, failover, multivalue)
- [ ] Global Accelerator vs CloudFront — when to use each, anycast IPs, health-aware routing
- [ ] Aurora Global Database: replication lag (< 1 s), promotion process, write forwarding
- [ ] DynamoDB Global Tables: eventual vs strong consistency, conflict resolution, replication lag
- [ ] S3 Cross-Region Replication (CRR): replication time control, bi-directional replication
- [ ] RTO vs RPO trade-offs in multi-region design — cost implications at each tier

**Knowledge portal cross-reference:**
- [`cloud/aws/14-disaster-recovery.md`](../cloud/aws/14-disaster-recovery.md) — full read
- [`cloud/aws/13-cloudfront-cdn.md`](../cloud/aws/13-cloudfront-cdn.md) — Global Accelerator comparison section
- [`production-hardening/`](../production-hardening/) — resilience patterns

**Practice scenario:** Design a multi-region active-passive architecture for a fintech application requiring RPO < 15 minutes and RTO < 30 minutes. Cost must be < 40% of a full active-active deployment.

### Week 6 — Compute, Serverless, and Event-Driven Architecture

**Core services:** Lambda, Step Functions, ECS, EKS, EventBridge, SQS, SNS, Kinesis, Fargate, EC2 Auto Scaling

**Topics to master:**

- [ ] Lambda: concurrency (reserved vs provisioned), execution environments, layers, SnapStart, response streaming
- [ ] Lambda destinations, DLQ patterns, error handling in async invocations
- [ ] Step Functions: Standard vs Express workflows, Map state, error handling, SDK integrations
- [ ] ECS vs EKS decision criteria — when Fargate, when EC2 backing nodes
- [ ] Kinesis Data Streams vs Kinesis Firehose vs Managed Kafka (MSK) — throughput, ordering, consumers
- [ ] SQS: FIFO vs Standard, message deduplication, visibility timeout, long polling, DLQ
- [ ] EventBridge: event buses, rules, pipes, scheduler, schema registry
- [ ] Fan-out pattern: SNS → SQS → Lambda at scale
- [ ] SAGA pattern for distributed transactions across microservices

**Knowledge portal cross-reference:**
- [`cloud/aws/01-serverless-lambda.md`](../cloud/aws/01-serverless-lambda.md) — full read
- [`cloud/aws/02-ecs-fargate.md`](../cloud/aws/02-ecs-fargate.md) — full read
- [`cloud/aws/03-eks-kubernetes.md`](../cloud/aws/03-eks-kubernetes.md) — full read
- [`cloud/aws/09-sqs-sns-eventbridge.md`](../cloud/aws/09-sqs-sns-eventbridge.md) — full read
- [`distributed-design-pattern/`](../distributed-design-pattern/) — SAGA, event sourcing, CQRS
- [`api-design/04-asyncapi-event-driven.md`](../api-design/04-asyncapi-event-driven.md)

### Week 7 — Data Architecture and Storage

**Core services:** RDS, Aurora, DynamoDB, ElastiCache, Redshift, S3, EFS, FSx, Glue, Athena, Lake Formation

**Topics to master:**

- [ ] RDS: Multi-AZ (synchronous) vs Read Replicas (asynchronous), cross-region read replicas, RDS Proxy
- [ ] Aurora: cluster storage, Aurora Serverless v2 scaling, Global Database, Parallel Query, Babelfish
- [ ] DynamoDB: partition key design, GSI vs LSI, DAX, on-demand vs provisioned, Streams + Lambda patterns
- [ ] ElastiCache: Redis vs Memcached, cluster mode, cross-AZ replication, lazy loading vs write-through
- [ ] Redshift: node types (RA3 vs DC2), Spectrum for S3 queries, Concurrency Scaling, data sharing
- [ ] S3: storage classes, Intelligent-Tiering, S3 Object Lambda, byte-range fetches, multipart upload
- [ ] Data lake pattern: S3 + Glue + Athena + Lake Formation permissions
- [ ] FSx: for Windows (SMB/AD integration) vs for Lustre (HPC/ML) vs for NetApp ONTAP
- [ ] EFS: performance modes, throughput modes, access points, cross-account access

**Knowledge portal cross-reference:**
- [`cloud/aws/07-rds-aurora.md`](../cloud/aws/07-rds-aurora.md) — full read
- [`cloud/aws/08-dynamodb.md`](../cloud/aws/08-dynamodb.md) — full read
- [`cloud/aws/10-s3-storage.md`](../cloud/aws/10-s3-storage.md) — full read
- [`data-solutions/04-data-lakehouse.md`](../data-solutions/04-data-lakehouse.md)
- [`data-solutions/05-data-modelling-warehouse.md`](../data-solutions/05-data-modelling-warehouse.md)
- [`data-solutions/07-caching-strategies.md`](../data-solutions/07-caching-strategies.md)

**Phase 2 Deliverable:** Design a complete data architecture for a SaaS analytics platform: OLTP layer (DynamoDB), analytical layer (Redshift + S3 data lake), real-time stream processing (Kinesis), and caching (ElastiCache). Include justification for every service choice.

**Domain 2 Validation Questions:**
- What is the replication lag of Aurora Global Database, and what does write forwarding do?
- A Lambda function needs to process exactly-once from an SQS FIFO queue — what are the configuration requirements?
- When would you choose DynamoDB Global Tables over Aurora Global Database?
- What is the maximum throughput of a single Kinesis Data Stream shard, and how do you scale beyond it?
- What is S3 Object Lambda and what problem does it solve?

---

## Phase 3 — Domain 3: Continuous Improvement for Existing Solutions (Weeks 8–10)

**Exam weight: 25% — approximately 18–19 questions**

This domain is about taking existing workloads and improving them across four axes: performance, reliability, security, and cost. Expect scenarios where you must identify the *single most impactful* improvement.

### Week 8 — Performance Optimisation

**Core services:** ElastiCache, CloudFront, Application Auto Scaling, EC2 Auto Scaling, Compute Optimizer, Trusted Advisor

**Topics to master:**

- [ ] Caching hierarchy: CloudFront (edge) → ElastiCache (application) → DAX (DynamoDB) — when each applies
- [ ] Auto Scaling: target tracking vs step scaling vs scheduled — choose based on load pattern
- [ ] EC2 Auto Scaling: predictive scaling, lifecycle hooks, warm pools
- [ ] Application Auto Scaling: Fargate tasks, DynamoDB tables, Aurora replicas
- [ ] Read replicas as a scaling strategy for read-heavy RDS workloads
- [ ] Kinesis Data Streams: re-sharding (split vs merge), enhanced fan-out consumers
- [ ] Transfer Acceleration for S3 — high-latency upload paths
- [ ] Compute Optimizer: right-sizing recommendations for EC2, Lambda, ECS, EBS

**Knowledge portal cross-reference:**
- [`cloud/aws/13-cloudfront-cdn.md`](../cloud/aws/13-cloudfront-cdn.md) — caching and performance
- [`cloud/aws/11-observability-cloudwatch.md`](../cloud/aws/11-observability-cloudwatch.md) — metrics-driven scaling
- [`data-solutions/07-caching-strategies.md`](../data-solutions/07-caching-strategies.md)

### Week 9 — Security Posture Improvement

**Core services:** Security Hub, GuardDuty, Macie, Inspector, WAF, Shield, Secrets Manager, KMS, CloudTrail, Config

**Topics to master:**

- [ ] KMS: CMK types (AWS managed, customer managed, custom key store), key policies, grants, envelope encryption
- [ ] Secrets Manager: rotation strategies, Lambda rotation function, cross-account secret sharing
- [ ] GuardDuty: threat types (Trojan, CryptoCurrency, data exfiltration), suppression rules, S3 protection
- [ ] Macie: sensitive data discovery in S3, automated findings
- [ ] Inspector v2: EC2 and container (ECR) vulnerability scanning, SBOM support
- [ ] WAF: rule groups, rate limiting, managed rule groups, IP reputation lists, Bot Control
- [ ] Shield Advanced: proactive engagement, cost protection, DDoS response team (DRT) access
- [ ] VPC Flow Logs: analysing with Athena, anomaly detection patterns
- [ ] Network Firewall vs Security Groups vs NACLs — layered defence model
- [ ] Security Hub: CSPM, benchmark findings (CIS, PCI-DSS, AWS Foundational)

**Knowledge portal cross-reference:**
- [`devsecops/`](../devsecops/) — all documents relevant here; prioritise `05-secret-management.md`, `07-container-security.md`
- [`security/`](../security/)
- [`cloud/aws/05-iam-least-privilege.md`](../cloud/aws/05-iam-least-privilege.md)

### Week 10 — Cost Optimisation and Operational Excellence

**Core services:** Cost Explorer, Budgets, Savings Plans, Spot Instances, Compute Optimizer, Trusted Advisor, Systems Manager

**Topics to master:**

- [ ] Savings Plans: Compute vs EC2 vs SageMaker — flexibility trade-offs, commitment terms
- [ ] Spot Instances: interruption handling, Spot Fleet strategies (diversified, capacity-optimised), spot blocks
- [ ] Reserved Instances: standard vs convertible, regional vs AZ-scoped
- [ ] S3 cost optimisation: lifecycle policies, Intelligent-Tiering, requester-pays
- [ ] Data transfer costs: cross-AZ vs cross-region vs internet egress — design to minimise
- [ ] Graviton3 adoption: 40% better price/performance for most workloads
- [ ] Systems Manager: Patch Manager, Parameter Store (vs Secrets Manager cost model), Session Manager
- [ ] AWS Well-Architected Tool: conducting a review, pillar scores, high-risk findings
- [ ] Cost allocation tags: enforcing via SCPs, Budgets per OU, Cost Explorer segmentation
- [ ] Trusted Advisor high-value checks: underutilised EC2, idle RDS, unassociated EIPs

**Knowledge portal cross-reference:**
- [`cloud/aws/15-cost-optimisation.md`](../cloud/aws/15-cost-optimisation.md) — full read
- [`cloud/aws/16-well-architected-framework.md`](../cloud/aws/16-well-architected-framework.md) — full read
- [`finops/`](../finops/)

**Phase 3 Deliverable:** Conduct a mock Well-Architected Review on a system you know (or a fictional one). Score it across all five pillars, identify the top 3 high-risk findings, and write an improvement plan for each — specifying the AWS service change, estimated cost delta, and expected reliability improvement.

**Domain 3 Validation Questions:**
- An RDS Aurora cluster is CPU-bound at 85% during peak. What are three specific, ordered steps you take before vertical scaling?
- What is the difference between a Compute Savings Plan and an EC2 Instance Savings Plan?
- When should you use Network Firewall vs WAF vs Security Groups?
- How does envelope encryption with KMS work, and why is it used for large object encryption in S3?
- A Spot Instance is interrupted mid-job. What mechanism ensures the job is retried?

---

## Phase 4 — Domain 4: Migration and Modernisation (Weeks 11–12)

**Exam weight: 20% — approximately 15 questions**

The most scenario-based domain. AWS migration questions have a "right framework" (the 7 Rs), and knowing which R applies in which scenario is the differentiator.

### Week 11 — Migration Strategies and Tools

**Core services:** AWS Migration Hub, Application Migration Service (MGN), DMS, SCT, DataSync, Snowball/Snowmobile, Transfer Family

**Topics to master:**

- [ ] The 7 R migration strategies: Retire, Retain, Rehost, Relocate, Replatform, Repurchase, Refactor — when to apply each
- [ ] Application Migration Service (MGN): agentless vs agent-based, cutover, test migration
- [ ] Database Migration Service (DMS): homogeneous vs heterogeneous migration, Schema Conversion Tool (SCT)
- [ ] DMS: replication instance sizing, task types (full load, CDC, full load + CDC), LOB handling
- [ ] DataSync: on-premises NFS/SMB → S3/EFS/FSx, scheduling, bandwidth throttling
- [ ] Snowball Edge: Storage Optimised vs Compute Optimised, clustering, edge computing use cases
- [ ] Snowmobile: when it makes sense (> 10 PB, physical security requirements)
- [ ] AWS Transfer Family: SFTP/FTPS/FTP → S3 or EFS, identity provider integration
- [ ] Migration Hub: tracking migrations across tools, network visualisation

**Knowledge portal cross-reference:**
- [`devops/04-deployment-strategies.md`](../devops/04-deployment-strategies.md) — lift-shift, blue/green, canary during migration

### Week 12 — Application Modernisation Patterns

**Core services:** ECS, EKS, Lambda, API Gateway, App Runner, Elastic Beanstalk, Amplify

**Topics to master:**

- [ ] Monolith-to-microservices decomposition: Strangler Fig pattern, anti-corruption layer
- [ ] Container adoption path: VMs → ECS/Fargate → EKS — decision criteria
- [ ] Serverless refactoring: identify functions that map to Lambda (short-lived, event-triggered, stateless)
- [ ] App Runner: fully managed container deployment — when to use vs ECS Fargate
- [ ] Elastic Beanstalk: as a replatforming step, platform versions, deployment modes
- [ ] Database modernisation: Oracle/SQL Server → Aurora (using DMS + SCT), PostgreSQL compatibility
- [ ] Decoupling tightly coupled synchronous systems: introduce SQS/EventBridge as an intermediary
- [ ] Read replica promotion as a zero-downtime migration handover strategy

**Knowledge portal cross-reference:**
- [`distributed-design-pattern/`](../distributed-design-pattern/) — Strangler Fig, SAGA, CQRS
- [`cloud/aws/03-eks-kubernetes.md`](../cloud/aws/03-eks-kubernetes.md)
- [`cloud/aws/02-ecs-fargate.md`](../cloud/aws/02-ecs-fargate.md)
- [`api-design/05-api-versioning.md`](../api-design/05-api-versioning.md) — managing API changes during modernisation

**Phase 4 Deliverable:** Design a migration plan for a monolithic Java application running on a 3-tier architecture in an on-premises datacentre. The plan must cover: discovery, database migration (Oracle → Aurora), application replatforming (Tomcat → ECS Fargate), network migration (on-premises → VPC), and a rollback strategy.

**Domain 4 Validation Questions:**
- What is the difference between Rehost and Relocate?
- You have 4 PB of data to migrate. The customer has a 1 Gbps internet connection. Should you use DataSync, Snowball, or Direct Connect? Walk through the calculation.
- What does DMS CDC mode do, and when is it necessary?
- A legacy application cannot be modified. Which R strategy applies, and what constraints does it impose?
- What is the Strangler Fig pattern and why is it preferred over a big-bang rewrite?

---

## Phase 5 — Consolidation and Weak-Area Remediation (Weeks 13–14)

### Week 13 — Targeted Gap Remediation

- [ ] Review your study log — which topics generated the most wrong answers in practice questions?
- [ ] Score yourself on each domain using a 65-question practice exam (Tutorials Dojo or Whizlabs).
- [ ] For every wrong answer, do not just memorise the answer — trace it back to the service documentation and understand the *why*.
- [ ] Focus extra time on the two lowest-scoring domains.

**High-failure-rate topics (based on historical exam reports):**
- Direct Connect Gateway vs Transit Gateway vs VGW routing
- SCP inheritance and management account exemptions
- Aurora Global Database promotion sequence
- Which Savings Plan type to recommend in a multi-region, multi-instance scenario
- Lambda concurrency: reserved vs provisioned — effect on throttling and cold starts
- FSx for Lustre integration modes with S3

### Week 14 — Architecture White Papers and Service Limits

- [ ] Read the [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/) — all six pillars (Operational Excellence, Security, Reliability, Performance Efficiency, Cost Optimisation, Sustainability).
- [ ] Read the [AWS Security Reference Architecture](https://docs.aws.amazon.com/prescriptive-guidance/latest/security-reference-architecture/welcome.html) (SRA) — the multi-account security account model.
- [ ] Study AWS service limits for exam-relevant services (SQS batch size, Lambda timeout, API Gateway payload, DMS replication instance types).
- [ ] Review the [AWS Prescriptive Guidance](https://aws.amazon.com/prescriptive-guidance/) patterns for migration and modernisation.

**Knowledge portal cross-reference:**
- [`cloud/aws/16-well-architected-framework.md`](../cloud/aws/16-well-architected-framework.md) — map each pillar to exam questions you got wrong
- [`cloud/aws/11-observability-cloudwatch.md`](../cloud/aws/11-observability-cloudwatch.md) — operational excellence scenarios

---

## Phase 6 — Full Mock Exams and Exam-Day Readiness (Weeks 15–16)

### Week 15 — Timed Full Mock Exams

Complete the following in order. Do each under timed conditions (180 minutes, no pause).

| Exam | Source | Target Score |
|------|--------|--------------|
| Practice Exam 1 | [Tutorials Dojo SAP-C02](https://portal.tutorialsdojo.com/courses/aws-certified-solutions-architect-professional-practice-exams/) | 70%+ |
| Practice Exam 2 | [Whizlabs SAP-C02](https://www.whizlabs.com/aws-solutions-architect-professional/) | 72%+ |
| Practice Exam 3 | [AWS Skill Builder Official](https://skillbuilder.aws/exam-prep/solutions-architect-professional) | 75%+ |
| Practice Exam 4 | Tutorials Dojo (section-mode per domain) | 80%+ per domain |

After each mock exam:
1. Review every wrong answer — no exceptions.
2. Tag wrong answers by domain and sub-topic.
3. Spend 30 minutes per wrong topic reading the relevant AWS documentation (not blogs — the official docs).

### Week 16 — Final Polish and Exam Strategy

**Exam technique:**

- [ ] **Read every question twice.** SAP-C02 questions are deliberately wordy. The constraint (cost, latency, compliance, operational overhead) is almost always in the second half of the question stem.
- [ ] **Look for the superlative:** "most cost-effective", "least operational overhead", "most resilient" — this narrows from 2 plausible answers to 1.
- [ ] **Eliminate clearly wrong answers first.** On multiple-response questions (select 2 or 3), wrong answers are usually identifiable by service misuse.
- [ ] **Flag and move.** If a question takes > 3 minutes, flag it and move. You have 144 seconds per question on average.
- [ ] **Never change an answer unless you have a specific reason.** First instinct is usually correct on SAP-C02.

**Known exam traps:**

| Trap | The Trick |
|------|-----------|
| "Highly available" + "cheapest" | HA usually requires multi-AZ; the cheapest HA option is often Fargate or Lambda over EC2 Multi-AZ |
| "Minimum operational overhead" | Managed services win (Aurora > RDS, Fargate > EC2, SSO > custom IdP) |
| "Existing on-premises AD" | IAM Identity Center with AD Connector, not Cognito |
| "Global users, low latency" | Global Accelerator for dynamic content, CloudFront for cacheable content |
| "SCP on management account" | SCPs do NOT apply to the management account — only to member accounts |
| "Lambda timeout > 15 minutes" | Lambda cannot do it — use Step Functions or ECS |
| "Prevent any IAM user from disabling CloudTrail" | SCP + deny on `cloudtrail:StopLogging` — not an IAM policy |
| "Real-time" vs "near-real-time" | Truly real-time = Kinesis Data Streams; near-real-time = Kinesis Firehose (buffering) |

**Two days before the exam:**
- [ ] Light review only — no new material.
- [ ] Re-read your own study notes and error log.
- [ ] Confirm exam booking, testing environment (ID, quiet room, webcam), or Pearson VUE centre location.

**Day of the exam:**
- [ ] Arrive / log in 30 minutes early.
- [ ] Do a quick mental warm-up: recall the four domain titles and their weightings.
- [ ] Use the scratch pad from the first 2 minutes to write down the key memory items: DR tiers, 7 Rs, SCP rules.

---

## Core AWS Services — Mastery Checklist

This is your definitive service checklist. You must be able to describe every service's purpose, key configuration options, limits, and common architectural patterns.

### Compute
- [ ] EC2 (instance families, placement groups, Nitro system, Graviton)
- [ ] Lambda (concurrency, SnapStart, layers, destinations)
- [ ] ECS + Fargate (task definitions, service auto scaling, capacity providers)
- [ ] EKS (managed node groups, Fargate profiles, IRSA, Karpenter)
- [ ] Step Functions (Standard vs Express, Map, Parallel, error handling)
- [ ] App Runner (fully managed, container or source code)
- [ ] Elastic Beanstalk (platform versions, rolling/immutable deployments)

### Networking
- [ ] VPC (subnets, route tables, NACLs, security groups, DHCP options)
- [ ] Transit Gateway (attachments, route tables, peering, multicast)
- [ ] Direct Connect (VIF types, Direct Connect Gateway, MACsec, LAG)
- [ ] Site-to-Site VPN (BGP, static, dual-tunnel, accelerated VPN)
- [ ] Route 53 (all routing policies, health checks, private hosted zones)
- [ ] CloudFront (cache policies, origin groups, Lambda@Edge, CF Functions)
- [ ] Global Accelerator (anycast routing, endpoint groups, health-aware)
- [ ] PrivateLink (endpoint services, interface endpoints)
- [ ] Network Firewall (stateful vs stateless rules, centralized inspection VPC)

### Storage
- [ ] S3 (all storage classes, lifecycle, replication, Object Lambda, pre-signed URLs, access points)
- [ ] EBS (volume types: gp3, io2 Block Express, st1, sc1; snapshots, multi-attach)
- [ ] EFS (performance modes, throughput modes, access points)
- [ ] FSx for Windows, Lustre, NetApp ONTAP, OpenZFS
- [ ] Snowball Edge, Snowmobile, DataSync, Transfer Family

### Databases
- [ ] RDS (Multi-AZ, read replicas, RDS Proxy, parameter groups)
- [ ] Aurora (cluster architecture, Serverless v2, Global Database, Parallel Query, Babelfish)
- [ ] DynamoDB (partition key design, GSI/LSI, DAX, Global Tables, Streams)
- [ ] ElastiCache (Redis cluster mode, replication groups, eviction policies)
- [ ] Redshift (RA3 nodes, Spectrum, concurrency scaling, data sharing)
- [ ] Neptune (graph DB, use cases)
- [ ] Timestream (time series, use cases vs DynamoDB)
- [ ] DocumentDB (MongoDB compatibility, use cases)
- [ ] Keyspaces (Cassandra compatibility)

### Messaging and Streaming
- [ ] SQS (FIFO vs Standard, visibility timeout, DLQ, batch processing)
- [ ] SNS (fan-out, message filtering, FIFO topics)
- [ ] EventBridge (event buses, rules, pipes, scheduler)
- [ ] Kinesis Data Streams (shards, consumers, enhanced fan-out, KCL)
- [ ] Kinesis Firehose (dynamic partitioning, transformation Lambda, buffering)
- [ ] MSK (Managed Kafka, MSK Connect, MSK Serverless)

### Security
- [ ] IAM (policies, roles, ABAC, permission boundaries, trust policies)
- [ ] IAM Identity Center (permission sets, SCIM, external IdP)
- [ ] KMS (key types, key policies, grants, envelope encryption, automatic rotation)
- [ ] Secrets Manager (rotation, cross-account, Lambda rotation function)
- [ ] Organizations + SCPs (inheritance, management account rules, `NotAction`)
- [ ] Control Tower (guardrails, account factory, landing zone)
- [ ] Security Hub, GuardDuty, Macie, Inspector
- [ ] WAF + Shield (Advanced), Network Firewall

### Operations and Governance
- [ ] CloudWatch (metrics, logs, alarms, dashboards, Contributor Insights, Synthetics)
- [ ] X-Ray (traces, service map, sampling rules, groups)
- [ ] CloudTrail (management events, data events, organisation trail)
- [ ] AWS Config (rules, conformance packs, aggregators, remediations)
- [ ] Systems Manager (Session Manager, Patch Manager, Parameter Store, Run Command)
- [ ] Trusted Advisor, Compute Optimizer, Well-Architected Tool
- [ ] Service Quotas (request increases, CloudWatch alarms on quota usage)

### Migration
- [ ] Application Migration Service (MGN)
- [ ] Database Migration Service (DMS) + Schema Conversion Tool (SCT)
- [ ] Migration Hub (tracking, network access analyser)
- [ ] DataSync, Snowball, Transfer Family

---

## Resources — Prioritised

### Free Resources (start here)
| Resource | URL | Priority |
|----------|-----|----------|
| AWS Skill Builder Exam Prep | [skillbuilder.aws](https://skillbuilder.aws/exam-prep/solutions-architect-professional) | ★★★★★ |
| AWS Official Documentation | [docs.aws.amazon.com](https://docs.aws.amazon.com/) | ★★★★★ |
| AWS Well-Architected Framework | [aws.amazon.com/architecture/well-architected](https://aws.amazon.com/architecture/well-architected/) | ★★★★★ |
| AWS Prescriptive Guidance | [aws.amazon.com/prescriptive-guidance](https://aws.amazon.com/prescriptive-guidance/) | ★★★★☆ |
| AWS Security Reference Architecture | [docs.aws.amazon.com/prescriptive-guidance/…/security-reference-architecture](https://docs.aws.amazon.com/prescriptive-guidance/latest/security-reference-architecture/welcome.html) | ★★★★☆ |
| AWS Whitepapers (Disaster Recovery) | [aws.amazon.com/disaster-recovery](https://docs.aws.amazon.com/whitepapers/latest/disaster-recovery-workloads-on-aws/disaster-recovery-workloads-on-aws.html) | ★★★★☆ |

### Paid Resources (choose one video course + one practice exam bank)
| Resource | Type | Notes |
|----------|------|-------|
| [Adrian Cantrill SAP-C02](https://learn.cantrill.io/p/aws-certified-solutions-architect-professional) | Video course | Most thorough; hands-on labs included |
| [A Cloud Guru SAP-C02](https://www.pluralsight.com/cloud-guru) | Video course | Good breadth, slightly lighter depth |
| [Tutorials Dojo SAP-C02](https://portal.tutorialsdojo.com/) | Practice exams | Best quality practice questions — essential |
| [Whizlabs SAP-C02](https://www.whizlabs.com/) | Practice exams | Good volume; use as secondary source |

### This Knowledge Portal (read before each phase)
| Phase | Docs |
|-------|------|
| 1 | `cloud/aws/12-multi-account-landing-zone.md`, `cloud/aws/05-iam-least-privilege.md`, `cloud/aws/04-vpc-networking.md` |
| 2 | `cloud/aws/14-disaster-recovery.md`, `cloud/aws/01-serverless-lambda.md`, `cloud/aws/09-sqs-sns-eventbridge.md`, `cloud/aws/07-rds-aurora.md`, `cloud/aws/08-dynamodb.md`, `cloud/aws/10-s3-storage.md` |
| 3 | `cloud/aws/15-cost-optimisation.md`, `cloud/aws/16-well-architected-framework.md`, `cloud/aws/13-cloudfront-cdn.md`, `devsecops/` |
| 4 | `cloud/aws/02-ecs-fargate.md`, `cloud/aws/03-eks-kubernetes.md`, `distributed-design-pattern/` |

---

## Readiness Checklist — Final Gate

Do not book the exam until you can check all of these:

```
Domains
  [ ] Domain 1 practice score consistently ≥ 75%
  [ ] Domain 2 practice score consistently ≥ 75%
  [ ] Domain 3 practice score consistently ≥ 75%
  [ ] Domain 4 practice score consistently ≥ 75%

Full mock exams
  [ ] 3 full timed mock exams completed
  [ ] Scoring ≥ 78% on most recent mock
  [ ] All wrong answers reviewed and traced to source documentation

Service knowledge
  [ ] Can describe Direct Connect Gateway vs Transit Gateway without notes
  [ ] Can draw the multi-account security model (management, log archive, security tooling accounts)
  [ ] Can explain all 7 R migration strategies with one example each
  [ ] Knows all four DR tiers with typical RTO/RPO ranges
  [ ] Can explain Aurora Global Database promotion sequence from memory
  [ ] Knows when Savings Plans beat Reserved Instances and vice versa

Exam mechanics
  [ ] Exam booked via Pearson VUE
  [ ] ID prepared (two forms if online proctored)
  [ ] Testing environment confirmed (quiet, clean desk, webcam working)
```

---

## After Passing

Once certified, the natural progression paths are:

| Path | Next Certification | Why |
|------|--------------------|-----|
| Security depth | [AWS Certified Security – Specialty](https://aws.amazon.com/certification/certified-security-specialty/) | Deepens IAM, KMS, GuardDuty, Detective |
| DevOps depth | [AWS Certified DevOps Engineer – Professional](https://aws.amazon.com/certification/certified-devops-engineer-professional/) | CI/CD, IaC, operational excellence |
| Networking depth | [AWS Certified Advanced Networking – Specialty](https://aws.amazon.com/certification/certified-advanced-networking-specialty/) | Deep Direct Connect, Transit Gateway, hybrid |
| Data depth | [AWS Certified Data Engineer – Associate](https://aws.amazon.com/certification/certified-data-engineer-associate/) | Data pipelines, Kinesis, Glue, Redshift |

Maintain the certification: **3-year renewal** via recertification exam or earning continuing education credits on AWS Skill Builder.

---

*Last updated: April 2026 — review against the official exam guide every 6 months as AWS updates the SAP-C02 content outline.*
