# SAP-C02 Phase 2 — Deep Knowledge Reference
## Domain 2: Design for New Solutions (Weeks 5–7)

> This document provides the detailed technical knowledge behind every bullet in the Phase 2 study plan.
> Cross-reference: [`aws-solutions-architect-professional.md`](./aws-solutions-architect-professional.md)

---

## Week 5 — Resilience and Multi-Region Architecture

### Disaster Recovery Tiers — RTO/RPO Framework

The four DR tiers represent a cost-vs-recovery-speed trade-off. Every SAP-C02 DR question requires you to select the correct tier based on the stated RTO/RPO.

| Tier | RTO | RPO | Cost | Key Pattern |
|------|-----|-----|------|-------------|
| **Backup & Restore** | Hours–days | Hours | Lowest | Snapshots/backups to S3; restore from scratch on failure |
| **Pilot Light** | 10–30 min | Minutes | Low | Core DB replicated; compute shut off; scale up and start on failover |
| **Warm Standby** | Minutes | Seconds–minutes | Medium | Scaled-down but running replica; scale up on failover |
| **Multi-Site Active/Active** | Near-zero | Near-zero | Highest | Full-capacity running in both regions simultaneously |

**Exam pattern:** The question will give you RTO and RPO requirements + a cost constraint. Map requirements:
- RTO > 1 hour → Backup & Restore is acceptable
- RTO 15–60 min → Pilot Light
- RTO 5–15 min → Warm Standby
- RTO < 5 min → Active/Active (or Warm Standby with automation)
- "Least cost" + "some downtime acceptable" → Always Backup & Restore

**Pilot Light minimum components to keep running:**
- Database (replicated from primary region)
- DNS (Route 53 records ready to be updated)
- AMIs or container images available in DR region

**Warm Standby minimum components to keep running:**
- Scaled-down EC2/ECS running behind a load balancer
- DB replica promoted automatically on failover
- Route 53 health-check-triggered failover routing

---

### Route 53 Routing Policies

| Policy | Use case | Health checks |
|--------|----------|--------------|
| **Simple** | Single resource, no health checks | No |
| **Weighted** | A/B testing; canary deployments | Optional |
| **Latency** | Route users to lowest-latency region | Optional |
| **Failover** | Active-passive DR; primary + standby | Required on primary |
| **Geolocation** | Route by user's country/continent | Optional |
| **Geoproximity** | Route by geographic distance + bias | Optional |
| **Multivalue** | Up to 8 healthy IPs returned; client-side LB | Optional |
| **IP-based** | Route by source IP prefix (CIDR block) | Optional |

**Geolocation vs Geoproximity:**
- **Geolocation**: Routes based on the country/continent of the DNS query origin. Records must be mapped to geographic locations. A "default" record is required for unmatched locations.
- **Geoproximity**: Routes to the nearest resource by geographic distance. You can apply a **bias** to expand or shrink the effective routing radius of a resource.

**Health checks:**
- Monitor an endpoint (IP/hostname) or another Route 53 health check (calculated health check).
- Calculated health check: define as healthy when X of N child checks are healthy.
- You can health-check CloudWatch alarms (for non-HTTP endpoints like DynamoDB).

**Failover routing:**
- PRIMARY record has a health check. If the health check fails, Route 53 serves the SECONDARY record.
- Both records can have health checks.

---

### Global Accelerator vs CloudFront

| Criterion | Global Accelerator | CloudFront |
|-----------|-------------------|------------|
| **Content type** | Dynamic, non-cacheable (APIs, TCP/UDP apps) | Cacheable static/dynamic web content |
| **Layer** | Network (Layer 4) | Application (Layer 7) |
| **Protocol** | TCP, UDP | HTTP/HTTPS |
| **Anycast IPs** | Two static anycast IPs | No static IPs; uses DNS |
| **Routing** | Health-aware routing to healthy endpoints | Cache hit/miss routing to origin |
| **DDoS protection** | Shield Standard included | Shield Standard included |
| **Latency improvement** | Routes via AWS global network from edge | Serves cached content from edge |

**When to use Global Accelerator:**
- Gaming (UDP), VoIP, IoT — non-HTTP protocols
- APIs where all requests must hit origin (no caching possible)
- You need static IPs (cannot use CloudFront which uses DNS)
- Multi-region active-active with health-aware failover at network layer

**When to use CloudFront:**
- Static assets (S3 origin)
- Web applications where caching reduces origin load
- Lambda@Edge or CloudFront Functions for edge-side logic

---

### Aurora Global Database

Aurora Global Database replicates across up to **5 secondary regions** with typically **< 1 second replication lag**.

**Architecture:**
```
Primary Region (eu-west-1)
├── Aurora Writer (one primary cluster)
└── Aurora Readers (within region)
    ↓ async replication (< 1 second lag)
Secondary Region (us-east-1)
└── Aurora Reader (read-only; can be promoted to writer on failover)
```

**Write forwarding:** Enabled on secondary clusters. Allows applications in the secondary region to issue writes to the secondary cluster's endpoint. Writes are transparently forwarded to the primary region's writer. Latency = round-trip to primary. Not suitable for latency-sensitive write paths.

**Promotion (failover) process:**
1. Detach secondary cluster from the Global Database.
2. Secondary cluster becomes a standalone Aurora writer.
3. Update application connection strings to point to new writer endpoint.
4. (After primary recovers) Re-add the old primary as a secondary.

**RTO for Global Database failover:** Typically < 1 minute (managed promotion) or < 60 seconds (manual promotion).

**Compared to Aurora cross-region Read Replicas:**
- Global Database uses dedicated replication infrastructure (not binlog).
- Replication lag typically 10–100 ms vs seconds for standard read replicas.
- Single failover action; no need to re-establish replication.

---

### DynamoDB Global Tables

DynamoDB Global Tables provide **multi-region, multi-master** (active-active) replication.

| Property | Detail |
|----------|--------|
| Replication | Asynchronous, typically < 1 second lag |
| Writes | Can write in ANY replica region — all are active writers |
| Conflict resolution | Last-writer-wins based on timestamp |
| Consistency | Strong consistency within a region; eventual across regions |
| Streams | DynamoDB Streams must be enabled (required for replication) |

**Global Tables vs Aurora Global Database for the exam:**

| Scenario | Choose |
|----------|--------|
| Relational schema (joins, transactions, complex queries) | Aurora Global Database |
| Key-value / document at massive scale, multi-region writes needed | DynamoDB Global Tables |
| RPO = zero, multi-region active writes | DynamoDB Global Tables |
| Read-heavy with < 1s replication lag tolerable | Aurora Global Database |

---

### S3 Cross-Region Replication (CRR)

| Property | Detail |
|----------|--------|
| Direction | One source bucket → one or more destination buckets |
| Objects replicated | New objects only (after replication is enabled); existing objects need S3 Batch Replication |
| Ownership | Can change object ownership to destination account |
| RTC (Replication Time Control) | SLA: 99.99% of objects replicated in 15 minutes; CloudWatch metrics available |
| Bi-directional | Available — enables active-active S3 across regions |
| Delete markers | NOT replicated by default; opt-in |

---

## Week 6 — Compute, Serverless, and Event-Driven Architecture

### AWS Lambda — Advanced Patterns

**Concurrency model:**

| Type | Description | Cold start behaviour |
|------|-------------|---------------------|
| **Unreserved** | Lambda uses shared account concurrency pool | Yes, on scale-out |
| **Reserved** | Dedicated concurrency for this function; cannot be used by others | Yes, on scale-out |
| **Provisioned** | Pre-warmed execution environments | NO cold starts |

- Default account concurrency limit: **1,000 concurrent executions** (soft limit).
- Reserved concurrency: cap a function's concurrent executions to prevent it monopolising the account pool. Also guarantees a floor.
- Provisioned concurrency: pre-warms N execution environments. Allocated from reserved concurrency.

**Throttling behaviour:**
- If reserved concurrency is reached → `429 TooManyRequests`.
- If account-level concurrency is reached → other functions are throttled.
- SQS trigger: on throttle, SQS holds messages and retries — no messages lost.
- Kinesis trigger: on throttle, iterator falls behind — shard processing blocked.

**Async invocation error handling:**

```
Lambda async invocation
  → Success: done
  → Failure: retried automatically up to 2 times (3 total attempts)
  → After all retries fail:
      → DLQ (SQS or SNS) if configured
      → OR Lambda Destinations (success/failure destination: SQS, SNS, EventBridge, another Lambda)
```

**Lambda Destinations** (newer, preferred over DLQ):
- Can route to different destinations on success vs failure.
- Includes the full invocation result in the destination payload.

**SnapStart (Java only):**
- Lambda takes a snapshot of an initialized execution environment.
- New invocations restore from the snapshot → sub-second init time.
- Use `@HandlerClass` and `CacheClient` patterns to avoid restoring stale connections.

**Lambda response streaming:**
- Stream response back to the caller progressively rather than waiting for the full response.
- Increases effective payload limit (6 MB buffered → 20 MB streamed).
- Use case: streaming LLM tokens, large file generation.

---

### AWS Step Functions

**Standard vs Express Workflows:**

| Property | Standard | Express |
|----------|----------|---------|
| Duration | Up to 1 year | Up to 5 minutes |
| Execution model | At-least-once | At-least-once (async) / at-most-once (sync) |
| Pricing | Per state transition | Per GB-second + per execution |
| Audit | Full history in console | CloudWatch Logs only |
| Use case | Long-running orchestration, human approval | High-volume, event-driven, short workflows |

**State types:**

| State | Purpose |
|-------|---------|
| Task | Invoke a Lambda, call an AWS API (SDK integration), call HTTP endpoint |
| Choice | Branch based on conditions |
| Wait | Pause execution for a duration or until a timestamp |
| Parallel | Run branches concurrently; collect all results before continuing |
| Map | Iterate over an array; each item processed by a sub-workflow |
| Pass | Pass input to output (transformation or injection) |
| Succeed / Fail | Terminal states |

**SDK integrations:** Step Functions can directly call 200+ AWS APIs without a Lambda wrapper:
```json
{
  "Type": "Task",
  "Resource": "arn:aws:states:::dynamodb:putItem",
  "Parameters": {
    "TableName": "Orders",
    "Item": {"orderId": {"S.$": "$.orderId"}}
  }
}
```

**Error handling:**
```json
"Catch": [{
  "ErrorEquals": ["Lambda.ServiceException", "States.TaskFailed"],
  "Next": "HandleError",
  "ResultPath": "$.error"
}],
"Retry": [{
  "ErrorEquals": ["Lambda.TooManyRequestsException"],
  "MaxAttempts": 3,
  "BackoffRate": 2,
  "IntervalSeconds": 1
}]
```

---

### ECS vs EKS Decision Criteria

| Criterion | ECS (Fargate) | EKS |
|-----------|---------------|-----|
| Kubernetes experience required | No | Yes |
| Operational complexity | Low (AWS-managed) | High (control plane managed, data plane not fully) |
| Scheduling control | Limited | Full (custom schedulers, CRDs, operators) |
| Ecosystem | AWS-native integrations | Entire CNCF ecosystem |
| Cost (compute) | Per task CPU/memory | Per node + EKS cluster fee ($0.10/hour) |
| Use for exam | "Least operational overhead containers" | "Kubernetes required" or "existing K8s workload migration" |

**When ECS wins:** Team has no Kubernetes expertise; need fastest path to containers; simpler workloads; smallest operational overhead.

**When EKS wins:** Existing Kubernetes manifests or operators; complex scheduling requirements; need Prometheus/Grafana native integration; service mesh (Istio/Linkerd).

**Fargate vs EC2 backing (both ECS and EKS):**

| Fargate | EC2 nodes |
|---------|-----------|
| No node management | Must manage EC2 fleet, patches, SSM |
| Higher per-task cost | Lower cost at scale (Graviton Spot) |
| Cannot run privileged containers | Can run DaemonSets, node-level tools |
| Karpenter not applicable | Karpenter for efficient auto-scaling |

---

### Messaging — SQS, SNS, EventBridge, Kinesis

**SQS — Standard vs FIFO:**

| Property | Standard | FIFO |
|----------|----------|------|
| Ordering | Best effort | Strict FIFO per MessageGroupId |
| Delivery | At-least-once | Exactly-once (within 5-minute deduplication window) |
| Throughput | Unlimited | 300 TPS (3,000 with batching) |
| Deduplication | No | Yes (content-based or MessageDeduplicationId) |
| Use case | Decoupling, high throughput | Financial transactions, ordering-sensitive operations |

**Key SQS concepts:**
- **Visibility timeout:** After a consumer reads a message, it becomes invisible for this duration. If not deleted before timeout expires, it reappears. Default 30 seconds; max 12 hours.
- **Long polling:** Consumer waits up to 20 seconds for a message rather than returning immediately if the queue is empty. Reduces empty receive calls and cost.
- **Dead Letter Queue (DLQ):** Messages that fail processing N times (`maxReceiveCount`) are moved to the DLQ. Use with CloudWatch alarm on `ApproximateNumberOfMessagesVisible` in DLQ.
- **Message retention:** 4 days default; 14 days maximum.

**Lambda trigger from SQS FIFO:**
- Lambda processes one MessageGroupId at a time (preserves ordering).
- `ReportBatchItemFailures` response format — Lambda reports partial batch failures so only failed messages go back to the queue.

---

**Kinesis Data Streams vs Kinesis Firehose:**

| Property | Kinesis Data Streams | Kinesis Firehose |
|----------|---------------------|-----------------|
| Consumers | Custom (KCL, Lambda, Flink) | Managed delivery only |
| Destinations | Any (you write consumer) | S3, Redshift, OpenSearch, Splunk, HTTP |
| Latency | Milliseconds | 60 seconds minimum (buffer) |
| Ordering | Per shard | No ordering guarantee |
| Retention | 24 hours (default); up to 7 days (365 days extended) | No retention — delivery only |
| Scaling | Manual (add/remove shards) or On-Demand mode | Automatic |
| Use case | Real-time processing, custom consumers | Near-real-time delivery to storage/analytics |

**Shard capacity:** Each shard handles:
- **1 MB/s ingest** (1,000 records/s at 1 KB each)
- **2 MB/s read** per consumer (classic); **2 MB/s per consumer** with Enhanced Fan-Out (parallel consumers, dedicated throughput)

**Enhanced Fan-Out:** Each registered consumer gets a dedicated 2 MB/s read bandwidth. Without it, all consumers share 2 MB/s per shard. Use for multiple consumers needing full throughput.

**Kinesis vs SQS decision:**

| Kinesis | SQS |
|---------|-----|
| Ordering within shard guaranteed | FIFO only; no ordering in Standard |
| Multiple consumers of same stream (fan-out) | One logical consumer per queue (fan-out needs SNS→multiple SQS) |
| Replay data within retention period | No replay — message deleted after consumption |
| Real-time analytics | Decoupling tasks |

---

**EventBridge:**

- **Event bus:** Default bus (all AWS service events), custom buses (your events), partner buses (SaaS events).
- **Rules:** Match events via content-based filtering → route to targets.
- **Pipes:** Point-to-point integration with optional filtering, enrichment (Lambda), and transformation between source and target.
- **Scheduler:** Cron or rate-based schedule → invoke any target. Replaces CloudWatch Events for scheduling.
- **Schema registry:** Discovers event schemas from events on the bus; generates code bindings.

**SNS Fan-out pattern:**
```
Producer → SNS Topic
             ├── SQS Queue A → Lambda (process order)
             ├── SQS Queue B → Lambda (send email)
             └── SQS Queue C → Lambda (update analytics)
```
Why SQS between SNS and Lambda? SQS buffers; Lambda can throttle/retry without losing messages. Direct SNS → Lambda has limited retry on throttle.

---

### SAGA Pattern for Distributed Transactions

In microservices, a SAGA replaces a traditional 2-phase commit (2PC) which doesn't scale across services.

**Choreography-based SAGA (events):**
```
Order Service → emits OrderCreated event
  → Inventory Service → reserves stock → emits StockReserved
    → Payment Service → charges card → emits PaymentProcessed
      → Order Service → confirms order

On failure:
  Payment fails → emits PaymentFailed
    → Inventory Service → emits StockReleased (compensating transaction)
      → Order Service → cancels order
```

**Orchestration-based SAGA (Step Functions):**
```
Step Functions workflow:
  1. Reserve Inventory          (Task)
  2. Process Payment            (Task)
     ↓ failure
  3. Compensate: Release Inventory (Task - rollback)
  4. Cancel Order               (Task - rollback)
```

Orchestration is preferred for complex, long-running transactions where visibility and auditability matter — Step Functions provides this.

---

## Week 7 — Data Architecture and Storage

### RDS Advanced Patterns

**Multi-AZ vs Read Replica:**

| Property | Multi-AZ | Read Replica |
|----------|----------|-------------|
| Purpose | High availability | Read scaling + reporting |
| Replication | Synchronous | Asynchronous |
| Standby readable | NO (standby is not accessible) | YES (read traffic) |
| Failover | Automatic (~60–120s) | Manual promotion |
| Cross-region | Yes (with MLMA for Aurora) | Yes (cross-region read replica) |

**RDS Proxy:**
- Maintains a warm pool of database connections.
- Applications connect to the Proxy; proxy multiplexes over fewer real DB connections.
- Critical for Lambda → RDS patterns: Lambda can create thousands of connections; RDS has per-instance connection limits (~300–5,000 depending on instance class).
- Supports IAM database authentication and Secrets Manager auto-rotation.

**Read Replica as cross-region DR:** You can promote a read replica in a secondary region to a standalone database — this is a valid low-cost DR strategy for RTO of ~10–30 minutes.

---

### Aurora Architecture Deep Dive

**Storage layer:** Aurora decouples storage from compute. The storage layer is a distributed, SSD-backed, self-healing storage system that spans 3 AZs (6 copies across 2 AZs minimum for writes, 3 AZs for durability). The cluster volume automatically grows in 10 GB increments up to 128 TB.

**Aurora Serverless v2:**
- Scales in ACU (Aurora Capacity Unit) increments of 0.5.
- Scales in seconds (not minutes like v1).
- Can scale to zero after inactivity (cold start penalty).
- Use for: dev/test, unpredictable spikes, multi-tenant SaaS with variable load per tenant.

**Aurora Parallel Query:**
- Pushes query processing down to the storage layer nodes.
- Reduces data transfer between storage and head node.
- Improves performance for analytical queries on large tables.
- Compatible with MySQL 5.6/5.7.

**Babelfish:** Aurora PostgreSQL feature that understands T-SQL (SQL Server syntax). Allows SQL Server applications to connect to Aurora PostgreSQL without rewriting queries. Useful for migration from SQL Server.

---

### DynamoDB — Partition Key Design and Advanced Patterns

**Partition key design rules:**
- Choose a key with **high cardinality** (many distinct values).
- Distribute reads/writes evenly — avoid "hot" partitions.
- Each partition: 10 GB max storage, 3,000 RCU, 1,000 WCU.

**Anti-patterns:**

| Anti-pattern | Problem | Solution |
|-------------|---------|---------|
| Date as partition key | All writes to same partition on same day | Add suffix: `date#random_id` |
| Small-lookup enum (status: pending/active) | Too few partitions | Composite key or write sharding |
| Sequential counter | All writes to one partition | UUID or timestamp with random suffix |

**Write sharding for hot keys:**
```python
# Instead of partition key = "PRODUCT_001"
# Use: partition key = "PRODUCT_001#" + random.randint(0, 9)
# When reading: query all 10 shards and merge results
```

**GSI vs LSI:**

| Property | LSI (Local Secondary Index) | GSI (Global Secondary Index) |
|----------|----------------------------|------------------------------|
| Partition key | Same as base table | Any attribute |
| Sort key | Different attribute | Any attribute |
| Created | At table creation only | Any time |
| Scope | Items with same partition key | Entire table |
| Consistency | Strong or eventual | Eventual only |
| Capacity | Shared with base table | Separate WCU/RCU |

**DynamoDB Streams + Lambda:** Captures every item-level change as a stream of records. Lambda polls the stream. Use for: change data capture, cross-table updates, event sourcing, search index sync (to OpenSearch).

**DAX (DynamoDB Accelerator):**
- In-memory write-through cache for DynamoDB.
- Reduces read latency from single-digit milliseconds to microseconds.
- Compatible with DynamoDB API (drop-in; no code changes except endpoint).
- Best for: read-heavy, cache-friendly workloads. NOT suitable for: strongly consistent reads (DAX returns cached data), write-heavy workloads, financial ledgers.

---

### ElastiCache Patterns

**Redis vs Memcached:**

| Feature | Redis | Memcached |
|---------|-------|-----------|
| Data structures | Strings, hashes, lists, sets, sorted sets, streams | Strings only |
| Persistence | Yes (RDB snapshots, AOF) | No |
| Replication | Yes (primary/replica) | No |
| Cluster mode | Sharded cluster (read/write scaling) | Sharding (but simpler) |
| Pub/Sub | Yes | No |
| Lua scripting | Yes | No |
| Multi-thread | No | Yes |
| Use for | Sessions, leaderboards, queues, rate limiting | Simple high-throughput caching |

**Redis Cluster Mode Enabled:**
- Data is sharded across multiple node groups (shards).
- Each shard has a primary + up to 5 replicas.
- Scales both read and write throughput.
- Key constraint: all keys in a pipeline must map to the same shard (use hash tags `{user}:session` to force co-location).

**Caching strategies:**

| Strategy | How it works | Pros | Cons |
|----------|-------------|------|------|
| **Lazy loading** | Check cache → miss → fetch from DB → populate cache | Only caches needed data | Cache miss = 3 round trips; stale data possible |
| **Write-through** | Write to DB → immediately write to cache | Always fresh | Caches data that may never be read; extra write |
| **Write-behind** | Write to cache → async write to DB | Low write latency | Risk of data loss on cache failure |
| **TTL** | Set expiration on cached items | Controls staleness | Choose TTL based on acceptable staleness |

---

### Amazon Redshift

**Node types:**

| Type | Use case |
|------|---------|
| **RA3** | Managed storage (S3-backed); pay for compute and storage separately; can scale compute without migrating data |
| **DC2** | Dense compute (local SSD); fixed storage; legacy; lower cost for small clusters |

**Redshift Spectrum:** Run queries against data in S3 directly from Redshift without loading. Uses external tables defined in the Glue Data Catalog. Pushes query predicates to S3 — only reads relevant data.

**Concurrency Scaling:** Automatically adds transient Redshift clusters during peak load. Queries are routed to transient clusters transparently. First 1 hour/day per cluster is free; charged per second after that.

**Data Sharing (cross-cluster data sharing):**
- Producer cluster exposes a datashare.
- Consumer cluster (different accounts/regions possible) queries the datashare as if local.
- No data movement — queries against producer's storage.
- Use for: analytics isolation, separation of read and write workloads.

---

### S3 Advanced Features

**Storage class selection:**

| Class | Access pattern | Min duration | Retrieval fee |
|-------|---------------|-------------|--------------|
| Standard | Frequent | None | None |
| Standard-IA | Infrequent (once/month) | 30 days | Yes |
| One Zone-IA | Infrequent, non-critical | 30 days | Yes |
| Glacier Instant | Archive, access < 5ms | 90 days | Yes |
| Glacier Flexible | Archive, access 1–5 hours | 90 days | Yes |
| Glacier Deep Archive | Archive, access 12–48 hours | 180 days | Yes |
| Intelligent-Tiering | Unknown access pattern | None | None (monitoring fee) |

**S3 Object Lambda:** Creates a Lambda function that transforms S3 GET responses on the fly. Example: serve redacted PII data from the same S3 bucket without storing two copies.

```
Client → S3 Object Lambda Access Point → Lambda (transform) → S3 GET → transformed response
```

**Byte-range fetches:** Download specific byte ranges of a large object in parallel. Use to implement parallel download, resume downloads, or read file headers only.

**Multipart upload:**
- Required for objects > 5 GB.
- Recommended for objects > 100 MB.
- Benefits: parallel upload, resume on failure, transfer throttling.

---

### Data Lake Pattern (S3 + Glue + Athena + Lake Formation)

```
Ingest → S3 Raw Zone (parquet or CSV)
  → Glue Crawler → Glue Data Catalog (schema discovery)
    → Glue ETL Job → S3 Curated Zone (parquet, partitioned)
      → Athena → SQL queries on S3 (pay per TB scanned)
        → Lake Formation → fine-grained column/row-level permissions
```

**Lake Formation** adds fine-grained access control on top of S3. You can grant column-level, row-level, and cell-level permissions via the catalog — without managing S3 bucket policies for each table.

---

## Phase 2 — Key Decision Framework

### "Which DB for this use case?" Decision Tree

```
Need: SQL + transactions + ACID compliance?
  → RDS or Aurora

Need: SQL + global active-active writes + relational?
  → No good AWS option; reconsider schema or use DynamoDB

Need: Key-value at any scale + single-digit ms latency?
  → DynamoDB

Need: Key-value at any scale + sub-ms + read-heavy?
  → DynamoDB + DAX

Need: Global active-active at any scale?
  → DynamoDB Global Tables

Need: Multi-region SQL with < 1s replication lag?
  → Aurora Global Database

Need: OLAP + complex SQL + data warehouse?
  → Redshift

Need: Full-text search or vector search?
  → OpenSearch Service

Need: Graph relationships?
  → Neptune

Need: Time-series IoT or financial ticks?
  → Timestream
```

### "Which messaging service?" Decision Tree

```
Need: Real-time stream processing + multiple consumers + replay?
  → Kinesis Data Streams

Need: Near-real-time delivery to S3/Redshift/OpenSearch (no custom consumer)?
  → Kinesis Firehose (Delivery Stream)

Need: Decoupling + retry + DLQ + task queues?
  → SQS (Standard for throughput; FIFO for ordering + deduplication)

Need: Fan-out to multiple subscribers?
  → SNS (push) or EventBridge (content-based routing)

Need: Event-driven workflow orchestration with visibility?
  → EventBridge + Step Functions

Need: Managed Kafka (existing Kafka producers/consumers)?
  → MSK (Managed Streaming for Apache Kafka)
```

---

## Phase 2 — Exam Trap Summary

| Trap | Correct Answer |
|------|---------------|
| "Aurora Multi-AZ standby — can you read from it?" | NO. Standby is not accessible. Add Aurora Readers for read scaling |
| "Lambda concurrency 0 — what does reserved=0 do?" | Sets reserved concurrency to 0 = throttles ALL invocations (useful to disable a function) |
| "DynamoDB strong consistency on GSI" | NOT possible. GSIs support eventual consistency only |
| "LSI added after table creation" | NOT possible. LSIs must be created at table creation time |
| "Kinesis shard limit exceeded — what happens to Lambda trigger?" | Lambda iterator falls behind; shard processing is blocked (unlike SQS which just retries) |
| "Gateway endpoint (S3) accessible from on-premises via DX?" | NO. Interface endpoint required for on-premises DX/VPN access to S3 |
| "Express Step Functions — execution history in console?" | NO. Only CloudWatch Logs. Standard workflows have full console history |
| "Which DR tier: RPO < 1 min, RTO < 5 min?" | Multi-Site Active/Active OR Warm Standby with full automation |
| "Replication time for CRR with RTC enabled" | 15-minute SLA for 99.99% of objects |
| "DynamoDB Global Tables vs Aurora Global — multi-region writes?" | DynamoDB Global Tables (active-active); Aurora Global is active-passive (promote only) |
