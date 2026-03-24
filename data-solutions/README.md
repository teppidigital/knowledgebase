# Data Solutions Knowledge Base

A comprehensive reference for modern data architecture patterns — covering data ingestion, storage, querying, streaming, machine learning data infrastructure, governance, privacy, and organisational models.

---

## Patterns Index

| #                                          | Pattern                                      | Key Topics                                                                                                     |
| ------------------------------------------ | -------------------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| [01](01-batch-ingestion-etl.md)            | **Batch Ingestion & ETL Pipelines**          | ETL vs ELT, incremental loads, watermarks, SCD, dbt, Airflow, Snowflake MERGE                                  |
| [02](02-change-data-capture.md)            | **Change Data Capture (CDC)**                | Debezium, WAL tailing, Kafka Connect, Schema Registry, idempotent consumers, replication slots                 |
| [03](03-stream-processing.md)              | **Real-time Stream Processing**              | Apache Flink, Kafka Streams, windowing, watermarks, exactly-once, fraud detection topology                     |
| [04](04-data-lakehouse.md)                 | **Data Lakehouse Architecture**              | Apache Iceberg, Delta Lake, medallion architecture (Bronze/Silver/Gold), Spark MERGE, EMR Serverless           |
| [05](05-data-modelling-warehouse.md)       | **Data Modelling & Warehouse Design**        | Star schema, Kimball, Data Vault, SCD Type 2, dbt snapshots, fact + dimension tables, surrogate keys           |
| [06](06-search-fulltext.md)                | **Search & Full-Text Search**                | Elasticsearch, full-text + vector hybrid search, BM25, kNN, facets, index lifecycle, relevance tuning          |
| [07](07-caching-strategies.md)             | **Caching Strategies & Patterns**            | Cache-Aside, Write-Through, TTL, event-driven invalidation, Redis Cluster, stampede protection, rate limiting  |
| [08](08-time-series-data.md)               | **Time-Series Data & Analytics**             | TimescaleDB, hypertables, continuous aggregates, InfluxDB, VictoriaMetrics, retention policies, Prometheus     |
| [09](09-graph-databases.md)                | **Graph Databases & Relationship Modelling** | Neo4j, Amazon Neptune, Cypher, fraud ring detection, identity graph, recommendation engine                     |
| [10](10-olap-query-engines.md)             | **Analytical Query Engines & OLAP**          | ClickHouse, Trino, Apache Druid, DuckDB, federated queries, columnar storage, materialized views               |
| [11](11-feature-store-ml.md)               | **ML Feature Store & Data for AI**           | Feast, online/offline store, point-in-time join, training/serving skew, feature pipelines, real-time inference |
| [12](12-data-governance-catalogue.md)      | **Data Governance & Data Catalogue**         | DataHub, OpenMetadata, lineage, PII classification, Great Expectations, data quality dimensions                |
| [13](13-data-privacy-compliance.md)        | **Data Privacy & Compliance Engineering**    | GDPR, right to erasure, tokenisation, pseudonymisation, PII vault, AES-256-GCM, CCPA                           |
| [14](14-data-mesh.md)                      | **Data Mesh Architecture**                   | Domain ownership, data products, data contracts, federated governance, self-serve infrastructure               |
| [15](15-realtime-analytics-reverse-etl.md) | **Real-time Analytics & Reverse ETL**        | Embedded analytics, ClickHouse dashboards, SSE live counters, Salesforce sync, Reverse ETL patterns            |

---

## Patterns by Category

### Ingestion & Movement

- [01 — Batch Ingestion & ETL](01-batch-ingestion-etl.md) — Airflow/dbt incremental load, Snowflake MERGE, SCD patterns
- [02 — Change Data Capture](02-change-data-capture.md) — Debezium WAL tailing, idempotent Kafka consumers
- [03 — Stream Processing](03-stream-processing.md) — Flink, Kafka Streams, windowing, exactly-once guarantees

### Storage & Architecture

- [04 — Data Lakehouse](04-data-lakehouse.md) — Iceberg/Delta, medallion layers, Spark pipelines
- [05 — Data Modelling](05-data-modelling-warehouse.md) — Star schema, SCD2, dbt fact & dimension models
- [08 — Time-Series Data](08-time-series-data.md) — TimescaleDB hypertables, continuous aggregates, retention

### Query & Access

- [06 — Search & Full-Text](06-search-fulltext.md) — Elasticsearch BM25 + kNN hybrid search, faceted queries
- [07 — Caching](07-caching-strategies.md) — Redis Cache-Aside, stampede protection, rate limiting
- [09 — Graph Databases](09-graph-databases.md) — Neo4j Cypher, fraud rings, identity graphs
- [10 — OLAP Engines](10-olap-query-engines.md) — ClickHouse, Trino federated queries, materialized views

### AI & Analytics

- [11 — Feature Store & ML](11-feature-store-ml.md) — Feast, point-in-time joins, real-time feature retrieval
- [15 — Real-time Analytics & Reverse ETL](15-realtime-analytics-reverse-etl.md) — Live dashboards, SSE, CRM sync

### Governance & Compliance

- [12 — Data Governance](12-data-governance-catalogue.md) — DataHub, lineage, Great Expectations quality suites
- [13 — Data Privacy](13-data-privacy-compliance.md) — GDPR erasure, tokenisation vault, PII classification
- [14 — Data Mesh](14-data-mesh.md) — Domain ownership, data contracts, federated governance

---

## Tool Ecosystem Reference

| Tool                           | Category          | Purpose                                              |
| ------------------------------ | ----------------- | ---------------------------------------------------- |
| **Apache Airflow**             | Orchestration     | DAG-based pipeline scheduling                        |
| **Prefect / Dagster**          | Orchestration     | Modern Python workflow engines                       |
| **dbt**                        | Transform         | SQL-first ELT transformations with tests and lineage |
| **Debezium**                   | CDC               | Log-based change data capture                        |
| **Apache Kafka**               | Messaging         | Durable event streaming backbone                     |
| **Apache Flink**               | Stream processing | Stateful, exactly-once streaming                     |
| **Kafka Streams**              | Stream processing | Lightweight JVM stream processing                    |
| **Apache Iceberg**             | Table format      | ACID tables on object storage                        |
| **Delta Lake**                 | Table format      | Databricks-native open table format                  |
| **Apache Spark**               | Compute           | Distributed batch and streaming compute              |
| **Snowflake**                  | Warehouse         | Managed cloud data warehouse (SQL)                   |
| **BigQuery**                   | Warehouse         | Serverless Google Cloud warehouse                    |
| **ClickHouse**                 | OLAP              | Sub-second columnar analytics                        |
| **Apache Druid**               | OLAP              | High-concurrency real-time analytics                 |
| **Trino / Presto**             | Federated SQL     | Query across any data source with ANSI SQL           |
| **DuckDB**                     | Embedded OLAP     | Local analytical queries on Parquet/CSV              |
| **Elasticsearch / OpenSearch** | Search            | Full-text + vector search engine                     |
| **TimescaleDB**                | Time-series       | PostgreSQL extension for time-series data            |
| **InfluxDB**                   | Time-series       | Purpose-built TSDB with line protocol                |
| **Neo4j**                      | Graph DB          | Native graph database for relationship queries       |
| **Amazon Neptune**             | Graph DB          | Managed graph DB (Gremlin / openCypher)              |
| **Redis**                      | Cache / Streams   | Distributed in-memory cache and pub/sub              |
| **Feast**                      | Feature Store     | Open-source ML feature store                         |
| **Great Expectations**         | Data Quality      | Programmatic data quality testing                    |
| **Soda**                       | Data Quality      | SQL-based data quality checks                        |
| **DataHub**                    | Governance        | Open-source data catalogue and lineage               |
| **OpenMetadata**               | Governance        | All-in-one data governance platform                  |

---

## Data Architecture Decision Guide

| If you need…                              | Pattern to start with                                            |
| ----------------------------------------- | ---------------------------------------------------------------- |
| Move data from DB to warehouse daily      | [01 — Batch Ingestion & ETL](01-batch-ingestion-etl.md)          |
| Propagate DB changes in near real-time    | [02 — CDC](02-change-data-capture.md)                            |
| Process millions of events per second     | [03 — Stream Processing](03-stream-processing.md)                |
| Cheap, queryable historical data at scale | [04 — Data Lakehouse](04-data-lakehouse.md)                      |
| Structure data for BI and reporting       | [05 — Data Modelling](05-data-modelling-warehouse.md)            |
| Build a product search feature            | [06 — Search](06-search-fulltext.md)                             |
| Reduce DB load and response time          | [07 — Caching](07-caching-strategies.md)                         |
| Store and query IoT / metric data         | [08 — Time-Series](08-time-series-data.md)                       |
| Detect fraud rings or relationships       | [09 — Graph Databases](09-graph-databases.md)                    |
| Sub-second analytics on billions of rows  | [10 — OLAP Engines](10-olap-query-engines.md)                    |
| Serve ML features in real-time            | [11 — Feature Store](11-feature-store-ml.md)                     |
| Know what data exists and who owns it     | [12 — Data Governance](12-data-governance-catalogue.md)          |
| Handle GDPR right-to-erasure at scale     | [13 — Privacy & Compliance](13-data-privacy-compliance.md)       |
| Scale data across many domain teams       | [14 — Data Mesh](14-data-mesh.md)                                |
| Show live metrics in a product UI         | [15 — Real-time Analytics](15-realtime-analytics-reverse-etl.md) |

---

## Related Sections

| Section                                                                  | Relevance                                                                                |
| ------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------- |
| [`system-design/`](../system-design/README.md)                           | CQRS, Event Sourcing, Sharding, Replication — architectural foundations for data systems |
| [`distributed-design-pattern/`](../distributed-design-pattern/README.md) | Write-Ahead Log, Gossip, Vector Clocks — distributed primitives used by data systems     |
| [`devops/`](../devops/README.md)                                         | Database DevOps, alerting on pipeline SLOs, DORA metrics                                 |
| [`security/`](../security/README.md)                                     | Encryption at rest/in-transit, secrets management for DB credentials                     |
| [`devsecops/`](../devsecops/README.md)                                   | Data asset scanning in CI, IaC security for data infrastructure                          |
| [`cloud-native/aws/`](../cloud-native/aws/README.md)                     | AWS Glue, S3, DynamoDB, Kinesis, Redshift                                                |
| [`cloud-native/azure/`](../cloud-native/azure/README.md)                 | Azure Data Factory, ADLS, Synapse, Cosmos DB, Event Hubs                                 |
