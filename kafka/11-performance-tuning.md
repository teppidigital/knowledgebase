# Performance Tuning

## Category

Apache Kafka — Operations

## Context

Kafka's performance profile is determined by the interaction between **producer batching**, **broker page cache**, **OS disk I/O**, and **network throughput**. Tuning is always a trade-off between **throughput** (maximise bytes/second) and **latency** (minimise end-to-end delay). Benchmark with realistic payloads before adopting any setting.

### Throughput vs Latency Trade-off Matrix

| Setting | Increase Throughput | Decrease Latency |
|---------|--------------------|--------------------|
| `linger.ms` | Increase (10–100 ms) | Decrease (0–5 ms) |
| `batch.size` | Increase (128 KB–1 MB) | Decrease (16–32 KB) |
| `compression.type` | lz4 or zstd | none |
| `acks` | `1` or `0` | `1` (avoid `all`) |
| `fetch.min.bytes` (consumer) | Increase | Decrease |
| `fetch.max.wait.ms` (consumer) | Increase | Decrease |
| `num.io.threads` (broker) | Increase | Already fast |
| Partition count | Increase (parallelism) | Decrease (reduce coordination) |

### Broker OS Tuning

| Parameter | Recommended Value | Rationale |
|-----------|------------------|-----------|
| `vm.swappiness` | `1` | Prefer page cache eviction over swapping |
| `vm.dirty_ratio` | `80` | Allow large dirty page buffers for sequential writes |
| `vm.dirty_background_ratio` | `5` | Background flush threshold |
| `fs.file-max` | `1000000` | Each partition = file descriptors |
| `net.core.rmem_max` / `wmem_max` | `134217728` | 128 MB socket buffers |
| Disk scheduler | `mq-deadline` (HDD) / `none` (NVMe) | Optimise for sequential I/O |
| Mount option | `noatime` | Avoid inode timestamp updates on reads |

### Compression Comparison

| Algorithm | CPU Cost | Compression Ratio | Decompression Speed | Best For |
|-----------|----------|-------------------|--------------------|---------| 
| none | Zero | 1× | — | Already small payloads |
| gzip | High | 3–4× | Moderate | Archival, cold topics |
| snappy | Low | 2–3× | Fast | Balanced default |
| **lz4** | Very Low | 2–3× | **Very fast** | **Recommended default** |
| **zstd** | Low | **4–5×** | Fast | Best compression, Kafka 2.1+ |

## Pros

- Page cache: sequential reads hit OS buffer cache, not disk — Kafka is often fully in-memory  
- Zero-copy transfer: broker uses `sendfile()` syscall to send from page cache directly to network  
- Batching multiplies throughput — 1000 records sent as one batch is dramatically faster than 1000 sends  
- lz4 compression is nearly free CPU-wise but halves network and storage cost  
- Partition-level parallelism scales linearly with consumer count  

## Cons

- Increasing `linger.ms` trades latency for throughput — not suitable for real-time financial systems  
- Large batches increase memory pressure on producer (`buffer.memory`) and broker heap  
- Too many partitions increase controller metadata overhead and slow leader election  
- Compression on the producer is decompressed and recompressed on the broker if it does not match topic compression — avoid mismatches  
- Page cache tuning affects the whole OS — Kafka should ideally run on dedicated nodes  

## Design Diagram

```mermaid
flowchart LR
    subgraph Producer JVM
        APP[App Thread<br/>send records] --> ACC[Accumulator<br/>Buffer batch.size / linger.ms]
        ACC --> COMP[Compress<br/>lz4 / zstd]
        COMP --> NW1[Network Thread → Broker]
    end

    subgraph Broker
        NW2[Network Thread<br/>receive] --> RQ[Request Queue<br/>num.network.threads]
        RQ --> IO[I/O Thread<br/>num.io.threads]
        IO -->|Page Cache| PC[(OS Page Cache<br/>sequential write)]
        PC -.->|flush| DISK[(SSD / NVMe<br/>log segments)]
        PC -->|zero-copy sendfile| CSEND[Consumer Send]
    end

    subgraph Consumer JVM
        CSEND --> DECOMP[Decompress]
        DECOMP --> CB[poll() callback]
    end
```

## Code Sample

### TypeScript — Producer tuned for high throughput

```typescript
import { Kafka, Partitioners, CompressionTypes } from 'kafkajs';

const kafka = new Kafka({
  clientId: 'bulk-event-producer',
  brokers: process.env.KAFKA_BROKERS!.split(','),
});

// High-throughput producer
const producer = kafka.producer({
  createPartitioner: Partitioners.DefaultPartitioner,
  idempotent: true,
  // KafkaJS does not expose linger.ms directly;
  // batch records application-side before calling send()
});

export async function sendBatch<T>(
  topic: string,
  events: Array<{ key: string; value: T }>,
): Promise<void> {
  await producer.send({
    topic,
    compression: CompressionTypes.LZ4,
    messages: events.map(e => ({
      key: e.key,
      value: JSON.stringify(e.value),
    })),
  });
}
```

### TypeScript — Consumer tuned for high throughput

```typescript
import { Kafka } from 'kafkajs';

const consumer = kafka.consumer({
  groupId: 'bulk-processor',
  maxBytesPerPartition: 10_485_760,  // 10 MB per partition per fetch
  minBytes: 1_048_576,               // fetch.min.bytes = 1 MB
  maxWaitTimeInMs: 500,              // fetch.max.wait.ms
});

await consumer.run({
  autoCommit: true,
  autoCommitInterval: 5000,
  autoCommitThreshold: 1000,
  partitionsConsumedConcurrently: 8,
  eachBatch: async ({ batch }) => {
    // Process entire batch at once — avoids per-message overhead
    const values = batch.messages.map(m => JSON.parse(m.value!.toString()));
    await processBulk(values);
  },
});
```

### Shell — Kafka performance test tools

```bash
# Producer throughput test
kafka-producer-perf-test.sh \
  --topic perf-test \
  --num-records 10000000 \
  --record-size 1024 \
  --throughput -1 \
  --producer-props bootstrap.servers=localhost:9092 \
    acks=all compression.type=lz4 batch.size=131072 linger.ms=10

# Consumer throughput test
kafka-consumer-perf-test.sh \
  --bootstrap-server localhost:9092 \
  --topic perf-test \
  --messages 10000000 \
  --fetch-size 10485760

# Check page cache usage on broker
cat /proc/meminfo | grep -E 'MemFree|Cached|Buffers'
free -h
```

### Shell — OS tuning script for Kafka broker

```bash
#!/usr/bin/env bash
# Apply on all Kafka broker nodes

# Page cache / swap
sysctl -w vm.swappiness=1
sysctl -w vm.dirty_ratio=80
sysctl -w vm.dirty_background_ratio=5

# Network buffers
sysctl -w net.core.rmem_max=134217728
sysctl -w net.core.wmem_max=134217728
sysctl -w net.ipv4.tcp_rmem='4096 87380 134217728'
sysctl -w net.ipv4.tcp_wmem='4096 65536 134217728'

# File descriptors
ulimit -n 1000000
echo 'kafka soft nofile 1000000' >> /etc/security/limits.conf
echo 'kafka hard nofile 1000000' >> /etc/security/limits.conf

# Mount the Kafka data partition with noatime
# (Add noatime to /etc/fstab before mounting)
mount -o remount,noatime /var/kafka/data
```

## References

- [Kafka Documentation — Performance Tuning](https://kafka.apache.org/documentation/#brokerconfigs)
- [Confluent — Optimizing Kafka Deployment for Performance](https://docs.confluent.io/platform/current/kafka/deployment.html)
- [Kafka Performance Testing Tools](https://kafka.apache.org/documentation/#basic_ops_benchmarking)
- [LinkedIn — Kafka at Scale](https://engineering.linkedin.com/kafka/kafka-linkedin-current-and-future)
