# AWS Migration Tools and Strategies

## Category
Cloud Native, Migration, AWS MGN, DMS, DataSync, Snowball, AWS Transfer Family

## Context

AWS migration is structured around the **7 Rs framework** and a portfolio of specialised tools. The key skill — both in practice and on the SAP-C02 exam — is selecting the correct tool for the scenario: what is being migrated (server, database, files, or a full platform), at what scale, with what latency constraints, and with what acceptable downtime.

---

## The 7 R Migration Strategies

| R | Name | Definition | Indicators |
|---|------|------------|-----------|
| **Retire** | Remove | Decommission — application is no longer needed | 20–30% of portfolios typically; consolidation post-merger |
| **Retain** | Keep | Leave on-premises; defer migration | Compliance; recent hardware investment; too risky now |
| **Rehost** | Lift and shift | Move to AWS as-is; no code or architecture changes | Large server count; fast migration required; risk-averse |
| **Relocate** | Platform lift and shift | Move to AWS without changing operating model (e.g., VMware → VMware Cloud on AWS) | VMware environments; container workloads moving to EKS |
| **Replatform** | Lift, tinker, and shift | Minimal cloud optimisations: managed DB, PaaS | Move MySQL → Aurora; Tomcat → Elastic Beanstalk; JBoss → ECS |
| **Repurchase** | Drop and shop | Replace with SaaS | Legacy CRM → Salesforce; on-prem exchange → Microsoft 365 |
| **Refactor / Re-architect** | Re-architect | Redesign to cloud-native with significant code changes | Scalability requirements; move to serverless or microservices |

**Migration wave planning:**
- Wave 1: Retire candidates + simple Rehost (highest confidence).
- Wave 2: Replatform candidates (managed DBs, containerisation).
- Wave 3+: Refactor candidates (complex; requires planning and development cycles).

---

## AWS Application Migration Service (MGN)

MGN is the **primary, recommended service for Rehost migrations** — it migrates full servers (OS + applications + data) to EC2.

### Architecture

```
Source Server (on-prem or other cloud)
├── MGN Agent installed
│     → Establishes outbound TLS connection to AWS Replication Server (port 443)
│     → Block-level replication begins (port 1500 TCP to Replication Server)
│
└── Continuous block-level replication
        ↓
  Replication Server (t3.small, in Staging Area Subnet)
        ↓
  EBS Volumes (staging copy)
        ↓
  [Test launch] → EC2 instance in Test subnet → validate
        ↓
  [Cutover launch] → EC2 instance in Production subnet → traffic cutover
```

### Key Stages

| Stage | Description |
|-------|-------------|
| **Not ready** | Agent not installed or initial sync not complete |
| **Ready for testing** | Initial replication complete; can launch test instance |
| **Test in progress** | Test EC2 instance running for validation |
| **Ready for cutover** | Tests passed; delta replication continues |
| **Cutover in progress** | Final sync; cutover instance launched; source stopped |
| **Cutover complete** | Migration done; source can be decommissioned |

### Agentless Replication (VMware only)

- Uses vSphere APIs — no agent installation on VMs.
- Requires MGN connector deployed in the VMware environment.
- Snapshots replicated to AWS periodically (near-continuous, but slightly behind agent-based).
- Use when: VM agent installation is not permitted, or vSphere API access is easier.

### MGN vs SMS (Server Migration Service)

MGN is the **replacement** for the older SMS. SMS is deprecated for new migrations. Use MGN for all new server migration projects.

### MGN vs DMS

| | MGN | DMS |
|--|-----|-----|
| What migrates | Full server (OS + apps + data) | Database tables, rows, schema |
| Rehost | Yes | N/A |
| Database migration | No (brings the DB files as-is) | Yes (converts data) |
| Heterogeneous migration | No (same OS) | Yes (Oracle → Aurora PostgreSQL) |
| Downtime requirement | Minimal (cutover window only) | Near-zero (with Full Load + CDC) |

---

## AWS Database Migration Service (DMS)

DMS migrates database schema and data between source and target databases, including heterogeneous migrations (different engine types).

### Replication Instance

A managed EC2 instance that runs the DMS replication software. Sized based on migration throughput needs.

- Place in same VPC and AZ as the target database for lowest latency during migration.
- Multi-AZ replication instance: standby instance in another AZ for HA of the migration process itself.
- Instance types: `dms.t3.medium` to `dms.r6i.8xlarge`.

### Migration Task Types

| Type | What it does | Downtime |
|------|------------|---------|
| **Full Load** | Bulk copy of all existing rows | Application must be stopped during migration |
| **CDC only** | Captures ongoing changes; assumes data already pre-loaded | No downtime (for change sync only) |
| **Full Load + CDC** | Bulk copy first, then switches to CDC | Minimal: stop app after CDC lag reaches near-zero |

### CDC Requirements

| Source DB | Requirement |
|-----------|------------|
| MySQL/Aurora MySQL | Binary logging enabled; `binlog_format = ROW` |
| PostgreSQL | Logical replication enabled; `wal_level = logical`; `max_replication_slots ≥ 1` |
| Oracle | Supplemental logging enabled; LogMiner or Binary Reader mode |
| SQL Server | MS-Replication or MS-CDC enabled |

### Supported Migration Paths (exam-relevant)

| Source | Target | Type |
|--------|--------|------|
| MySQL | Aurora MySQL | Homogeneous |
| PostgreSQL | Aurora PostgreSQL | Homogeneous |
| Oracle | Oracle RDS | Homogeneous |
| SQL Server | SQL Server RDS | Homogeneous |
| Oracle | Aurora MySQL | Heterogeneous (needs SCT) |
| Oracle | Aurora PostgreSQL | Heterogeneous (needs SCT) |
| SQL Server | Aurora PostgreSQL | Heterogeneous (needs SCT) |
| MongoDB | DocumentDB | Homogeneous (DMS native) |

### Schema Conversion Tool (SCT)

SCT is a **standalone desktop application** (runs on your local machine) that:
1. Connects to source and target databases.
2. Converts DDL (tables, views, indexes) + stored procedures, functions, triggers.
3. Produces an assessment report: what is auto-converted vs what requires manual intervention.
4. Outputs converted SQL scripts for the target dialect.

**SCT assessment categories:**
- **Automatically converted:** No action needed.
- **Manually converted with simple changes:** Minor syntax differences.
- **Manually converted with complex changes:** Significant rewrite; flag for developer effort.

**SCT is only required for heterogeneous migrations.** For MySQL → Aurora MySQL, SCT is not needed (schema is compatible).

### DMS Data Validation

After migration, DMS can run a validation task to compare row counts and a sampled comparison of data between source and target. Use to ensure migration completeness.

### DMS Serverless

DMS Serverless automatically provisions and scales replication capacity. No need to choose or manage a replication instance. Use for: unpredictable load, variable data velocity, cost-optimised migrations.

---

## AWS DataSync

DataSync is a **managed file and object transfer service** for moving data at scale between on-premises storage and AWS storage, or between AWS storage services.

### Supported Endpoints

**Sources:**
- NFS v3/v4 (on-premises NAS, Linux share)
- SMB 2/3 (Windows file server)
- HDFS (Hadoop)
- Object storage (NFS-compatible)
- AWS storage (S3, EFS, FSx — for cross-service/region/account transfers)

**Destinations:**
- Amazon S3 (all storage classes)
- Amazon EFS
- FSx for Windows File Server
- FSx for Lustre
- FSx for NetApp ONTAP
- FSx for OpenZFS

### DataSync Agent

An agent (virtual machine) deployed in your on-premises environment or in an AWS Direct Connect-connected data centre. The agent handles:
- Connecting to your NFS/SMB shares.
- Optimising data transfer (parallel streams, compression).
- Sending data to AWS (encrypted in transit via TLS).

For transfers entirely within AWS (e.g., S3 → EFS cross-region), no agent is needed.

### Key Features

| Feature | Detail |
|---------|--------|
| **Data integrity** | Checksums computed at source and verified at destination |
| **Bandwidth throttling** | Configurable max rate (GB/hour) |
| **Scheduling** | One-time or recurring (hourly/daily/weekly/monthly) |
| **Filtering** | Include/exclude patterns by file name or path |
| **Preservation** | Preserves file metadata: permissions, timestamps, ownership |
| **Encryption** | TLS in transit; destination encryption per storage class |

### DataSync Throughput

DataSync uses multiple parallel streams. Practical throughput depends on source system IOPS, network bandwidth, and file size distribution (many small files = slower than few large files).

Rough guideline: `~10 TB per day on a 1 Gbps connection with typical file sizes`.

### DataSync vs Storage Gateway vs Snowball

| | DataSync | Storage Gateway | Snowball |
|--|----------|----------------|---------|
| Transfer type | Bulk, scheduled, one-time | Ongoing hybrid access | Physical offline |
| Access model | Push (transfer and done) | Persistent mount (NFS/SMB to S3) | Ship device |
| Network requirement | Reliable connectivity | Ongoing connectivity | No (or minimal) connectivity |
| Data integrity | Yes (checksums) | Yes | Yes |
| Use case | Migration; large bulk transfer | Hybrid: on-prem apps need S3 | > 10 TB + poor connectivity |

---

## AWS Snow Family

### Devices

| Device | Usable Storage | Compute | Use case |
|--------|---------------|---------|---------|
| **Snowcone HDD** | 8 TB | 2 vCPUs, 4 GB | Edge; smallest form factor; IoT/tactile |
| **Snowcone SSD** | 14 TB | 2 vCPUs, 4 GB | Edge; slightly larger |
| **Snowball Edge Storage Optimised** | 80 TB | 40 vCPUs, 80 GB RAM | Large-scale data migration |
| **Snowball Edge Compute Optimised** | 28 TB NVMe | 104 vCPUs, 416 GB RAM; GPU optional | ML inference, video processing at edge |
| **Snowmobile** | 100 PB per vehicle | Custom | Exabyte-scale migration |

### Data Migration Decision by Volume

```
Volume < 1 TB and good connectivity → DataSync or simple S3 CLI upload
Volume 1–10 TB and decent connectivity → DataSync (with agent)
Volume > 10 TB and poor connectivity → Snowball Edge Storage Optimised
Volume > 10 PB → Multiple Snowball Edge (clustered up to 10) or Snowmobile
```

**Throughput comparison to internet transfer:**
```
1 Gbps = 10 TB/day → 100 TB takes 10 days
If 100 TB migration needs to complete in 1 week → borderline; DataSync possible
If 500 TB → 50 days → Snowball (ship in 1–2 weeks, faster overall)
```

### Edge Computing (Snowball Compute Optimised)

Beyond data migration, Snowball Edge can run EC2 instances and Lambda functions at the edge (no AWS connectivity required). Use cases:
- ML inference in factories, oil rigs, ships.
- Video analytics in field locations.
- Local compliance: data never leaves the facility.

### Snowball Process

1. **Order** the device from AWS Console.
2. **Ship** arrives at your datacentre (pre-configured with your AWS account and S3 bucket target).
3. **Transfer** data using Snowball client or S3 adapter.
4. **Ship back** to AWS.
5. **AWS ingests** data to your designated S3 bucket.
6. Device is securely wiped (NIST 800-88 standard).

---

## AWS Transfer Family

Transfer Family provides **managed SFTP, FTPS, FTP, and AS2 protocol endpoints** backed by S3 or EFS — without managing file transfer servers.

### Protocol Support

| Protocol | Use case |
|----------|---------|
| **SFTP** | Secure, SSH-based; most common for partner integrations |
| **FTPS** | FTP with TLS; legacy systems requiring FTP-family |
| **FTP** | Unencrypted; only use within private VPCs |
| **AS2** | EDI (Electronic Data Interchange) for B2B transactions |

### Identity Providers

| Provider | How |
|---------|-----|
| **Service-managed** | Usernames/keys stored in Transfer Family (built-in) |
| **AWS Managed AD** | Active Directory users authenticate |
| **Custom Lambda** | API Gateway + Lambda → validate credentials against any external system |

### Use Case

Lift-and-shift of managed file transfer infrastructure:
- Trading partners currently SFTP to your on-premises server → after migration they SFTP to the same hostname (DNS update), files land in S3.
- No changes required on the partner side if hostname and credentials remain the same.

### Custom Identity Provider Pattern

```
Partner SFTP client
  → Transfer Family endpoint
    → Calls Amazon API Gateway (Lambda authoriser)
      → Lambda validates user against your custom identity store (LDAP, database)
        → Returns IAM role ARN for this user's S3/EFS access
          → Transfer Family assumes the role → scoped access to S3 prefix or EFS directory
```

This allows per-user isolation (each user can only see their folder in S3).

---

## AWS Migration Hub

Migration Hub provides a **central tracking dashboard** across migration tools (MGN, DMS, and partner tools).

### Application Discovery Service

Two modes for discovering on-premises servers:

| Mode | How | What it captures |
|------|-----|-----------------|
| **Agent-based** | Lightweight agent on each server | CPU, memory, disk, network I/O; installed software; running processes |
| **Agentless** | VMware vCenter connector | VM inventory, utilisation from vCenter API; no OS-level detail |

Discovery data → Migration Hub → Helps build dependency groups and migration waves.

### Migration Hub Strategy Recommendations

Analyses discovered server data and workload patterns to recommend a migration strategy (7 R) for each application. Uses ML to identify:
- Applications that can be rehosted immediately.
- Applications with database complexity requiring DMS.
- Applications benefiting from containerisation.

---

## Pros

- **MGN:** Agent-based continuous replication means near-zero downtime cutover; no data loss risk; supports most OS types.
- **DMS:** Supports virtually all popular database engines; Full Load + CDC enables near-zero-downtime database migrations; heterogeneous migration without manual ETL scripting.
- **DataSync:** Managed, scheduled, integrity-guaranteed file transfers; handles large-scale NFS migrations without scripted rsync.
- **Snow Family:** Works in environments with no or poor internet connectivity; physical transfer is faster than internet for > 10 TB.
- **Transfer Family:** No client-side changes; drop-in replacement for SFTP servers; files land directly in S3.

---

## Cons

- **MGN:** Agent installation required (agentless only for VMware); EC2 instance types must be tested and right-sized post-rehost.
- **DMS:** Heterogeneous migrations require SCT and significant manual effort for complex stored procedures; LOB handling can degrade performance; replication instance is a potential bottleneck.
- **DataSync:** Requires DataSync agent for on-premises sources; agent is a single point of failure (deploy HA agent pairs); small-file-heavy workloads are slower.
- **Snow Family:** Turnaround time (order → ship → return → ingest) can be 1–2 weeks; physical security of the device during transport.
- **Transfer Family:** Costs per-protocol-endpoint + per-GB; FTP (no encryption) should only be used within private VPCs.

---

## When to Use

| Scenario | Tool |
|----------|------|
| Migrate 50 Windows Server VMs to EC2 | MGN |
| Migrate Oracle database to Aurora PostgreSQL | DMS + SCT |
| Migrate MySQL to Aurora MySQL (same engine) | DMS (no SCT needed) |
| Ongoing CDC replication for cut-over synchronisation | DMS (Full Load + CDC) |
| Transfer 30 TB from on-premises NFS share to S3 | DataSync |
| Transfer 500 GB weekly from on-premises to S3 (recurring) | DataSync (scheduled) |
| Transfer 500 TB with 100 Mbps internet link | Snowball Edge |
| Transfer 5 PB of archive data | Multiple Snowball Edge or Snowmobile |
| Edge ML inference at factory with no internet | Snowball Edge Compute Optimised |
| Give trading partners SFTP access to S3 without changing their clients | Transfer Family |
| Discover and inventory 1,000 on-premises servers | Application Discovery Service (agent or agentless) |

---

## Common Mistakes

- Using DMS alone for heterogeneous migration without running SCT first to assess schema complexity.
- Not enabling binary logging/supplemental logging on the source database before starting a DMS CDC task — CDC will fail.
- Expecting DataSync to provide ongoing hybrid access — use Storage Gateway for that; DataSync is for transfer tasks.
- Ordering Snowball when DataSync would complete in the available time window — calculate the transfer time first.
- Forgetting that MGN migrates the full OS — the migrated EC2 instance will have the same OS, application, and configuration as the source. Database engine is also copied as-is (not converted).
- Using Snowmobile when volume is < 10 PB — Snowball Edge is sufficient and simpler to coordinate.
