# SAP-C02 Phase 3 — Deep Knowledge Reference
## Domain 3: Continuous Improvement for Existing Solutions (Weeks 8–10)

> This document provides the detailed technical knowledge behind every bullet in the Phase 3 study plan.
> Cross-reference: [`aws-solutions-architect-professional.md`](./aws-solutions-architect-professional.md)

---

## Week 8 — Performance Optimisation

### Caching Hierarchy

Understand where each caching layer sits and what it caches:

```
Browser cache / CDN (CloudFront edge)
  → ElastiCache (application-level, in-region)
    → DAX (DynamoDB-specific)
      → RDS Read Replica (database read scaling, not a cache per se)
```

**CloudFront caching:**
- Caches at 450+ PoP locations worldwide.
- Cache key: URL + headers + query strings (configure with Cache Policy).
- TTL controlled via Cache-Control headers from origin or CloudFront default TTL.
- **Origin groups:** Primary + failover origin. On 5xx from primary, CloudFront automatically requests from failover.
- **Lambda@Edge:** Run code at CloudFront nodes (N. Virginia replicated globally). Triggers: viewer request, viewer response, origin request, origin response.
- **CloudFront Functions:** Lighter, faster, cheaper than Lambda@Edge. JavaScript only. Viewer request/response only. Use for: URL rewriting, header manipulation, token validation.

| | CloudFront Functions | Lambda@Edge |
|--|---------------------|-------------|
| Runtime | JavaScript (ES5.1) | Node.js, Python |
| Trigger points | Viewer request/response | All 4 triggers |
| Execution time | < 1 ms | 5–30 s |
| Network access | No | Yes |
| Cost | $0.10/million | $0.60/million + duration |

---

### Auto Scaling Strategies

**EC2 Auto Scaling policies:**

| Policy type | How it scales | Best for |
|-------------|--------------|---------|
| **Target tracking** | Maintains a metric at a target value (e.g., CPU = 60%) | Steady, predictable load |
| **Step scaling** | Steps of capacity based on alarm breach magnitude | Irregular spikes |
| **Scheduled scaling** | Scale at a specific time | Known patterns (business hours, batch jobs) |
| **Predictive scaling** | ML-based forecast + pre-scaling | Recurring patterns with lead time needed |

**Important warm-up and cooldown settings:**
- **Cooldown period:** After scale-out, wait N seconds before another scale-out (prevents over-scaling). Default 300 s.
- **Instance warmup:** Time for a new instance to serve traffic. Metrics during warmup are not counted toward scaling triggers.
- **Lifecycle hooks:** Pause instance at launch (`Pending:Wait`) or termination (`Terminating:Wait`) to run custom actions (install agent, drain connections, upload logs).

**Warm pools:**
- Pre-initialised, stopped EC2 instances waiting in a pool.
- When scale-out triggered, instances start from Stopped state (~30 seconds) instead of cold launch (~3–5 minutes).
- Cost: pay for EBS volumes only while stopped (no compute charge).
- Reuse for: Java apps with slow startup, large AMI installs.

**Application Auto Scaling** — targets beyond EC2:
- ECS service tasks (target tracking on CPU or custom metric)
- DynamoDB read/write capacity units
- Aurora read replicas
- Lambda provisioned concurrency
- SageMaker endpoint variants
- Spot Fleet capacity

---

### Kinesis Resharding

**Shards define throughput:**
- 1 shard = 1 MB/s write + 2 MB/s read
- Add shards for more write throughput; remove shards to reduce cost.

**Split:** 1 shard → 2 shards (increases throughput). The original shard becomes read-only; consumers must drain it before the new shards are active.

**Merge:** 2 adjacent shards → 1 shard (reduces cost). Adjacent means their hash key ranges are contiguous. Consumers must read from the merged shard.

**Resharding impact on consumers:**
- KCL handles resharding automatically — workers detect new/removed shards.
- During split: consumers must finish processing the parent shard before consuming child shards.
- Do NOT reshard too frequently — resharding takes time and disrupts consumer assignments.

---

### Compute Optimizer and Right-Sizing

**Compute Optimizer** analyses CloudWatch metrics for:
- EC2 instances (CPU, memory with CW agent, network)
- EC2 Auto Scaling groups
- Lambda functions (memory allocation vs actual memory used)
- ECS on Fargate (CPU/memory)
- EBS volumes (IOPS, throughput)

**Recommendations:** Over-provisioned, under-provisioned, or optimised. Also recommends Graviton-based instances where savings are available.

**Trusted Advisor high-value checks:**
- Underutilised EC2 instances (< 10% max CPU over 4 days)
- Idle RDS instances (no connections)
- Unassociated Elastic IPs
- Underutilised EBS volumes
- Reserved Instance expiry alerts

---

### S3 Transfer Acceleration

Uses CloudFront edge locations as upload accelerators. Files upload to the nearest edge node, then transferred to S3 via the AWS backbone network.

When beneficial: Uploading large files from geographically distant clients (cross-region or high-latency connections). Not beneficial for short distances.
Cost: additional fee per GB transferred; only charged if faster than standard upload.

---

## Week 9 — Security Posture Improvement

### AWS KMS — Deep Dive

**Key types:**

| Type | Control | Rotation | Use case |
|------|---------|---------|---------|
| **AWS managed key** | AWS manages everything | Automatic (every year) | Default for services (S3-SSE-KMS, RDS) |
| **Customer managed key (CMK)** | You define key policy, rotation | Optional automatic (yearly) | Custom key policy, cross-account, audit |
| **Customer managed + custom key store** | Keys stored in CloudHSM | Manual | Regulatory: keys must never leave HSM |

**Key policy vs IAM policy:**
- Key policy is **resource-based policy on the key**.
- Unlike IAM, the CMK key policy is the **primary access control** — IAM alone cannot grant KMS access.
- The key policy must explicitly allow the IAM principal (or grant access to the root account so IAM can delegate).

**Default key policy (root account delegation):**
```json
{
  "Sid": "Enable IAM User Permissions",
  "Effect": "Allow",
  "Principal": {"AWS": "arn:aws:iam::123456789012:root"},
  "Action": "kms:*",
  "Resource": "*"
}
```
This allows IAM policies to control the key. Without this statement, only the key policy itself controls access.

**Grants:** Temporary, narrowly-scoped permissions delegated to a principal, often used by AWS services (EBS, S3) on your behalf without needing key policy changes.

**Envelope Encryption:**
```
CMK (never leaves KMS)
  → GenerateDataKey API → returns:
      plaintext data key (in memory only)
      encrypted data key (stored alongside encrypted data)

Encrypt:
  plaintext data → AES encryption with plaintext data key → ciphertext
  plaintext data key → discarded from memory
  encrypted data key → stored with ciphertext

Decrypt:
  encrypted data key → KMS Decrypt → plaintext data key
  ciphertext → AES decryption with plaintext data key → plaintext
```

Why envelope encryption? KMS has a 4 KB limit per encrypt operation. Envelope encryption uses the CMK only to protect a small data key, then uses that data key to encrypt arbitrarily large data.

**Cross-account KMS:**
1. CMK key policy: allow `kms:Decrypt` for the external account's root or specific role.
2. External account's IAM policy: allow `kms:Decrypt` on the CMK ARN.

---

### Secrets Manager

**Rotation:**
- Supported databases: RDS, Aurora, Redshift, DocumentDB — native rotation without custom code.
- Custom rotation: Lambda function with four phases: `createSecret`, `setSecret`, `testSecret`, `finishSecret`.

**Rotation impact on applications:** During rotation, the old version (`AWSPREVIOUS`) is kept until the new version (`AWSCURRENT`) is validated. Applications calling `GetSecretValue` get the current version; zero downtime if using the managed rotation feature.

**Cross-account secret access:**
1. Secret's resource policy: allow `secretsmanager:GetSecretValue` for the external account/role.
2. Secret's KMS key policy: allow `kms:Decrypt` for the external account.
3. External IAM policy: allow the API call.

**Secrets Manager vs Parameter Store (cost model):**

| Feature | Secrets Manager | Parameter Store Standard | Parameter Store Advanced |
|---------|----------------|-------------------------|-------------------------|
| Cost | $0.40/secret/month + $0.05/10k API calls | Free | $0.05/parameter/month |
| Rotation | Built-in | Manual (you build) | Manual |
| Value size | 64 KB | 4 KB | 8 KB |
| Cross-account | Yes | With resource policy | With resource policy |

Use Secrets Manager when: rotation is required, or the value is a database credential.
Use Parameter Store when: configuration values, non-rotating secrets, cost-sensitive.

---

### GuardDuty

GuardDuty is a **threat detection service** using ML + managed threat intelligence. It analyses CloudTrail, VPC Flow Logs, DNS logs, and EKS audit logs without requiring you to enable those services separately for GuardDuty.

**Finding categories:**

| Category | Example threats |
|----------|----------------|
| Backdoor | EC2 instance communicating with known C2 (command-and-control) server |
| Cryptocurrency | EC2 instance running Bitcoin miner |
| UnauthorizedAccess | Unusual IAM user activity from Tor exit node |
| Recon | Port scanning, failed login attempts |
| Exfiltration | EC2 instance sending large data volume to unusual IP |
| S3 | Bucket accessible from Tor node; bucket policy changed to allow public access |
| EKS | Privilege escalation attempt in Kubernetes |

**Suppression rules:** Automatically archive findings matching specific criteria (e.g., suppress EC2 findings from your monitoring IP CIDR).

**Organisational integration:** Designate a delegated administrator account. GuardDuty auto-enrolls all current and new member accounts. Aggregated findings in the administrator account.

---

### Amazon Inspector v2

Inspector v2 continuously scans EC2 instances and ECR container images for **software vulnerabilities (CVEs)**.

| Target | What it scans |
|--------|--------------|
| EC2 | OS packages, installed software for CVEs; network reachability |
| ECR | Container image layers scanned on push and continuously re-evaluated as new CVEs emerge |

**SBOM (Software Bill of Materials):** Inspector v2 can export an SBOM in CycloneDX or SPDX format for all scanned resources.

**Inspector vs Macie:**
- Inspector: vulnerability scanning (CVEs, software) on compute and containers.
- Macie: sensitive data discovery in S3 (PII, credentials, financial data).

---

### AWS WAF

WAF operates at Layer 7 and can be attached to: CloudFront, ALB, API Gateway, AppSync, Cognito User Pool.

**Rule types:**

| Type | Description |
|------|-------------|
| **AWS managed rule groups** | Pre-built rules for OWASP Top 10, known bad IPs, bot signatures |
| **Custom rules** | Match on IP, headers, body, query strings, URI |
| **Rate-based rules** | Limit requests per 5-minute window per IP (or aggregated by custom key) |
| **IP sets** | Allow/deny lists of IP addresses or CIDRs |
| **Regex pattern sets** | Match arbitrary regex against request components |

**Bot Control managed rule group:** Categorises bots as common (search engines, monitoring) and targeted (credential stuffers, scrapers). Includes CAPTCHA challenge capability.

**Rule group capacity:** WCUs (WAF Capacity Units). Default limit: 1,500 WCU per web ACL. Each managed rule group costs a fixed WCU.

---

### AWS Shield

| Tier | Coverage | Features |
|------|----------|---------|
| **Shield Standard** | All customers, automatic | Basic L3/L4 DDoS protection (SYN floods, UDP reflection) |
| **Shield Advanced** | Opt-in, $3,000/month/org | L7 protection, DRT access, cost protection, proactive engagement, WAF included |

**Shield Advanced cost protection:** AWS credits you for scaling costs caused by a verified DDoS attack (EC2, ELB, CloudFront, Route 53 charges).

**Proactive engagement:** The DRoS (AWS DDoS Response Team) reaches out proactively during an attack when health checks fail — you don't need to raise a support case.

**DRT (DDoS Response Team) access:** Grant the DRT access to WAF rules and Shield Advanced configurations so they can write mitigations on your behalf during an attack.

---

### Network Layered Defence Model

```
Internet → 
  Route 53 (anycast; Shield Standard protects Route 53 for free)
  CloudFront (edge; Shield Standard + WAF ACL)
  ALB (Shield Advanced; WAF ACL; security groups)
  VPC Network Firewall (stateful L3-L7; centralised inspection VPC)
  Security Groups (stateful; instance-level)
  NACLs (stateless; subnet-level; deny specific IPs)
  Application (auth, rate limiting)
```

**Network Firewall vs WAF vs Security Groups:**

| Layer | Tool | What it inspects |
|-------|------|-----------------|
| L3/L4 network (VPC perimeter) | AWS Network Firewall | IP, ports, protocols, stateful rules, Suricata IDS |
| L7 HTTP/HTTPS | WAF | HTTP headers, body, URI, rate limiting, OWASP rules |
| Instance/service | Security Groups | Stateful; allow rules only; source IP or SG |
| Subnet | NACLs | Stateless; allow + deny; source IP; numbered rules |

---

### VPC Flow Logs Analysis with Athena

```sql
-- Find top talkers by bytes sent
SELECT srcaddr, dstaddr, SUM(bytes) AS total_bytes
FROM vpc_flow_logs
WHERE action = 'ACCEPT'
GROUP BY srcaddr, dstaddr
ORDER BY total_bytes DESC
LIMIT 20;

-- Find rejected traffic from a specific source
SELECT srcaddr, dstaddr, dstport, protocol, action
FROM vpc_flow_logs
WHERE srcaddr = '203.0.113.10'
  AND action = 'REJECT';
```

Flow log record fields: `version, account-id, interface-id, srcaddr, dstaddr, srcport, dstport, protocol, packets, bytes, start, end, action, log-status, vpc-id, subnet-id, instance-id, ...`

---

### Security Hub

Security Hub aggregates findings from: GuardDuty, Inspector, Macie, IAM Access Analyzer, Firewall Manager, and third-party integrations.

**Security standards (benchmarks):**
- CIS AWS Foundations Benchmark
- AWS Foundational Security Best Practices (FSBP)
- PCI-DSS

Each standard is a collection of Config-based checks that produces pass/fail findings. Security Hub gives an overall security score per standard.

**Custom actions:** Send filtered findings to EventBridge → trigger Step Functions or Lambda for automated remediation.

---

## Week 10 — Cost Optimisation and Operational Excellence

### Savings Plans

| Type | Flexibility | Discount |
|------|-------------|---------|
| **Compute Savings Plans** | Any EC2 instance family, size, AZ, OS, tenancy; also Fargate and Lambda | Up to 66% |
| **EC2 Instance Savings Plans** | Specific instance family in a region (e.g., m5 in eu-west-1); any AZ, size, OS | Up to 72% |
| **SageMaker Savings Plans** | SageMaker instance usage | Up to 64% |

**Commitment:** 1 or 3 years; all upfront, partial upfront, or no upfront.
**Compute Savings Plans** are the most flexible — they automatically apply as you change instance types. Choose these for environments with evolving compute requirements.

**Savings Plans vs Reserved Instances:**

| | Savings Plans | Reserved Instances |
|--|--------------|-------------------|
| Scope | Account/org | AZ-scoped or region-scoped |
| Billing discount | Applied to On-Demand bill | Applied to matching usage |
| Transferability | N/A (applies automatically) | Marketplace for Standard RIs |
| Applies to Fargate/Lambda | Yes (Compute SP) | No |
| Instance flexibility | Yes (Compute SP) | Convertible RI only |

**Exam scenario:** Multi-region, multi-instance type workload migrating to Graviton → Compute Savings Plan (most flexible across all dimensions). Single instance family locked to one region → EC2 Instance Savings Plan (higher discount).

---

### Spot Instances

**Interruption:** AWS can reclaim Spot instances with 2-minute warning. Applications must handle this gracefully.

**Interruption handling:**
- Use the **instance metadata service** endpoint `http://169.254.169.254/latest/meta-data/spot/termination-time` (polled) or **EventBridge** event `EC2 Spot Instance Interruption Warning` (preferred — no polling needed).
- Checkpoint state to S3 or EFS before termination.
- Drain connections via load balancer before shutdown.

**Spot Fleet strategies:**

| Strategy | Behaviour | Use case |
|----------|-----------|---------|
| **lowest-price** | Always launch from cheapest pool | Cost minimisation; risk of simultaneous interruption |
| **diversified** | Spread across all configured pools | Reduces interruption risk |
| **capacity-optimised** | Launch from pools with most available capacity | Minimise interruptions |
| **price-capacity-optimised** | Best balance of price and capacity availability | Recommended for most use cases |

**EC2 Auto Scaling with Spot:**
- Use mixed instances policy: base of On-Demand + remainder as Spot.
- Configure multiple instance types as overrides (diversification).
- `SpotAllocationStrategy: price-capacity-optimized` in launch template.

---

### Reserved Instances

**Standard RI:** Fixed instance family, size, OS, tenancy in a region or AZ.
- AZ-scoped: Reserves capacity in a specific AZ. Useful for Spot competition.
- Region-scoped: Applies to any matching instance in the region across AZs. No capacity reservation.

**Convertible RI:** Can be exchanged for a different RIs of equal or greater value (instance family, OS, tenancy). No Marketplace resale.

**RI Marketplace:** Sell unused Standard RIs with remaining term. Buyer gets the time remaining on the RI at a negotiated price.

---

### S3 Cost Optimisation

**Lifecycle policies:**
```
STANDARD → after 30 days → STANDARD_IA → after 90 days → GLACIER_INSTANT → after 180 days → GLACIER_DEEP_ARCHIVE
```

**Intelligent-Tiering tiers (automatic, no retrieval fee):**
- Frequent Access (default)
- Infrequent Access (after 30 days without access → saves 40%)
- Archive Instant Access (after 90 days → saves 68%)
- Archive Access (after 90–180 days, opt-in → saves 71%)
- Deep Archive Access (after 180+ days, opt-in → saves 95%)

**Data transfer cost reduction:**
- Use VPC endpoints for S3 access from EC2/Lambda → no NAT gateway costs, no internet egress.
- Co-locate compute in same AZ as data where possible (cross-AZ = $0.01/GB).
- Use S3 bucket region in same region as compute.

---

### AWS Systems Manager — Key Features

**Session Manager:**
- Secure shell/RDP access to EC2/on-premises servers WITHOUT opening port 22/3389.
- Traffic tunnelled through SSM agent → no bastion host required.
- All sessions logged to CloudWatch and S3.

**Parameter Store:**
- Hierarchical key-value store for configuration and secrets.
- Standard: free, 10,000 parameters, 4 KB.
- Advanced: $0.05/parameter/month, 100,000 parameters, 8 KB, parameter policies (TTL-based expiration, notification on access).
- SecureString type: encrypted with KMS.

**Patch Manager:**
- Defines patch baselines (approved/rejected patches by severity, CVE, product).
- Patch groups (tags on instances) → maintenance windows → patching schedules.
- Generates compliance reports via Config.

**Run Command:**
- Run scripts or documents across fleets of instances without SSH.
- Output to S3 or CloudWatch Logs.
- Rate control: target percentage of instances concurrently.

---

### AWS Well-Architected Framework — Six Pillars

**Exam questions test: which pillar does this scenario violate, and what is the most impactful improvement?**

| Pillar | Focus | Key tool |
|--------|-------|---------|
| **Operational Excellence** | Run and monitor; small, frequent, reversible changes | Systems Manager, CloudWatch, CodePipeline |
| **Security** | Protect data, systems; least privilege; encryption | IAM, KMS, GuardDuty, Shield, WAF |
| **Reliability** | Recover from failure; meet demand; distributed system design | Multi-AZ, Auto Scaling, health checks, chaos engineering |
| **Performance Efficiency** | Use resources efficiently; adapt to demand | Instance types, caching, serverless, Compute Optimizer |
| **Cost Optimisation** | Eliminate waste; appropriate consumption model | Savings Plans, Spot, Graviton, lifecycle policies |
| **Sustainability** | Minimise environmental impact | Graviton, Fargate, efficient storage, rightsizing |

**Well-Architected Tool:** Submit workload details → answer questions per pillar → receive high-risk (HRI) and medium-risk findings → generate improvement plans. Can be used across org via delegated admin.

**Priority of remediation:** Fix HRIs (High Risk Items) first. In exam scenarios, the "most impactful improvement" is almost always addressing an HRI in the highest-weight pillar for the stated scenario.

---

### Cost Allocation Tags and FinOps

**Tag strategy:**
| Tag key | Example values | Purpose |
|---------|---------------|---------|
| `Environment` | production, staging, dev | Cost by environment |
| `Team` | payments, checkout, platform | Cost by team |
| `CostCentre` | CC-1234, CC-5678 | Showback/chargeback |
| `Application` | order-service, api-gateway | Cost by application |

**SCP enforcement of tags:**
```json
{
  "Sid": "DenyWithoutRequiredTags",
  "Effect": "Deny",
  "Action": ["ec2:RunInstances", "rds:CreateDBInstance"],
  "Resource": "*",
  "Condition": {
    "Null": {
      "aws:RequestTag/Environment": "true",
      "aws:RequestTag/CostCentre": "true"
    }
  }
}
```

**Budgets:** Set cost or usage budgets per service, linked account, tag, or purchase type. Trigger: notify via SNS, or invoke a Lambda or SSM OpsItem. Budget actions can stop EC2/RDS or apply SCPs automatically when thresholds breach.

---

## Phase 3 — Key Decision Framework

### "Performance problem — where to look first?"

```
High latency for users?
  → Check CloudFront — serving static content from origin? Add CloudFront caching.

High database CPU?
  1. Add Aurora Read Replicas (read scalability)
  2. Add ElastiCache (reduce DB queries entirely)
  3. Consider read replica offload for reporting queries
  4. Only then: vertical scaling

High Lambda cold start?
  → Enable Provisioned Concurrency for latency-critical functions

EC2 over-provisioned?
  → Compute Optimizer → right-size or switch instance family (Graviton)

S3 upload slow from distant clients?
  → S3 Transfer Acceleration
```

### "Security gap — which service?"

```
Misconfigured resources (open S3 bucket, no MFA)?
  → AWS Config rules + Security Hub

Malware / C2 communication / credential compromise?
  → GuardDuty

CVEs in EC2 or container images?
  → Inspector v2

PII / sensitive data in S3?
  → Macie

L7 attacks (SQL injection, XSS)?
  → WAF (attached to CloudFront or ALB)

DDoS?
  → Shield Advanced + WAF rate limiting

Database credential rotation?
  → Secrets Manager with rotation

Centralised secrets management?
  → Secrets Manager (rotating) or Parameter Store SecureString (static)
```

---

## Phase 3 — Exam Trap Summary

| Trap | Correct Answer |
|------|---------------|
| "KMS key policy without root delegation — can IAM control the key?" | NO. Without the root account statement in the key policy, IAM cannot grant access |
| "Secrets Manager vs Parameter Store for DB password" | Secrets Manager (supports native DB rotation) |
| "Shield Standard covers what?" | Basic L3/L4 (SYN floods, UDP reflection) — automatic, everyone |
| "GuardDuty requires Flow Logs/CloudTrail to be enabled separately?" | NO. GuardDuty accesses these sources independently |
| "Compute Savings Plan applies to Fargate?" | YES. Compute SP covers EC2 + Fargate + Lambda |
| "EC2 Instance Savings Plan covers Lambda?" | NO. Only EC2 instances (specific family in a region) |
| "RDS read replica synchronous?" | NO. Read replicas are always asynchronous; Multi-AZ is synchronous |
| "Spot instance 2-minute warning — how to receive?" | EventBridge EC2 Spot Interruption Warning event (recommended) OR poll IMDS |
| "Standard RI AZ-scoped benefit" | Reserves capacity in that specific AZ |
| "Network Firewall vs WAF — what's the difference?" | Network Firewall: L3/L7 at VPC perimeter (any protocol); WAF: L7 HTTP only at edge/ALB |
| "CloudFront Functions vs Lambda@Edge — network access" | CloudFront Functions have NO network access; Lambda@Edge can call external APIs |
