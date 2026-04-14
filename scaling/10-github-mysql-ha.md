# GitHub — MySQL High Availability with Orchestrator and gh-ost

## Category

Scaling, Database, MySQL, High Availability, Failover Automation, Online Schema Migration, ProxySQL

## Scale at the Time

| Metric | Value |
|--------|-------|
| Repositories | Hundreds of millions |
| Git push events per day | Millions |
| MySQL clusters | Multiple (primary + replicas per service) |
| Schema change frequency | Several per week per service |
| Failover time (manual, before) | 20–45 minutes |
| Failover time (automated, after) | < 30 seconds |
| Online migration tool | gh-ost (GitHub's own) |

---

## Initial Architecture

GitHub runs on MySQL for all relational data: repositories, users, issues, pull requests, organisations, and permissions. For years, GitHub operated a traditional primary-replica MySQL setup managed manually — human operators coordinated failover, schema migrations, and replication topology changes.

```
GitHub App → MySQL Primary (all writes)
           → MySQL Replica 1 (reads)
           → MySQL Replica 2 (reads)
           → MySQL Replica 3 (reads)
```

This worked until GitHub's scale and velocity made manual operations too slow and error-prone.

---

## The Problem

### 1. Manual Failover Takes 20–45 Minutes
When the MySQL primary fails (hardware failure, crash, degraded performance), a human operator must:
1. Identify the failure (alert, triage, confirm)
2. Determine the best replica to promote (most current replication position)
3. Stop replication on all replicas
4. Promote the chosen replica to primary
5. Reconfigure all other replicas to replicate from the new primary
6. Update application configuration with the new primary hostname
7. Verify replication topology is healthy

Under pressure during an incident, this took 20–45 minutes. During that time, all writes to GitHub were unavailable.

### 2. Blocking Schema Migrations
MySQL's `ALTER TABLE` acquires a metadata lock for the duration of the DDL operation. For a table with 100M+ rows, an `ALTER TABLE` to add a column takes hours during which:
- All writes to the table block (queue behind the lock)
- All reads that need a table lock block
- The migration itself can cause I/O saturation, slowing the entire instance

GitHub engineers could not add a column to `issues` (a multi-hundred-million-row table) without a lengthy maintenance window.

### 3. Replication Topology Complexity
As GitHub added replicas, geo-distributed failover replicas, and delayed replicas (for disaster recovery), managing the replication graph manually became error-prone. A wrong `CHANGE MASTER TO` command during failover could create a replication split-brain or cause data loss.

### 4. Replica Lag Under Heavy Writes
MySQL replication is single-threaded by default (even with parallel replication, it is bounded). During large data backfills or high write spikes, replicas fall behind. Application code reading from replicas may get stale data — worse, the lag is invisible without explicit monitoring.

### 5. Connection Routing After Failover
When the primary changes, the application must update its database connection string to point at the new primary. Without a proxy layer, this required a configuration change and redeployment of the application — adding minutes to an already-long failover.

---

## The Solution

### S1. Orchestrator — Automated Failover and Topology Management

GitHub built and open-sourced **Orchestrator**: a MySQL topology management and failover automation tool.

**What Orchestrator does:**
- Continuously discovers and maps the full MySQL replication topology (which replica replicates from which primary, at what position)
- Monitors each node's health (replication lag, binary log position, uptime)
- Detects primary failure (heartbeat timeout) and automatically promotes the most up-to-date replica
- Reconfigures remaining replicas to replicate from the new primary
- Provides a web UI and API for manual topology operations

**Failover flow with Orchestrator:**
1. Orchestrator detects heartbeat failure on primary (3 consecutive missed heartbeats, configurable)
2. Selects the best replica: replica with smallest replication lag + meets promotion rules (not delayed replica, not cross-DC replica unless configured)
3. Promotes selected replica to primary (`RESET SLAVE; SET GLOBAL read_only=OFF`)
4. Relocates sibling replicas to replicate from new primary (`CHANGE MASTER TO`)
5. Runs custom hooks (update VIP, update DNS, notify Slack)
6. Total time: **< 30 seconds** for a two-replica cluster

### S2. ProxySQL — Connection Proxying with Automatic Rerouting

GitHub deployed **ProxySQL** as a connection proxy between the application and MySQL:
- Application connects to ProxySQL (always the same hostname)
- ProxySQL maintains connection pools to each MySQL replica
- ProxySQL routes write queries to primary, read queries to replicas (based on query rules)
- After failover, Orchestrator updates ProxySQL's routing configuration — the application connection string never changes

```
GitHub App → ProxySQL (stable endpoint, :3306)
               → MySQL Primary (writes)
               → MySQL Replica 1 (reads)
               → MySQL Replica 2 (reads)
```

ProxySQL query routing rules (example):
```sql
INSERT INTO mysql_query_rules (rule_id, active, match_pattern, destination_hostgroup)
VALUES
  (1, 1, '^SELECT',  20),   -- reads → replica hostgroup
  (2, 1, '.',        10);   -- everything else → primary hostgroup
```

### S3. gh-ost — Online Schema Migrations Without Locks

GitHub built and open-sourced **gh-ost** (GitHub Online Schema Transmogrifier): an online DDL tool that performs schema changes without table locks.

**How gh-ost works:**
1. Creates a "ghost" table with the new schema
2. Copies existing rows from the real table to the ghost table in small batches (configurable rate)
3. Captures ongoing changes to the real table via MySQL binary log (binlog streaming, not triggers)
4. Applies binlog changes to the ghost table in real time, keeping it in sync
5. Once the ghost table is fully caught up, atomically renames `real_table` → `old_table`, `ghost_table` → `real_table` (two-step atomic rename, near-zero downtime)
6. Optionally renames the `old_table` for later cleanup

**gh-ost is safe because:**
- No triggers on the table (triggers double write load and can cause deadlocks)
- Rate-limited: configurable rows/second, replication lag threshold (pauses if replica lag exceeds threshold)
- Resumable: if interrupted, restarts from the last checkpoint
- Testable on replica: can run the migration on a replica first to validate timing and correctness

### S4. Delayed Replica for Point-in-Time Recovery

GitHub maintains a **delayed replica** — a MySQL replica intentionally configured to lag the primary by 1 hour:

```sql
-- Delayed replica: applies events 3600 seconds behind primary
CHANGE MASTER TO MASTER_DELAY = 3600;
```

**Use case:** If someone accidentally drops a table or corrupts data, the delayed replica gives a 1-hour window to stop it at the correct point-in-time before the DROP TABLE event arrives, then export the table data from it.

Without a delayed replica, recovery from accidental data destruction requires restoring from the last full backup (potentially many hours old) and replaying binary logs — which takes much longer.

### S5. Replication Heartbeat Monitoring

GitHub monitors replica lag with a heartbeat table that the primary writes to every second. Replicas report their lag as the difference between `NOW()` and the last heartbeat timestamp they have applied:

```sql
-- Heartbeat table on primary
CREATE TABLE heartbeat (
  id          INT NOT NULL DEFAULT 1,
  ts          DATETIME(6) NOT NULL,
  PRIMARY KEY (id)
);

-- Heartbeat writer (runs on primary, every 1 second)
INSERT INTO heartbeat (id, ts) VALUES (1, NOW(6))
ON DUPLICATE KEY UPDATE ts = NOW(6);

-- Lag query on replica
SELECT TIMESTAMPDIFF(MICROSECOND, ts, NOW(6)) / 1000000 AS lag_seconds
FROM heartbeat;
```

---

## Key Learnings

1. **Manual failover at 20–45 minutes is unacceptable** — automated failover with Orchestrator or similar (MHA, Patroni for PostgreSQL) eliminates this; invest in automation before you need it during an incident
2. **Never ALTER TABLE in production without an online DDL tool** — adopt gh-ost, pt-online-schema-change, or MySQL 8.0 instant DDL from the start; the first time you need to add a column to a 100M-row table without it will be a crisis
3. **A proxy layer is mandatory for connection rerouting after failover** — ProxySQL allows the application connection string to remain stable across failover events; without it, failover requires an application config change and redeployment
4. **Maintain a delayed replica** — a 1-hour delayed replica is cheap insurance against accidental data deletion; it has saved teams from restoring from backup many times
5. **Monitor replication lag with a heartbeat table, not SHOW SLAVE STATUS** — `Seconds_Behind_Master` in `SHOW SLAVE STATUS` is unreliable (uses system clocks, not event timestamps); a heartbeat table gives accurate, queryable lag from application code
6. **Promote the replica with the smallest replication lag** — during failover, the replica with the most up-to-date replication position loses the fewest transactions; Orchestrator does this automatically but ERRANT transactions on replicas can still cause problems
7. **gh-ost rate limiting prevents migration from degrading production** — default gh-ost throttles based on replication lag; configure it to pause if replica lag exceeds your SLO threshold

---

## Architecture Diagram

```mermaid
graph TD
    App["GitHub Application<br/>(Rails)"]
    ProxySQL["ProxySQL<br/>(connection pool + query routing)<br/>Stable endpoint — never changes"]

    subgraph "MySQL Replication Topology"
        Primary[("MySQL Primary<br/>(writes)")]
        Replica1[("MySQL Replica 1<br/>(reads)")]
        Replica2[("MySQL Replica 2<br/>(reads)")]
        Delayed[("MySQL Delayed Replica<br/>(1 hour lag — disaster recovery)")]
        FailoverCandidate[("Failover Candidate<br/>(promoted by Orchestrator)")]
    end

    Orchestrator["Orchestrator<br/>(topology discovery + health check)<br/>Automated failover < 30s"]
    GhOst["gh-ost<br/>(online schema migration)<br/>No table locks"]
    Heartbeat["Heartbeat Writer<br/>(updates heartbeat table every 1s)"]

    App --> ProxySQL
    ProxySQL --writes--> Primary
    ProxySQL --reads--> Replica1
    ProxySQL --reads--> Replica2
    Primary --replication--> Replica1
    Primary --replication--> Replica2
    Primary --replication (delayed 1h)--> Delayed
    Primary --replication--> FailoverCandidate

    Orchestrator --monitors--> Primary
    Orchestrator --monitors--> Replica1
    Orchestrator --monitors--> FailoverCandidate
    Orchestrator --on failure: promotes--> FailoverCandidate
    Orchestrator --reconfigures routing--> ProxySQL

    GhOst --binlog stream--> Primary
    GhOst --batch copy + apply--> Replica1
    Heartbeat --> Primary
```

---

## Code / Config

### Orchestrator configuration (orchestrator.conf.json, excerpt)

```json
{
  "MySQLTopologyUser": "orchestrator",
  "MySQLTopologyPassword": "secret",
  "MySQLOrchestratorMaxPoolConnections": 3,
  "InstancePollSeconds": 5,
  "DiscoverByShowSlaveHosts": true,
  "RecoverMasterClusterFilters": ["*"],
  "RecoveryPollSeconds": 1,
  "MasterFailoverLostInstancesDowntimeMinutes": 1,
  "PostFailoverProcesses": [
    "echo 'Failover: promoted {failover.NewMaster.Hostname}' | notify-slack",
    "update-proxysql {failover.NewMaster.Hostname} {failover.Cluster}"
  ],
  "UnseenInstanceForgetHours": 1,
  "DetachLostReplicasAfterMasterFailover": true,
  "ApplyMySQLPromotionAfterMasterFailover": true,
  "MasterFailoverDetachSlaveMasterHost": false,
  "FailMasterPromotionIfSQLThreadNotUpToDate": true
}
```

### gh-ost schema migration command

```bash
# Online ALTER TABLE: add a column to the issues table
# - Runs on production primary
# - throttles if replica lag > 1000ms
# - max 2000 rows/sec copy rate (adjustable at runtime via UNIX socket)
# - uses binary log streaming (no triggers)

gh-ost \
  --host="mysql-primary.internal" \
  --port=3306 \
  --user="gh-ost" \
  --password="secret" \
  --database="github_production" \
  --table="issues" \
  --alter="ADD COLUMN spam_score FLOAT DEFAULT NULL AFTER id" \
  --chunk-size=2000 \
  --max-lag-millis=1000 \
  --max-load="Threads_running=25" \
  --critical-load="Threads_running=50" \
  --throttle-control-replicas="replica1.internal,replica2.internal" \
  --cut-over=default \
  --allow-on-master \
  --ok-to-drop-table \
  --execute
```

### ProxySQL query routing rules

```sql
-- Route SELECT queries to read replicas (hostgroup 20)
-- Route everything else (INSERT/UPDATE/DELETE/BEGIN) to primary (hostgroup 10)
DELETE FROM mysql_query_rules;

INSERT INTO mysql_query_rules (rule_id, active, match_pattern, destination_hostgroup, apply) VALUES
  (1, 1, '^SELECT\s+\bSQL_NO_CACHE\b', 20, 1),  -- explicit replica read
  (2, 1, '^SELECT',                     20, 1),  -- all other SELECTs -> replica
  (3, 1, '^BEGIN',                      10, 1),  -- transactions -> primary
  (4, 1, '.',                           10, 1);  -- everything else -> primary

LOAD MYSQL QUERY RULES TO RUNTIME;
SAVE MYSQL QUERY RULES TO DISK;
```

---

## References

- [GitHub Engineering — Orchestrator: MySQL Replication Topology Manager](https://github.blog/engineering/orchestrator-mysql-replication-topology-manager/) 
- [GitHub Engineering — gh-ost: Online Schema Migrations for MySQL](https://github.blog/engineering/gh-ost-github-online-schema-migrations-for-mysql/) (2016)
- [GitHub — Orchestrator](https://github.com/openark/orchestrator)
- [GitHub — gh-ost](https://github.com/github/gh-ost)
- [ProxySQL Documentation](https://proxysql.com/Documentation/)
- [Percona — pt-heartbeat](https://docs.percona.com/percona-toolkit/pt-heartbeat.html)
