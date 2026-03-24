# Read Replicas (Database Replication)

## Category
Scalability, Performance, Availability

## Context

Read Replicas is a database scaling pattern where a **primary (master) database** handles all writes, and one or more **replica (read) databases** receive a copy of all data asynchronously. Read traffic is routed to replicas, offloading the primary and dramatically improving read throughput.

Most modern databases support replication natively: PostgreSQL streaming replication, MySQL binary log replication, MongoDB replica sets, AWS RDS read replicas.

---

## Pros

- **Horizontal read scaling**: Multiple replicas can serve read traffic in parallel.
- **Reduced primary load**: Offloads read queries, freeing primary resources for writes.
- **High availability**: Replicas can be promoted to primary if the primary fails.
- **Geographic distribution**: Place replicas closer to read-heavy regions for lower latency.
- **Analytics isolation**: Run slow analytical queries on a dedicated replica without impacting production.
- **Backups without downtime**: Take backups from the replica instead of the primary.

---

## Cons

- **Replication lag**: Replicas may be slightly behind the primary — stale reads.
- **Consistency issues**: Users may read their own writes as stale if not handled correctly.
- **Write bottleneck remains**: All writes still go to a single primary.
- **Failover complexity**: Promoting a replica to primary requires coordination.
- **Increased operational cost**: More database instances to manage and pay for.

---

## Design Diagram

```mermaid
graph TD
    App["Application"]
    WriteRouter["Write Router"]
    ReadRouter["Read Router (Load Balanced)"]

    Primary[("Primary DB<br/>(All Writes)")]
    Replica1[("Read Replica 1<br/>(Async Replication)")]
    Replica2[("Read Replica 2<br/>(Async Replication)")]
    Replica3[("Read Replica 3<br/>(Analytics)")]

    App -->|"INSERT / UPDATE / DELETE"| WriteRouter
    App -->|"SELECT"| ReadRouter

    WriteRouter --> Primary
    Primary -->|"Replication stream"| Replica1
    Primary -->|"Replication stream"| Replica2
    Primary -->|"Replication stream"| Replica3

    ReadRouter --> Replica1
    ReadRouter --> Replica2
```

---

## Code Sample

### Read/Write Splitting (Node.js / PostgreSQL)

```javascript
// db/connection.js
const { Pool } = require('pg');

// Primary: handles all writes
const primaryPool = new Pool({ connectionString: process.env.PRIMARY_DB_URL });

// Read replicas: round-robin read distribution
const replicaPools = [
  new Pool({ connectionString: process.env.REPLICA_1_URL }),
  new Pool({ connectionString: process.env.REPLICA_2_URL }),
];

let replicaIndex = 0;

function getReadPool() {
  const pool = replicaPools[replicaIndex % replicaPools.length];
  replicaIndex++;
  return pool;
}

function getWritePool() {
  return primaryPool;
}

module.exports = { getReadPool, getWritePool };
```

### Repository with Read/Write Routing

```javascript
// repositories/user.repository.js
const { getReadPool, getWritePool } = require('../db/connection');

async function getUserById(id) {
  // Route to replica
  const db = getReadPool();
  const { rows } = await db.query('SELECT * FROM users WHERE id = $1', [id]);
  return rows[0];
}

async function createUser(name, email) {
  // Route to primary
  const db = getWritePool();
  const { rows } = await db.query(
    'INSERT INTO users (name, email) VALUES ($1, $2) RETURNING *',
    [name, email]
  );
  return rows[0];
}

async function listUsers(limit = 50) {
  const db = getReadPool();
  const { rows } = await db.query('SELECT * FROM users ORDER BY created_at DESC LIMIT $1', [limit]);
  return rows;
}

module.exports = { getUserById, createUser, listUsers };
```

### Read-Your-Writes Consistency (sticky session or synchronous read from primary)

```javascript
// After a write, read from primary to guarantee freshness
async function updateUserAndReturn(userId, data) {
  const writeDb = getWritePool();

  await writeDb.query(
    'UPDATE users SET name = $1 WHERE id = $2',
    [data.name, userId]
  );

  // Read fresh data from primary (not replica) to avoid stale read
  const { rows } = await writeDb.query('SELECT * FROM users WHERE id = $1', [userId]);
  return rows[0];
}
```

### AWS RDS Read Replica (Terraform)

```hcl
# terraform/rds-replica.tf
resource "aws_db_instance" "primary" {
  identifier        = "myapp-primary"
  engine            = "postgres"
  engine_version    = "15.4"
  instance_class    = "db.r6g.large"
  allocated_storage = 100
  db_name           = "myapp"
  username          = "admin"
  password          = var.db_password
  multi_az          = true
}

resource "aws_db_instance" "replica_1" {
  identifier          = "myapp-replica-1"
  replicate_source_db = aws_db_instance.primary.identifier
  instance_class      = "db.r6g.large"
  publicly_accessible = false
}

resource "aws_db_instance" "replica_2" {
  identifier          = "myapp-replica-2"
  replicate_source_db = aws_db_instance.primary.identifier
  instance_class      = "db.r6g.large"
  publicly_accessible = false
}
```
