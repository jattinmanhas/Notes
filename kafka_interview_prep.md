# Kafka Interview Prep

---

## 1. What is Kafka?

Apache Kafka is a **distributed, fault-tolerant, high-throughput event streaming platform**. It's used for:
- Real-time data pipelines
- Stream processing
- Event-driven architectures
- Log aggregation

Originally built at LinkedIn, open-sourced in 2011, now an Apache top-level project.

---

## 2. Core Concepts

### Topics
- A **topic** is a named category/feed to which records are published.
- Topics are split into **partitions** for parallelism and scalability.
- Topics are **append-only** logs — data is immutable once written.

### Partitions
- Each topic has one or more partitions.
- Messages within a partition are **ordered** and assigned a sequential **offset**.
- Ordering is guaranteed **within** a partition, NOT across partitions.
- More partitions = higher parallelism and throughput.

### Offsets
- A unique, monotonically increasing integer assigned to each message within a partition.
- Consumers track their position using offsets.
- Kafka itself stores committed offsets in an internal topic: `__consumer_offsets`.

### Brokers
- A Kafka cluster is made up of multiple **brokers** (servers).
- Each broker stores a subset of partition data.
- One broker acts as the **leader** for a partition; others are **followers** (replicas).

### Producers
- Produce (write) messages to topics.
- Can choose which partition to write to: round-robin, key-based hashing, or custom partitioner.
- Can configure delivery guarantees: `acks=0`, `acks=1`, `acks=all`.

### Consumers
- Read messages from topics.
- Consumers belong to a **consumer group**.
- Each partition is consumed by **exactly one consumer** in a group at a time.
- Multiple groups can independently consume the same topic.

### Consumer Groups
- Allow horizontal scaling of consumption.
- Kafka rebalances partitions among consumers when group membership changes.
- If consumers > partitions, some consumers are idle.

### Zookeeper / KRaft
- Historically Kafka used **ZooKeeper** for cluster coordination, metadata, and leader election.
- Since Kafka 2.8+, **KRaft mode** (Kafka Raft) removes the ZooKeeper dependency — Kafka manages its own metadata internally.

---

## 3. Kafka Architecture Deep Dive

### Replication
- Kafka replicates each partition across `replication.factor` brokers.
- One replica is the **leader** (handles all reads/writes); others are **ISR** (In-Sync Replicas).
- If the leader fails, one ISR is elected as the new leader.

### ISR (In-Sync Replicas)
- Replicas that are fully caught up with the leader.
- `min.insync.replicas` controls the minimum number of replicas that must acknowledge a write for it to succeed (used with `acks=all`).

### Log Compaction
- Instead of deleting old messages by time/size, Kafka can **compact** a topic: keeps the latest value for each key.
- Useful for changelog/event-sourcing patterns (e.g., maintaining the latest state of a record).

### Retention
- Messages are retained for a configurable duration (`log.retention.hours`) or size (`log.retention.bytes`), regardless of whether they've been consumed.
- Default retention: **7 days**.

---

## 4. Producer Internals

### Acknowledgements (`acks`)
| Setting | Meaning | Risk |
|---|---|---|
| `acks=0` | Fire and forget, no confirmation | Possible data loss |
| `acks=1` | Leader acknowledges | Loss if leader fails before replication |
| `acks=all` | All ISRs must acknowledge | Strongest guarantee, higher latency |

### Idempotent Producer
- Set `enable.idempotence=true` to prevent duplicate messages on retries.
- Kafka assigns a **Producer ID (PID)** and sequence numbers per partition.

### Transactions
- Kafka supports **exactly-once semantics (EOS)** via transactions.
- Use `transactional.id` to enable; wrap produce calls in `beginTransaction()` / `commitTransaction()`.

### Batching & Compression
- Producers batch messages to increase throughput: `batch.size`, `linger.ms`.
- Compression codecs: `gzip`, `snappy`, `lz4`, `zstd`. Applied per batch.

### Partitioner
- **Default**: hash of the key → partition. If no key, round-robin.
- Custom partitioners can implement `org.apache.kafka.clients.producer.Partitioner`.

---

## 5. Consumer Internals

### Poll Loop
- Consumers use a **poll loop** to fetch batches of records.
- `max.poll.interval.ms`: max time between polls before the consumer is considered dead and triggers a rebalance.
- `session.timeout.ms`: how long before a consumer with no heartbeats is removed from the group.

### Commit Strategies
| Strategy | How | Risk |
|---|---|---|
| Auto commit | `enable.auto.commit=true` | At-least-once (may reprocess on crash) |
| Manual sync commit | `commitSync()` after processing | Safer, blocks until committed |
| Manual async commit | `commitAsync()` | Non-blocking, may lose commit on failure |

### Delivery Semantics
- **At-most-once**: commit before processing. May lose messages.
- **At-least-once**: commit after processing. May reprocess.
- **Exactly-once**: requires idempotent consumers or Kafka Streams EOS.

### Rebalance
- Triggered when: consumer joins/leaves, partition count changes, or heartbeat times out.
- During rebalance, consumption pauses — can be a performance concern.
- **Cooperative rebalance** (`CooperativeStickyAssignor`) minimizes partition movement and pause time vs. the older eager protocol.

---

## 6. Kafka Streams

- A **client library** (not a separate cluster) for building stream processing apps on top of Kafka.
- Supports: `map`, `filter`, `flatMap`, `groupBy`, `aggregate`, `join`, `windowing`.
- Uses **state stores** (RocksDB by default) for stateful operations.
- Supports **exactly-once processing** with `processing.guarantee=exactly_once_v2`.

### KStream vs KTable
| | KStream | KTable |
|---|---|---|
| Represents | Unbounded stream of events | Changelog / latest state per key |
| Analogy | Append-only log | Database table |
| Join behavior | Every record | Latest value per key |

### Windowing
- **Tumbling**: fixed, non-overlapping windows (e.g., every 5 min).
- **Hopping**: fixed windows that overlap (e.g., 5 min window, every 1 min).
- **Sliding**: windows defined by time difference between records.
- **Session**: gap-based, groups records within an inactivity gap.

---

## 7. Kafka Connect

- Framework for **streaming data between Kafka and external systems** without writing code.
- **Source connectors**: pull data into Kafka (e.g., from MySQL, S3).
- **Sink connectors**: push data out of Kafka (e.g., to Elasticsearch, HDFS).
- Runs in **standalone** or **distributed** mode.
- Connectors are configured via JSON REST API.

---

## 8. Performance & Tuning

### Throughput Tuning (Producer)
- Increase `batch.size` and `linger.ms`
- Use compression (`lz4` or `snappy` for speed, `gzip`/`zstd` for ratio)
- Increase `buffer.memory`

### Throughput Tuning (Consumer)
- Increase `fetch.min.bytes` and `fetch.max.wait.ms`
- Process records in parallel within the consumer

### Latency Tuning
- Set `linger.ms=0` (don't wait to batch)
- Set `acks=1` (avoid waiting for all ISRs)
- Reduce `fetch.max.wait.ms`

### Partition Count
- More partitions = more parallelism but also more overhead (file handles, ZooKeeper/KRaft znodes, replication traffic).
- Rule of thumb: target throughput / throughput per partition.

---

## 9. Exactly-Once Semantics (EOS)

Three layers needed:
1. **Idempotent producer** — deduplicates retries at the broker level.
2. **Transactions** — atomic multi-partition writes.
3. **Transactional consumer** — reads only committed data (`isolation.level=read_committed`).

In Kafka Streams, set `processing.guarantee=exactly_once_v2` and it handles all of this automatically.

---

## 10. Common Interview Questions

### Fundamentals

**Q: What is the difference between a queue and Kafka?**
Traditional queues (RabbitMQ, SQS) delete messages after consumption. Kafka retains messages and allows multiple consumer groups to independently replay the same data.

**Q: How does Kafka guarantee ordering?**
Only within a single partition. To guarantee global order for a topic, use a single partition (sacrifices parallelism).

**Q: What happens when a consumer crashes mid-processing?**
If using manual commit, uncommitted offsets are re-read on restart → at-least-once delivery. If auto-commit was just fired, messages might be lost (at-most-once).

**Q: What is the role of the `__consumer_offsets` topic?**
Kafka stores committed consumer group offsets in this internal compacted topic. It replaced ZooKeeper-based offset storage in Kafka 0.9+.

**Q: Can a consumer read from multiple partitions?**
Yes. One consumer can be assigned multiple partitions, but one partition cannot be assigned to multiple consumers in the same group simultaneously.

---

### Architecture

**Q: What is ISR and why does it matter?**
In-Sync Replicas are followers that are fully caught up with the leader. With `acks=all` and `min.insync.replicas=2`, a write only succeeds if at least 2 ISRs acknowledge it — preventing data loss even on leader failure.

**Q: What happens when the leader broker goes down?**
Kafka detects the failure via heartbeats, the controller picks a new leader from the ISR list, and clients automatically reconnect to the new leader after a metadata refresh.

**Q: What is log compaction and when would you use it?**
Log compaction retains only the latest value per key, instead of deleting old data by time. Use it for event-sourcing, change data capture (CDC), or any scenario where you want to replay the "current state" of keyed records.

**Q: Explain the difference between `replication.factor` and `min.insync.replicas`.**
`replication.factor` is how many total copies exist. `min.insync.replicas` is the minimum that must be "in-sync" for a write to be accepted (with `acks=all`). If ISR count drops below this minimum, the broker rejects writes with `NotEnoughReplicasException`.

---

### Producer / Consumer

**Q: What's the difference between `acks=1` and `acks=all`?**
`acks=1` only waits for the leader to acknowledge; followers may not have the data yet. `acks=all` waits for all ISRs to confirm, giving the strongest durability guarantee.

**Q: How do you implement exactly-once in Kafka?**
Enable `enable.idempotence=true`, set a `transactional.id`, use `beginTransaction()`/`commitTransaction()` in the producer, and set `isolation.level=read_committed` in the consumer.

**Q: What is a consumer group rebalance and how can you minimize its impact?**
A rebalance reassigns partitions among group members, pausing consumption. Use `CooperativeStickyAssignor` for incremental rebalancing, tune `session.timeout.ms` and `max.poll.interval.ms` appropriately, and use static membership (`group.instance.id`) to avoid rebalances on restarts.

**Q: What is `linger.ms`?**
The time a producer waits to accumulate more messages into a batch before sending. A higher value increases throughput (larger batches) at the cost of latency.

---

### Kafka Streams

**Q: What is the difference between KStream and KTable?**
KStream represents an unbounded stream of immutable events. KTable represents a changelog — only the latest value per key is kept. KTable can be thought of as a materialized view of a KStream grouped by key.

**Q: How does Kafka Streams handle state?**
Via embedded **state stores** (backed by RocksDB by default). State stores are automatically changelog-backed to a Kafka topic, so they can be restored after a failure.

**Q: What is a GlobalKTable?**
A KTable that is fully replicated to every instance of the application (unlike a regular KTable, which is partitioned). Used for reference data lookups that need to be available on every node.

---

### Operational / Scenario

**Q: How do you scale Kafka consumers?**
Add more consumers to the consumer group — Kafka will rebalance partitions. Maximum parallelism = number of partitions. Add more partitions to the topic if you need more consumer instances.

**Q: How do you handle a lagging consumer group?**
- Identify lag using `kafka-consumer-groups.sh --describe`
- Scale up consumers (up to partition count)
- Optimize consumer processing (batching, async processing)
- If urgent: increase retention so messages aren't lost while catching up

**Q: How would you migrate a topic to more partitions?**
Use `kafka-topics.sh --alter --partitions N`. Note: existing messages stay in their current partitions; only new messages are distributed across all N partitions. Key-based ordering semantics change, so test carefully.

**Q: What is a dead letter queue (DLQ) in Kafka?**
A separate topic where messages that fail processing are written, rather than blocking the consumer indefinitely. Prevents poison-pill messages from stalling a partition.

**Q: How do you monitor Kafka?**
- **Lag**: `kafka-consumer-groups.sh`, or via JMX metrics, or tools like Burrow, Confluent Control Center.
- **Broker health**: under-replicated partitions, active controller count, request rates.
- **Producer metrics**: record send rate, error rate, request latency.
- Common stacks: JMX → Prometheus (JMX Exporter) → Grafana.

---

## 11. Key Numbers to Know

| Config | Default | Notes |
|---|---|---|
| `log.retention.hours` | 168 (7 days) | Message retention |
| `log.segment.bytes` | 1 GB | Log segment file size |
| `max.message.bytes` | 1 MB | Max message size (broker) |
| `replication.factor` | 1 (convention: 3) | Copies per partition |
| `default.replication.factor` | 1 | Broker-level default |
| `min.insync.replicas` | 1 | Minimum ISRs for write acceptance |
| `session.timeout.ms` | 45000 ms | Consumer heartbeat timeout |
| `max.poll.interval.ms` | 300000 ms | Max time between polls |

---

## 12. Quick Cheat Sheet

```
Topic → Partitions → Offsets (ordering within partition only)
Producer → acks (0/1/all) → Idempotent → Transactions
Consumer → Consumer Group → Rebalance → Commit (auto/sync/async)
Broker → Leader + ISR → Replication Factor → min.insync.replicas
Retention → Time or Size based (default 7 days)
Log Compaction → Latest value per key (for stateful topics)
Kafka Streams → KStream (events) vs KTable (state)
Kafka Connect → Source (in) + Sink (out) connectors
EOS = Idempotent Producer + Transactions + read_committed consumer
```
