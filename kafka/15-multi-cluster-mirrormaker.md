# Multi-Cluster & MirrorMaker 2

## Category

Apache Kafka — Operations

## Context

**MirrorMaker 2 (MM2)** is Kafka's built-in replication tool built on the Kafka Connect framework. It continuously mirrors topics, consumer group offsets, and access control lists between Kafka clusters. MM2 is used for **geo-replication**, **disaster recovery (DR)**, **data migration**, and **multi-region active-passive or active-active deployments**.

### Multi-Cluster Deployment Patterns

| Pattern | Description | Failover | Write Conflicts |
|---------|-------------|----------|----------------|
| **Active-Passive** | All writes to primary; secondary is DR standby | Manual failover | None |
| **Active-Active** | Both clusters accept writes; bidirectional replication | Automatic / fast | Possible — requires deduplication |
| **Hub-and-Spoke** | Central hub replicates to multiple regional clusters | Per-region | None |
| **Aggregation** | Multiple edge clusters replicate into central cluster | — | None |

### MirrorMaker 2 vs MirrorMaker 1

| Aspect | MM1 | MM2 |
|--------|-----|-----|
| Framework | Standalone consumer/producer | Kafka Connect distributed |
| Topic auto-detection | Yes | Yes |
| Offset translation | No | Yes (`RemoteClusterUtils.translateOffsets`) |
| ACL replication | No | Yes |
| Consumer group sync | No | Yes |
| Multi-cluster topologies | Manual | Config-driven |

### Key MM2 Concepts

| Concept | Description |
|---------|-------------|
| **Replication policy** | Controls topic naming on target; default adds `source.` prefix |
| **Heartbeat connector** | Publishes heartbeats to measure replication lag |
| **Checkpoint connector** | Syncs committed offsets so consumers can resume on target |
| **Offset lag** | Target log end offset − source log end offset (per partition) |

## Pros

- Built-in offset translation — consumers can failover to target cluster without replaying from beginning
- Consumer group checkpoint sync enables near-zero-RPO failover with careful latency tuning
- Declarative configuration — define all replication flows in a single config file
- Topic auto-discovery — new topics matching a regex are automatically mirrored
- ACL replication reduces operational overhead when failing over

## Cons

- Active-active requires application-level deduplication — MM2 does not merge conflicting writes
- Replication lag adds latency to DR RPO; aggressive tuning trades broker CPU/network for lower lag
- Consumer group sync is eventually consistent — short replay window may be unavoidable on failover
- Topic renaming (`source.topic-name`) requires consumer config update after failover (or custom replication policy)
- `tasks.max` scales throughput but each task consumes broker threads and connections

## Design Diagram

```mermaid
flowchart LR
    subgraph Primary Region — eu-west-1
        PK[(Kafka<br/>Primary)]
        PP[Producers]
        PC[Consumers<br/>CG: app-v1]
        PP --> PK
        PK --> PC
    end

    subgraph DR Region — us-east-1
        DK[(Kafka<br/>DR / Standby)]
        DC[Consumers<br/>CG: app-v1<br/>(standby / inactive)]
        DK --> DC
    end

    subgraph MirrorMaker 2
        MM[MM2 Worker Cluster<br/> eu-west-1 or standalone]
        HC[Heartbeat Connector]
        CC[Checkpoint Connector]
        MC[Mirror Connector]
    end

    PK -->|Mirror Source topics| MC --> DK
    PK -->|Heartbeats| HC --> DK
    PC -->|Offsets| CC --> DK

    NOTE[On failover:<br/>1. Redirect producers to DK<br/>2. Translate offsets via checkpoint<br/>3. Resume consumers from translated offset]
```

## Code Sample

### Properties — MirrorMaker 2 standalone config (`mm2.properties`)

```properties
# Cluster aliases
clusters = primary, dr

# Primary cluster connection
primary.bootstrap.servers = kafka-primary.eu-west-1.example.com:9093
primary.security.protocol = SASL_SSL
primary.sasl.mechanism = SCRAM-SHA-512
primary.sasl.jaas.config = org.apache.kafka.common.security.scram.ScramLoginModule \
  required username="${PRIMARY_USER}" password="${PRIMARY_PASS}";

# DR cluster connection
dr.bootstrap.servers = kafka-dr.us-east-1.example.com:9093
dr.security.protocol = SASL_SSL
dr.sasl.mechanism = SCRAM-SHA-512
dr.sasl.jaas.config = org.apache.kafka.common.security.scram.ScramLoginModule \
  required username="${DR_USER}" password="${DR_PASS}";

# Replication flow: primary → dr
primary->dr.enabled = true
primary->dr.topics = payments\..*,orders\..*,accounts\..*
primary->dr.groups = .*
primary->dr.emit.heartbeats.enabled = true
primary->dr.emit.checkpoints.enabled = true
primary->dr.sync.group.offsets.enabled = true
primary->dr.sync.group.offsets.interval.seconds = 10

# Replication settings
replication.factor = 3
tasks.max = 8

# Offset lag monitoring
checkpoints.topic.replication.factor = 3
heartbeats.topic.replication.factor = 3
offset-syncs.topic.replication.factor = 3

# Do NOT rename topics on target (override default prefix behaviour)
replication.policy.class = org.apache.kafka.connect.mirror.IdentityReplicationPolicy
```

### TypeScript — Consumer failover with offset translation

```typescript
import { Kafka } from 'kafkajs';
import {
  KafkaAdminClient,
  RemoteClusterUtils,
} from './mirrormaker-utils'; // hypothetical helper

async function resumeConsumerOnDR(
  groupId: string,
  topics: string[],
): Promise<void> {
  const drKafka = new Kafka({
    clientId: `${groupId}-failover`,
    brokers: process.env.DR_KAFKA_BROKERS!.split(','),
  });

  const drAdmin = drKafka.admin();
  await drAdmin.connect();

  // Fetch translated offsets from checkpoint topic on DR cluster
  // MM2 wrote these via the Checkpoint Connector
  const translatedOffsets = await drAdmin.listOffsets(
    [{ topic: 'primary.checkpoints.internal', partition: 0 }],
  );

  console.log('Translated offsets for group', groupId, translatedOffsets);

  const drConsumer = drKafka.consumer({ groupId });
  await drConsumer.connect();

  for (const topic of topics) {
    await drConsumer.subscribe({ topic, fromBeginning: false });
  }

  // Override offsets to the translated checkpoints
  drConsumer.on(drConsumer.events.GROUP_JOIN, async () => {
    for (const topic of topics) {
      // Seek to checkpoint offset per partition
      // In practice: read mm2-offsets topic and parse per partition
      await drConsumer.seek({ topic, partition: 0, offset: '0' });
    }
  });

  await drConsumer.run({
    eachMessage: async ({ message }) => {
      console.log('Processing from DR:', message.value?.toString());
    },
  });
}
```

### Shell — Monitor MirrorMaker 2 replication lag

```bash
# Check MM2 heartbeat topic for replication lag
kafka-console-consumer.sh \
  --bootstrap-server kafka-dr:9092 \
  --topic heartbeats \
  --from-beginning \
  --max-messages 10

# Compare high watermarks between clusters
kafka-run-class.sh kafka.tools.GetOffsetShell \
  --bootstrap-server kafka-primary:9092 \
  --topic payments.created

kafka-run-class.sh kafka.tools.GetOffsetShell \
  --bootstrap-server kafka-dr:9092 \
  --topic payments.created

# Check MM2 connector status
curl http://mm2-connect:8083/connectors/MirrorSourceConnector/status | jq
curl http://mm2-connect:8083/connectors/MirrorCheckpointConnector/status | jq
```

### YAML — Strimzi KafkaMirrorMaker2 CRD

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaMirrorMaker2
metadata:
  name: primary-to-dr
spec:
  version: 3.7.0
  replicas: 3
  connectCluster: dr
  clusters:
    - alias: primary
      bootstrapServers: kafka-primary.eu-west-1.svc:9093
      tls:
        trustedCertificates:
          - secretName: primary-cluster-ca-cert
            certificate: ca.crt
      authentication:
        type: scram-sha-512
        username: mm2-user
        passwordSecret:
          secretName: mm2-primary-credentials
          password: password
    - alias: dr
      bootstrapServers: kafka-dr.us-east-1.svc:9093
      tls:
        trustedCertificates:
          - secretName: dr-cluster-ca-cert
            certificate: ca.crt
      authentication:
        type: scram-sha-512
        username: mm2-user
        passwordSecret:
          secretName: mm2-dr-credentials
          password: password
  mirrors:
    - sourceCluster: primary
      targetCluster: dr
      sourceConnector:
        tasksMax: 8
        config:
          replication.factor: 3
          offset-syncs.topic.replication.factor: 3
          sync.topic.acls.enabled: "true"
      heartbeatConnector:
        config:
          heartbeats.topic.replication.factor: 3
      checkpointConnector:
        tasksMax: 4
        config:
          checkpoints.topic.replication.factor: 3
          sync.group.offsets.enabled: "true"
          sync.group.offsets.interval.seconds: "10"
      topicsPattern: "payments\..*|orders\..*|accounts\..*"
      groupsPattern: ".*"
```

## References

- [Kafka MirrorMaker 2 Documentation](https://kafka.apache.org/documentation/#georeplication)
- [MirrorMaker 2 Design (KIP-382)](https://cwiki.apache.org/confluence/display/KAFKA/KIP-382%3A+MirrorMaker+2.0)
- [Strimzi MirrorMaker 2](https://strimzi.io/docs/operators/latest/deploying.html#assembly-deployment-kafka-mirror-maker-str)
- [Confluent — Multi-Region Clusters](https://docs.confluent.io/platform/current/multi-dc-deployments/index.html)
