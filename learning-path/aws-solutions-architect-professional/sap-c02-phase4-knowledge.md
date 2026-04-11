# SAP-C02 Phase 4 — Deep Knowledge Reference
## Domain 4: Accelerate Workload Migration and Modernisation (Weeks 11–12)

> This document provides the detailed technical knowledge behind every bullet in the Phase 4 study plan.
> Cross-reference: [`aws-solutions-architect-professional.md`](./aws-solutions-architect-professional.md)

---

## Week 11 — Migration Strategies and Tools

### The 7 Rs — Migration Strategies

Every SAP-C02 migration scenario requires selecting the correct R. Know the definition, a concrete example, and the constraints of each.

| R | Strategy | Definition | When to use |
|---|----------|------------|------------|
| **Retire** | Remove | Decommission; the application is no longer needed | Duplicate apps post-merger; unused functionality |
| **Retain** | Keep | Leave on-premises; do not migrate yet | Regulatory constraints; recent investment; too risky |
| **Rehost** | Lift and shift | Move to AWS with no changes to the application code or architecture | Fastest migration; large server count; consolidation |
| **Relocate** | Lift and shift at platform level | Move to AWS without changing the deployment model (VMware Cloud on AWS; containers to EKS) | VMware environments; container workloads with K8s |
| **Replatform** | Lift, tinker, and shift | Minimal code changes to leverage cloud-native platform | Move Tomcat app to Elastic Beanstalk; Oracle → Aurora; Tomcat → ECS |
| **Repurchase** | Drop and shop | Replace with a SaaS product | CRM → Salesforce; ERP → SAP on AWS; Email → Microsoft 365 |
| **Refactor/Re-architect** | Re-architect | Significantly redesign to be cloud-native | Monolith → microservices; move to serverless; event-driven redesign |

**Exam decision heuristics:**

```
"Cannot touch the application code" → Rehost (fastest) or Retain (if can't move)
"Minimal changes acceptable; want managed DB" → Replatform
"End users complain about old UX; vendor has cloud version" → Repurchase
"Business wants to scale elastically, current design can't" → Refactor
"Application hasn't been used in 2 years" → Retire
"Compliance requires it to stay on-premises" → Retain
"Running VMware; want same tooling on AWS" → Relocate (VMware Cloud on AWS)
```

**Rehost vs Relocate:**
- Rehost: Application runs on EC2 in AWS — different hypervisor, different tooling.
- Relocate: Application runs on VMware Cloud on AWS — same vSphere tooling, same APIs, same VMware management plane. No retraining required for VMware admins.

---

### AWS Application Migration Service (MGN)

MGN is the **recommended rehost service**, replacing the older SMS (Server Migration Service).

**How it works:**
1. Install the **MGN agent** on the source server (on-premises or other cloud).
2. Agent replicates the server's block storage continuously to AWS (to a **replication server** in a staging subnet).
3. When ready: launch a **test instance** to validate the migrated server.
4. **Cutover:** Stop source server, launch **cutover instance** from the final replicated state.

**Agentless replication (Agentless Snapshot Replication):**
- Available for VMware vCenter — no agent installed on VMs.
- Uses vSphere API to take snapshots and replicate.
- Less real-time than agent-based; use when agents cannot be installed.

**Key architecture:**
```
On-premises server (agent) ─────────→ Replication Server (staging subnet, t3.small)
                                          ↓ (replication via VPN/DX on port 1500 TCP)
                                       EBS volumes (staging)
                                          ↓ (on cutover)
                                       Real EC2 instance (production subnet)
```

**MGN vs DMS:** MGN migrates whole servers (OS + application + data). DMS migrates databases only.

---

### AWS Database Migration Service (DMS)

**Migration types:**

| Type | Use case | Downtime |
|------|----------|---------|
| **Full Load** | One-time copy of all data; no ongoing sync | Application down during migration |
| **CDC only** | Ongoing change capture (source already has data at target) | Near-zero (sync ongoing changes) |
| **Full Load + CDC** | Full initial load then ongoing change capture until cutover | Minimal (near-zero after load completes) |

**Replication instance:** An EC2 instance that runs the DMS software. Size based on migration throughput. Place in same VPC as target DB for low latency.

**Source and target support:**

| Migration | Source → Target |
|-----------|----------------|
| Homogeneous | MySQL → MySQL; Postgres → Aurora Postgres; Oracle → Oracle |
| Heterogeneous | Oracle → Aurora MySQL (requires SCT first); SQL Server → Aurora Postgres |

**Schema Conversion Tool (SCT):**
- Must be run BEFORE DMS for heterogeneous migrations.
- Converts DDL (CREATE TABLE, stored procedures, views, triggers) to target dialect.
- Reports conversion complexity: auto-convertible vs requires manual intervention.
- Run on a local Windows/Linux machine (not cloud-based).

**LOB (Large Object) handling:**
- `Limited LOB mode`: truncate LOBs to a max size. Fastest.
- `Full LOB mode`: replication server fetches the full LOB from source. Slower but complete.
- `Inline LOB mode`: auto-selects based on LOB size per row.

**DMS for ongoing replication / CDC:**
- Source must have binary logging (MySQL) or supplemental logging (Oracle) enabled.
- SCT is not needed for per-table CDC — only for schema changes.

**DMS Serverless:** Automatically scales replication capacity based on workload. No need to specify or manage a replication instance. Billing per capacity unit/hour.

---

### AWS DataSync

DataSync is a **managed data transfer service** for moving files between on-premises storage and AWS storage services.

**Supported sources and destinations:**

| Source | Destination |
|--------|-------------|
| NFS (on-premises) | S3 (any storage class) |
| SMB (Windows shares) | EFS |
| HDFS | FSx for Windows |
| AWS storage (S3, EFS, FSx) | AWS storage (cross-region, cross-account) |
| Object storage (NFS compatible) | FSx for Lustre, FSx for NetApp ONTAP |

**Key features:**
- Validates data integrity (checksums at source and destination).
- Bandwidth throttling — set maximum transfer bandwidth.
- Scheduling — one-time or recurring (hourly, daily, weekly).
- DataSync agent: deployed as VM (VMware/Hyper-V) or EC2 on-premises.
- Encryption in transit (TLS).

**DataSync vs Snowball vs storage gateway:**
```
DataSync: online bulk transfer (structured, scheduled, with integrity checks)
Snowball: offline bulk transfer (> 10 TB, poor connectivity, physical security)
Storage Gateway: ongoing hybrid access (on-premises apps need persistent access to S3)
```

**Throughput calculation for exam:**
```
4 PB of data, 1 Gbps connection:
  Theoretical: 4 PB / (1 Gbps / 8 bits) = 4,000,000 GB / 125 MB/s = 37 days
  Practical with overhead: ~50+ days

Rule of thumb: 1 Gbps full utilisation = ~10 TB/day.
  4 PB = 4,000 TB / 10 TB/day = 400 days → Use Snowball.
  40 TB = 40 days of reasonable window → DataSync might work.
  Only use internet if transfer can complete in acceptable time.
```

---

### AWS Snow Family

| Device | Storage | Use case |
|--------|---------|---------|
| **Snowcone** | 8 TB HDD or 14 TB SSD | Smallest; edge computing; tactical/harsh environments |
| **Snowball Edge Storage Optimised** | 80 TB usable | Large-scale data migration |
| **Snowball Edge Compute Optimised** | 28 TB NVMe + 40 vCPUs, 104 GB RAM | Edge compute (IoT, ML inference, content processing) |
| **Snowmobile** | 100 PB per truck | Exabyte-scale migration; physical/security requirements |

**Clustering:** Up to 10 Snowball Edge devices can be clustered for a larger erasure-coded storage pool.

**When to use Snowmobile:** > 10 PB to migrate AND physical security controls required (it's a truck with 24/7 security escort).

---

### AWS Transfer Family

Transfer Family provides managed **SFTP, FTPS, FTP, and AS2 endpoints** backed by S3 or EFS.

**Identity providers:**
- Service-managed (Transfer Family own user directory)
- AWS Directory Service (AD)
- Custom Lambda-based identity provider (API Gateway + Lambda)

**Use case:** Organisations with trading partners or legacy systems using SFTP; lift-and-shift of managed file transfer infrastructure without changing client configurations.

---

### AWS Migration Hub

Central dashboard tracking migration progress across tools (MGN, DMS, Server Migration Service).

**Network Access Analyser:** Analyses VPC configurations to identify unintended network access paths before migration.

**Migration Hub Refactor Spaces:** Manages the infrastructure for Strangler Fig pattern — provides an environment and routing layer for incremental modernisation.

---

## Week 12 — Application Modernisation Patterns

### Strangler Fig Pattern

The Strangler Fig is the safest, lowest-risk approach to decomposing a monolith. New functionality is built as separate microservices; old monolith routes are gradually replaced.

**Architecture:**
```
Client → API Gateway / Load Balancer (routing layer)
            ├── New: Payment Microservice (ECS/Lambda)     ← migrated routes
            ├── New: Inventory Microservice (ECS/Lambda)   ← migrated routes
            └── Legacy: Monolith (EC2/Elastic Beanstalk)  ← remaining routes

Over time: monolith routes shrink → eventually retired
```

**Implementation on AWS:**
1. ALB with listener rules: path-based routing (`/api/payments/*` → new service; all other → monolith).
2. API Gateway with HTTP integration: route traffic to new Lambda/ECS or to monolith ALB.
3. Migration Hub Refactor Spaces: manages the routing layer and environment isolation.

**Anti-corruption layer:** An adapter between old monolith domain model and new microservice domain model. Prevents legacy concepts from leaking into new bounded contexts.

---

### Microservices Decomposition Principles

**Bounded contexts (Domain-Driven Design):**
Each microservice owns one bounded context. Services are isolated by data (each has its own database — no shared DB schema).

**Decomposition approaches:**
1. **By business capability:** Order management, Customer management, Inventory — maps to teams.
2. **By sub-domain:** Identify core, supporting, and generic sub-domains. Start migrating generic sub-domains (lower risk).
3. **By volatility:** Extract the parts that change most frequently first — they benefit most from independent deployability.

**Database-per-service:**
- Order Service → DynamoDB (high write throughput, simple access patterns)
- Customer Service → Aurora PostgreSQL (relational, reporting queries)
- Inventory Service → Redis + DynamoDB (low-latency reads, eventual consistency acceptable)

---

### Container Adoption Path

```
Phase 1 (Replatform): Monolith → Docker container → ECS Fargate
  - Containerise the existing application
  - Same code, same architecture, now in a container
  - Benefit: easier deployment, no server management

Phase 2 (Refactor): Decompose into microservices → ECS or EKS
  - Break out individual services into separate containers
  - Each service has its own CI/CD pipeline
  - ECS if team unfamiliar with Kubernetes; EKS if K8s required

Phase 3 (Cloud-native): Serverless where applicable
  - Event-driven processes → Lambda
  - Synchronous APIs → API Gateway + Lambda or ECS Fargate
  - Scheduled tasks → EventBridge Scheduler + Lambda
```

**App Runner:**
- Fully managed container platform — no ECS concepts, no task definitions, no service discovery.
- Input: container image or source code repo.
- AWS handles scaling, load balancing, TLS, health checks.
- Use case: simple web services, APIs where the team doesn't want platform knowledge.
- When NOT to use: stateful, require VPC-complex networking, large data volumes.

---

### Serverless Refactoring

**Characteristics of good Lambda candidates:**
- Short-lived operations (< 15 minutes)
- Event-triggered (not always-on)
- Stateless (no in-process state between invocations)
- Variable or spiky load (Lambda scales from 0 to thousands instantly)

**Anti-patterns for Lambda:**
- Jobs > 15 minutes (use Step Functions + Lambda, or ECS/Fargate)
- WebSocket long-lived connections (use API Gateway WebSocket API, but Lambda manages connections)
- Shared mutable state in memory (use DynamoDB or ElastiCache)
- Extremely latency-sensitive startup (use Provisioned Concurrency or Fargate instead)

**Strangler Fig applied to Lambda:**
```
Legacy REST controller (monolith)
  → Extract business logic to Lambda functions
  → API Gateway routes to Lambda for extracted endpoints
  → Remaining endpoints still hit monolith
  → DB access layer stays same initially; evolve toward per-service DB
```

---

### Database Modernisation — Oracle/SQL Server → Aurora

**Migration path:**

```
Step 1: Run SCT (Schema Conversion Tool)
  → Converts schemas, procedures, triggers
  → Reports manual effort required (action items)

Step 2: Manually fix high-complexity items
  → Rewrite stored procedures, Oracle-specific syntax

Step 3: Full Load with DMS
  → Migrate all existing data with application down or near-zero downtime (Full Load + CDC)

Step 4: Validate
  → Row count comparison
  → Query-level validation (DMS data validation feature)

Step 5: Cutover
  → Stop writes to source
  → Let CDC drain
  → Update connection strings to Aurora
  → Monitor for errors
  → Keep source read-only for rollback period (2 weeks)
```

**Aurora PostgreSQL for Oracle:** Supports Babelfish for T-SQL compatibility. For PL/SQL (Oracle): use AWS SCT + manual rewrite.

---

### Decoupling Synchronous Systems

**Problem:** Tight coupling via synchronous HTTP calls — downstream service failure causes upstream failure (cascade failure).

**Solution patterns:**

```
1. Synchronous (existing)
   Service A → HTTP → Service B → HTTP → Service C
   (If C fails, B fails, A fails)

2. Async with SQS
   Service A → SQS queue → Service B → SQS queue → Service C
   (If C is slow, jobs queue; A and B continue; backpressure managed)

3. Event-driven with EventBridge
   Service A → emits OrderCreated event → EventBridge
     → Service B (fulfillment) receives event
     → Service C (notification) receives event
   (Fully decoupled; order of processing independent)
```

**Introducing SQS as an intermediary:**
- Add SQS queue between services without changing service business logic.
- Service A writes to queue instead of calling B directly.
- Service B polls the queue.
- Easy to add DLQ, retry, visibility timeout controls.

---

### Read Replica Promotion as a Zero-Downtime Migration Strategy

**Pattern:**

```
1. Create a read replica of the source database (same region or cross-region).
2. Application continues writing to source.
3. After replica is caught up:
   a. Stop writes to source (brief downtime or maintenance window).
   b. Promote the read replica to a standalone primary.
   c. Update connection strings.
   d. Resume writes to new primary.
5. Source becomes the old/legacy database — drain and decommission.
```

This is commonly used to:
- Migrate from RDS MySQL → Aurora MySQL (create Aurora read replica of RDS, promote).
- Migrate across versions in-place (major version upgrade via replica promotion).
- Move to a different instance size by creating a replica and promoting.

**RDS to Aurora migration via read replica:**
```
RDS MySQL (source) → Create Aurora Read Replica → Promote Aurora replica → 
Update connection strings → Decommission RDS source
```

---

## Phase 4 — Key Decision Framework

### "Which migration tool for this scenario?"

```
Need to migrate a full server (OS + apps + data) to EC2?
  → Application Migration Service (MGN)

Need to migrate a database (schema + data)?
  → DMS (+ SCT for heterogeneous)

Need to transfer files/data from NFS/SMB to S3/EFS?
  → DataSync

Data volume too large or connectivity too slow for online transfer?
  → Snowball Edge (up to ~80 TB per device, clusterable)
  → Snowmobile (> 10 PB, physical security requirements)

Need to give legacy partners SFTP access to S3?
  → Transfer Family

Need to track migration progress across multiple tools?
  → Migration Hub
```

### "Which modernisation approach?"

```
"Cannot change code, need to get to AWS fast" → Rehost (MGN)
"On VMware, want same tooling" → Relocate (VMware Cloud on AWS)
"Move to managed platform, minimal code change" → Replatform (Beanstalk, RDS, ECS)
"Existing system has scalability problems, need redesign" → Refactor
"Reduce risk, incremental migration" → Strangler Fig pattern
"Decompose monolith to microservices" → Bounded context decomposition + Strangler Fig
"Tightly coupled services causing cascades" → Introduce SQS/EventBridge between services
```

---

## Phase 4 — Exam Trap Summary

| Trap | Correct Answer |
|------|---------------|
| "Rehost vs Relocate — what's the difference?" | Rehost = change to EC2. Relocate = same platform, e.g. VMware Cloud on AWS, same VMware tooling |
| "DMS migrates the application code?" | NO. DMS migrates database schema and data only. MGN migrates full servers |
| "SCT is required for homogeneous migrations?" | NO. SCT is only required for heterogeneous migrations (e.g., Oracle → Aurora Postgres) |
| "DataSync vs Storage Gateway for ongoing hybrid access?" | DataSync = one-time or scheduled bulk transfer. Storage Gateway = persistent hybrid access |
| "Snowball device count for 800 TB?" | 800 TB / 80 TB per device = 10 Snowball Edge devices |
| "DMS Full Load + CDC — when can you cut over?" | After initial load completes, wait for CDC lag to reach near-zero, then cut over |
| "Strangler Fig requires big-bang deployment?" | NO. Strangler Fig is incremental; routes migrate one by one |
| "App Runner vs ECS Fargate for simple API?" | App Runner for simplest operations (no service mesh, no complex networking). ECS Fargate for VPC integration, service discovery, sidecars |
| "Lambda > 15 min job" | Step Functions (orchestrate multiple Lambda calls) or ECS Fargate |
| "ROI calculation on migration: Retire vs Retain?" | Retire saves 100% of running costs. Retain = no migration cost but no cloud savings |
| "Read replica promotion causes data loss?" | NO (if you stop writes and wait for replication to catch up before promoting) |

---

## Cross-Phase Knowledge — Migration Project Phases

For architecting a full migration project (common in domain 4 scenarios):

```
Phase 1: Discover
  → AWS Migration Hub Discovery
  → AWS Application Discovery Service (agent or agentless)
  → Dependency mapping (Application Dependency Discovery)
  → Outputs: server inventory, dependency groups, TCO comparison

Phase 2: Assess
  → Classify each application: 7 R strategy
  → Score by: complexity, business criticality, migration urgency
  → Build migration wave plan (tackle easy apps first)

Phase 3: Mobilise
  → Prove the pattern: migrate 2–3 apps as pilot
  → Establish CI/CD pipelines for AWS
  → Train team on AWS platform

Phase 4: Migrate and Modernise
  → Execute waves
  → Rehost first (fastest), then replatform/refactor in subsequent waves
  → Validate each wave before proceeding

Phase 5: Operate and Optimise
  → Decommission on-premises servers after proven stable on AWS
  → Right-size (Compute Optimizer)
  → Apply Savings Plans/RIs
  → Ongoing Well-Architected Reviews
```
