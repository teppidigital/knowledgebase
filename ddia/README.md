# Designing Data-Intensive Applications

Key lessons from *Designing Data-Intensive Applications* (Martin Kleppmann — 2017).

A catalogue of 15 patterns and concepts for building data systems that are reliable, scalable, and maintainable — drawn from the definitive reference on distributed data engineering.

## Patterns

| # | Topic | Category | Key Concepts |
|---|-------|----------|--------------|
| 01 | [Reliability, Scalability, Maintainability](./01-reliability-scalability-maintainability.md) | Foundations | SLAs, fan-out, load parameters, operability |
| 02 | [Data Models and Query Languages](./02-data-models.md) | Foundations | Relational, document, graph; impedance mismatch |
| 03 | [Storage and Retrieval](./03-storage-retrieval.md) | Foundations | B-trees, LSM-trees, SSTables, indexes, OLTP vs OLAP |
| 04 | [Encoding and Evolution](./04-encoding-evolution.md) | Foundations | JSON, Protobuf, Avro, forward/backward compatibility |
| 05 | [Replication](./05-replication.md) | Distributed Data | Leader-follower, multi-leader, leaderless, replication lag |
| 06 | [Partitioning and Sharding](./06-partitioning-sharding.md) | Distributed Data | Key-range, hash, secondary indexes, hot spots |
| 07 | [Transactions](./07-transactions.md) | Distributed Data | ACID, isolation levels, serializability, 2PL |
| 08 | [The Trouble with Distributed Systems](./08-distributed-systems-trouble.md) | Distributed Data | Network faults, clocks, process pauses, Byzantine faults |
| 09 | [Consistency and Consensus](./09-consistency-consensus.md) | Distributed Data | Linearizability, CAP, total order broadcast, Raft |
| 10 | [Batch Processing](./10-batch-processing.md) | Derived Data | MapReduce, dataflow, joins, fault tolerance |
| 11 | [Stream Processing](./11-stream-processing.md) | Derived Data | Event streams, stateful processing, exactly-once, watermarks |
| 12 | [Derived Data and System Architecture](./12-derived-data-systems.md) | Derived Data | Lambda/Kappa, unbundling databases, event logs as truth |
| 13 | [OLAP and Column-Oriented Storage](./13-olap-column-stores.md) | Analytics | Column store, compression, materialized views, cubes |
| 14 | [Distributed Transactions](./14-distributed-transactions.md) | Distributed Data | 2PC, XA, Saga pattern, idempotency, at-least-once |
| 15 | [Data Architecture Patterns](./15-data-architecture-patterns.md) | Architecture | Lambda, Kappa, data mesh, event sourcing, CQRS at scale |

---

## Book Structure

| Part | Chapters | Theme |
|---|---|---|
| **Part I — Foundations** | 01–04 | What data systems are; how individual nodes store and exchange data |
| **Part II — Distributed Data** | 05–09, 14 | What happens when multiple machines collaborate |
| **Part III — Derived Data** | 10–13, 15 | How to build complex systems from simpler pieces |

---

## Categories

### Foundations
- **Reliability, Scalability, Maintainability** — The three properties all data systems must have
- **Data Models** — Choosing the right model for the domain
- **Storage and Retrieval** — How databases actually work under the hood
- **Encoding** — How data is serialised and how systems evolve without breaking

### Distributed Data
- **Replication** — Keeping copies of data in sync across nodes
- **Partitioning** — Splitting large datasets across multiple nodes
- **Transactions** — Making multiple operations appear atomic
- **Distributed Systems Trouble** — Why distributed systems are fundamentally harder than single-node
- **Consistency and Consensus** — The theoretical limits and practical solutions
- **Distributed Transactions** — Making transactions work across nodes

### Derived Data
- **Batch Processing** — Processing bounded datasets at scale
- **Stream Processing** — Processing unbounded, continuous data flows
- **Derived Data Systems** — Combining systems into a coherent architecture
- **OLAP and Column Stores** — Analytics workloads at scale
- **Data Architecture Patterns** — Integrating everything into a coherent whole

---

## The DDIA Mental Model

```
Single Node
  ├── How data is stored: B-trees, LSM-trees, SSTables
  ├── How data is queried: indexes, query optimisers
  └── How data is transmitted: encoding formats, schema evolution

Multiple Nodes
  ├── Replication: keeping copies consistent
  ├── Partitioning: distributing the data
  ├── Transactions: making multi-step ops safe
  └── Consensus: agreeing on the state of the world

Derived Data
  ├── Batch: Spark, MapReduce — process a snapshot
  ├── Stream: Kafka Streams, Flink — process continuously
  └── Architecture: compose the above into reliable, maintainable systems
```

## Source

Kleppmann, M. (2017). *Designing Data-Intensive Applications: The Big Ideas Behind Reliable, Scalable, and Maintainable Systems*. O'Reilly Media.
