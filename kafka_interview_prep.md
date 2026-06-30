# Kafka Interview Prep

These are written the way I'd actually explain things out loud in an interview — plain language first, technical term second. The goal isn't to recite definitions, it's to sound like someone who's actually used this stuff (which I have, on my own [Distributed Job Queue project](https://github.com/jattinmanhas/Distributed-Job-Queue)).

---

## 1. What is Kafka, really?

Think of Kafka as a **giant, durable message log** that multiple systems can write to and read from independently, without talking to each other directly.

The simplest way I explain it: imagine a notebook where you can only ever add new lines at the bottom — you can never erase or edit a line once it's written. Multiple people can read that notebook at their own pace, and re-read old pages whenever they want. That's basically a Kafka topic.

It was built at LinkedIn (2011), open-sourced, now an Apache project. People use it for:
- Real-time data pipelines (e.g., "every order placed should trigger 5 different downstream systems")
- Event-driven architectures (services reacting to "things that happened" instead of polling each other)
- Log aggregation, stream processing, etc.

**Why not just use a database or a regular queue?** That's actually a great question to be ready for — see Q&A section.

---

## 2. Core Concepts (explained like I'd say them out loud)

### Topics — "a named stream of events"
A topic is just a category name, like `orders` or `payment-events`. Producers write to it, consumers read from it.

### Partitions — "how Kafka gets parallelism"
A topic isn't one single log — it's split into multiple **partitions**, each of which IS an append-only log on its own. Splitting it up is what lets multiple consumers read the same topic in parallel.

**The catch:** ordering is only guaranteed *within* a single partition, not across the whole topic. So if you need strict ordering for a particular entity (say, all events for `user_id=42`), you make sure they always land on the same partition — usually by using that ID as the partition key.

I'd explain it like: imagine splitting one notebook into 6 separate notebooks. Each notebook keeps things in order internally, but there's no guarantee notebook 3's page 10 happened before notebook 5's page 2.

### Offsets — "a bookmark"
Every message in a partition gets a number (offset), like a line number. Consumers just remember "I'm at offset 4521" — that's literally their bookmark. Kafka stores these bookmarks in a special internal topic called `__consumer_offsets`.

### Brokers — "the servers holding the data"
A Kafka cluster = a bunch of broker machines. Each partition's data physically lives on some subset of these brokers. One broker is the **leader** for a given partition (handles all reads/writes for it), the others hold copies as **followers**.

### Producers — "the writers"
They write messages to a topic. They decide (or let Kafka decide) which partition a message goes to — usually based on hashing a key.

Important nuance people get wrong: if you *don't* provide a key, modern Kafka clients (since KIP-480, roughly 2.4+) don't round-robin every single message anymore — they use **sticky partitioning**, meaning they stick to one partition for a batch of messages before switching, purely so batches get bigger and more efficient. If you've read older blog posts saying "no key = round robin," that's outdated — good to know because interviewers sometimes ask this specifically to see if your knowledge is current.

### Consumers & Consumer Groups — "how scaling reads works"
Consumers read from topics. They're organized into **consumer groups** — Kafka guarantees that within one group, each partition is read by exactly one consumer at a time. That's literally how Kafka achieves "scale out your reads": add more consumers (up to the number of partitions), and each one gets a slice of the work.

Different consumer groups are totally independent — they can each replay the whole topic from scratch if they want.

### ZooKeeper vs KRaft — "who keeps the cluster's brain"
Older Kafka used ZooKeeper as an external system to track cluster metadata and elect leaders. Since Kafka 2.8+ (production-ready around 3.3+), **KRaft mode** removes that dependency — Kafka manages its own metadata internally using the Raft consensus algorithm. (More on this below, since it's directly relevant to my project.)

---

## 3. Architecture Deep Dive

### Replication — "why Kafka doesn't lose your data"
Each partition is copied across `replication.factor` brokers (commonly 3). One copy is the **leader** — it handles every read and write. The others are **followers**, constantly pulling new data from the leader to stay caught up.

### ISR (In-Sync Replicas) — "who's actually caught up right now"
Not every follower is guaranteed to be fully caught up at any instant (network lag, a slow broker, etc). The ones that ARE caught up are called the ISR set. This matters because of `min.insync.replicas` — if you want strong durability, you say "a write only counts as successful once at least N replicas in the ISR have it." If the ISR shrinks below that number (say a broker dies), the broker will straight up reject new writes with `NotEnoughReplicasException` rather than silently risk data loss.

I think of ISR like "who's actually on the call right now" vs the full guest list — only people on the call get a say in whether the meeting can proceed.

### Log Compaction — "keep only the latest snapshot per key"
Normal Kafka deletes old data based on time/size. **Compacted** topics instead keep just the *latest* message per key, deleting older ones with the same key. Great for things like "current account balance per user" — you don't care about every historical value, just the latest, but you still want it replayable and durable.

### Retention — "how long Kafka keeps stuff around regardless of who's read it"
Default is 7 days (`log.retention.hours=168`). This is independent of whether consumers have actually read the data — Kafka isn't a queue that deletes on consumption, it's a log that ages out.

---

## 4. Producer Internals

### Acknowledgements (`acks`) — the durability dial

| Setting | What it means | What you risk |
|---|---|---|
| `acks=0` | Don't wait for any confirmation | Could lose messages and never know |
| `acks=1` | Wait for the leader only | If the leader dies right after, before followers replicate, you lose that message |
| `acks=all` | Wait for the full ISR set | Safest, but slower since you're waiting on more machines |

I'd frame this in an interview as a classic latency-vs-durability tradeoff, and give a concrete example: for my job queue's "at-least-once delivery" guarantee, I cared more about not losing jobs than shaving milliseconds, so I'd lean toward `acks=all`.

### Idempotent Producer — "don't double-write on retry"
Network blips happen — a producer sends a message, doesn't get the ack back in time, and retries. Without protection, that could create a duplicate. Setting `enable.idempotence=true` has Kafka assign each producer a Producer ID + sequence number per partition, so the broker can recognize "I've already seen sequence #47 from this producer" and silently drop the dupe.

### Transactions — "atomic writes across partitions/topics"
Sometimes you need multiple writes (maybe to different partitions or topics) to succeed or fail together — like writing both "order placed" and "inventory decremented" atomically. Kafka transactions (`transactional.id`, `beginTransaction()`/`commitTransaction()`) make that possible.

### Batching & Compression — "how throughput actually gets squeezed out"
Producers don't send one message at a time — they group messages into batches (`batch.size`) and can wait a bit (`linger.ms`) to let a batch fill up before sending, which is way more network-efficient. Compression (`lz4`, `snappy`, `gzip`, `zstd`) is applied per batch.

**Ordering connection (important nuance):** when idempotence is enabled, Kafka guarantees ordering is preserved even with retries — but only as long as `max.in.flight.requests.per.connection` is 5 or fewer (this became the safe default once idempotence shipped). If you crank that number higher *without* idempotence, a retried batch can land out of order relative to a batch that succeeded on the first try. This is a great one to mention proactively — it shows you understand the actual mechanics, not just the buzzwords.

---

## 5. Consumer Internals

### The Poll Loop — "consumers pull, they're not pushed to"
Consumers actively call `poll()` in a loop to fetch new records — Kafka doesn't push to them. Two configs that matter a lot operationally:
- `session.timeout.ms`: if Kafka doesn't hear a heartbeat from a consumer in this window, it assumes the consumer is dead and kicks it out of the group.
- `max.poll.interval.ms`: if a consumer takes too long *between* poll calls (e.g., stuck processing a slow message), Kafka assumes it's stuck and triggers a rebalance — even if heartbeats were fine.

### Commit Strategies — "when do you mark a message as done"

| Strategy | How it works | Risk |
|---|---|---|
| Auto commit | Kafka commits offsets periodically in the background | You might crash after committing but before finishing processing → message gets skipped, OR crash before committing but after processing → message gets reprocessed |
| Manual sync commit | You call `commitSync()` yourself after processing | Safest, but blocks until Kafka confirms |
| Manual async commit | `commitAsync()` — fire and forget | Faster, but a commit could get lost on failure |

### Delivery Semantics — explain with a story
- **At-most-once**: commit *before* processing. If you crash mid-processing, that message is just gone — never retried. (Rarely what people actually want.)
- **At-least-once**: commit *after* processing. If you crash mid-processing, you'll reprocess that message next time. This is what most systems use, including mine — and it's exactly why idempotent processing on the consumer side matters (you need your downstream logic to handle "I might see this twice" gracefully).
- **Exactly-once**: needs more machinery — either idempotent consumer logic or Kafka Streams' built-in EOS.

### Consumer Group Protocol — what's actually happening under the hood
This is the part most people skip, and it's worth knowing because it explains *why* rebalances are disruptive:
1. **JoinGroup**: every consumer in the group tells the coordinator (a specific broker) "I'm here." One consumer gets elected the group leader.
2. **SyncGroup**: the group leader computes the partition assignment (using whichever assignor strategy is configured) and sends it back through the coordinator to everyone.
3. Each rebalance bumps a **generation ID** — basically a version number for "who's in the group right now" — so the coordinator can detect and reject messages from a consumer using stale group info.

### Rebalance & Assignment Strategies — "who gets which partition"
A rebalance happens when membership changes (consumer joins/leaves/dies) or partition count changes. There are a few strategies for *how* partitions get reassigned:
- **Range**: assigns contiguous partition ranges per topic — simple but can be uneven across topics.
- **RoundRobin**: spreads partitions evenly across all consumers, but during a rebalance it throws away the *entire* assignment and starts over (this is the "eager" protocol — full stop-the-world).
- **Sticky**: tries to minimize how many partitions actually move during a rebalance, while still balancing load.
- **CooperativeSticky**: combines stickiness with an *incremental* rebalance protocol — instead of revoking everyone's partitions and reassigning from scratch, only the partitions that actually need to move get revoked, and everyone else just keeps consuming uninterrupted. This is what I'd reach for in production — way less of a "stop the world" event.

You can also reduce *how often* rebalances happen at all using `group.instance.id` (static membership) — this tells Kafka "this is the same consumer coming back," so a quick restart/redeploy doesn't trigger a full rebalance.

---

## 6. KRaft Mode — since this is literally in my project, I should be ready to go deep here

The short version: Kafka used to depend on ZooKeeper as a separate system to do two jobs — store cluster metadata (which broker has which partition, configs, ACLs) and run leader elections. KRaft replaces that external dependency with Kafka managing its own metadata using the **Raft consensus algorithm**, internally.

How I'd explain it conversationally:
- A small set of brokers are designated as **controllers** (the quorum). One of them is the **active controller** at any time — elected via Raft, the same way any Raft-based system elects a leader (majority vote among the quorum).
- All metadata changes (new topic created, partition leader changed, broker joined/left) get written to a special internal log called `__cluster_metadata` — it's literally just another Kafka-style append-only log, which is a nice "Kafka eating its own dog food" detail to mention.
- Other controllers (and eventually all brokers) replicate that log to stay in sync, the same general idea as ISR replication for regular partitions.
- If the active controller dies, the remaining quorum members elect a new one via Raft — no external ZooKeeper ensemble needed.

**Why this is better, in plain terms:** fewer moving parts (one system instead of two), faster controller failover (Raft elections are quicker than the old ZK-based controller failover), and it scales to way more partitions since metadata propagation doesn't have to go through ZooKeeper's coordination overhead.

**Connecting to my own project:** I used franz-go in KRaft mode, so a good thing to mention is that I didn't have to stand up a separate ZooKeeper service at all — just Kafka brokers configured with `process.roles=broker,controller` (or split controller/broker nodes for a "real" production-like setup).

---

## 7. Kafka Streams (lighter section — know it, but it's less likely to be deep-dived if it's not in your project)

It's a **client library** (not a separate cluster) for processing streams directly — `map`, `filter`, `groupBy`, `aggregate`, `join`, windowing, etc.

### KStream vs KTable — the analogy that always lands
- **KStream** = an event log. Every record matters, nothing gets overwritten. Think "every transaction that ever happened."
- **KTable** = a snapshot/changelog. Only the latest value per key matters. Think "current balance per account" — like a materialized view built by folding a KStream by key.

### GlobalKTable
Like a KTable, but the *entire* table is copied to every app instance (not partitioned). Useful for small reference/lookup data that every node needs locally, like a currency-code-to-symbol mapping.

### Windowing types
- **Tumbling**: fixed, non-overlapping (e.g., every 5 minutes, no overlap).
- **Hopping**: fixed size but overlapping (e.g., 5-min windows, sliding forward every 1 min).
- **Sliding**: defined by the time gap between individual records, not a fixed clock.
- **Session**: groups records that occur close together in time, closes the window after a gap of inactivity.

---

## 8. Kafka Connect (also lighter — good to know exists, not core to my project)

A framework for moving data in/out of Kafka without writing custom producer/consumer code.
- **Source connectors** pull data INTO Kafka (e.g., MySQL → Kafka via CDC).
- **Sink connectors** push data OUT of Kafka (e.g., Kafka → Elasticsearch).
- Configured via JSON over a REST API, runs standalone or distributed.

---

## 9. Performance & Tuning — frame these as tradeoffs, not just config names

**For throughput** (producer): bigger `batch.size`, higher `linger.ms` (wait longer to build bigger batches), use `lz4`/`snappy` for fast compression, bump `buffer.memory`.

**For throughput** (consumer): higher `fetch.min.bytes` and `fetch.max.wait.ms` (wait for more data before returning from a poll), parallelize processing within the consumer.

**For latency** (when throughput isn't the priority): `linger.ms=0` (send immediately, don't wait to batch), `acks=1` instead of `all`, lower `fetch.max.wait.ms`.

**Partition count** is its own tradeoff: more partitions = more parallelism, but also more file handles per broker, more replication traffic, and (in KRaft) more metadata to propagate. A reasonable way to size it: figure out your target throughput, divide by realistic per-partition throughput, that's roughly your partition count.

---

## 10. Exactly-Once Semantics (EOS) — the "all three pieces" answer

If asked "how do you get exactly-once in Kafka," the strong answer is to name all three layers, not just one:
1. **Idempotent producer** — broker-side dedup of retried writes.
2. **Transactions** — atomic writes across multiple partitions/topics.
3. **`isolation.level=read_committed`** on the consumer — so it only ever reads data from committed transactions, never sees in-flight/aborted writes.

In Kafka Streams, `processing.guarantee=exactly_once_v2` wires all three of these up for you automatically.

---

## 11. Operational Scenarios — failure stories I should be ready to walk through out loud

**"What happens when a leader broker dies?"**
Kafka detects it via missed heartbeats to the controller. The (KRaft) active controller picks a new leader from the current ISR list for every partition that broker led, updates `__cluster_metadata`, and clients refresh their metadata and start talking to the new leader. Brief unavailability for writes on those partitions during the handover, but no data loss as long as the new leader was actually in the ISR.

**"What if the ISR shrinks below `min.insync.replicas`?"**
New writes with `acks=all` get rejected outright (`NotEnoughReplicasException`) rather than silently accepted with weaker durability. This is Kafka choosing safety over availability — a good thing to flag if asked about CAP-theorem-style tradeoffs.

**"What's a 'preferred leader election' and why does it matter?"**
When a broker recovers after being down, it doesn't automatically reclaim leadership for its partitions — by default, leadership stays wherever it failed over to, even after the original broker is healthy again. Over time this can leave leadership unevenly distributed across the cluster. A preferred leader election rebalances leadership back to the "preferred" (originally assigned) replica once it's healthy, so load doesn't pile up on whichever broker happened to absorb the failover.

**"How do you handle a lagging consumer group?"**
Check actual lag with `kafka-consumer-groups.sh --describe`, then: scale consumers up to partition count, optimize processing (batch, parallelize, go async), and if it's urgent, bump retention so you don't lose data while you catch up.

**"How do you migrate a topic to more partitions?"**
`kafka-topics.sh --alter --partitions N`. Important gotcha to mention: existing messages stay put in their original partitions — only new messages get spread across the new partition count. If you relied on key-based ordering, this can quietly break it, so it needs care and testing, not just running the command.

---

## 12. Connecting concepts to my own project (use these as real examples, not hypotheticals)

- **Retry logic + exponential backoff + DLQ**: directly maps to the "what happens when a consumer fails mid-processing" and "what's a dead letter queue" questions — I'm not describing a textbook concept, I built the actual failure path.
- **At-least-once delivery with a goroutine-based worker pool**: ties straight into the delivery semantics section — I can explain *why* I chose at-least-once over at-most-once (didn't want to silently drop jobs) and how my workers handle potential duplicate processing.
- **Redis sliding-window rate limiting**: not Kafka-specific, but a good example if asked about backpressure or protecting downstream systems from a burst of messages.
- **franz-go in KRaft mode**: my answer to "have you worked with KRaft" isn't theoretical — I can talk about not needing to stand up ZooKeeper at all.

---

## 13. Common Interview Q&A (kept, lightly tightened)

**Q: What's the difference between a queue and Kafka?**
Traditional queues (RabbitMQ, SQS) delete a message once it's consumed. Kafka keeps messages around for a retention window regardless of consumption, and lets multiple independent consumer groups replay the same data from any point.

**Q: How does Kafka guarantee ordering?**
Only within a single partition. If you need a strict global order for a topic, you're stuck with one partition — which means you give up parallelism to get it.

**Q: What happens when a consumer crashes mid-processing?**
With manual commit, the offset was never committed, so on restart the message gets re-read — at-least-once. If auto-commit happened to fire right before the crash, that message could be skipped — at-most-once-style loss, even though auto-commit is generally framed as "at-least-once" in the common case.

**Q: What is the `__consumer_offsets` topic?**
An internal, compacted Kafka topic that stores each consumer group's committed offsets. Replaced the old ZooKeeper-based offset storage back in Kafka 0.9.

**Q: What's the difference between `replication.factor` and `min.insync.replicas`?**
`replication.factor` is the total number of copies of a partition that exist. `min.insync.replicas` is the minimum number that must be caught up for a write (with `acks=all`) to succeed. One's about how many copies exist; the other's about how many need to be *healthy* right now.

**Q: What's a GlobalKTable, and how's it different from a regular KTable?**
A regular KTable is partitioned like any other topic — each app instance only sees its assigned partitions. A GlobalKTable is fully replicated to every instance, which is useful for small reference data that every node needs to look up locally without a network hop.

**Q: How do you monitor a Kafka cluster?**
Consumer lag via `kafka-consumer-groups.sh` or JMX metrics (tools like Burrow or Confluent Control Center help here too). Broker health via under-replicated partition count and active controller count. Producer health via send/error rates and request latency. Common stack: JMX → Prometheus (JMX Exporter) → Grafana.

---

## 14. Key Numbers to Know

| Config | Default | What it controls |
|---|---|---|
| `log.retention.hours` | 168 (7 days) | How long messages stick around |
| `log.segment.bytes` | 1 GB | Size of each log segment file |
| `max.message.bytes` | 1 MB | Largest single message a broker accepts |
| `replication.factor` | 1 (commonly set to 3) | Copies per partition |
| `min.insync.replicas` | 1 | Minimum healthy replicas needed for a write to succeed |
| `session.timeout.ms` | 45000 ms | How long before a silent consumer is kicked from its group |
| `max.poll.interval.ms` | 300000 ms | Max time allowed between poll() calls before triggering a rebalance |
| `max.in.flight.requests.per.connection` | 5 (safe default with idempotence on) | How many unacked requests can be in flight — affects ordering on retry |

---

## 15. Quick Mental Model (say this out loud as a 30-second summary if asked "explain Kafka")

> "Kafka's an append-only, distributed log. Topics are split into partitions for parallelism, each partition has one leader broker and some in-sync replicas for durability. Producers write with a durability/latency tradeoff controlled by `acks`. Consumers in a group split up the partitions and track their position with offsets. Since KRaft, the cluster manages its own metadata and leader elections internally via Raft, instead of depending on ZooKeeper. And depending on how you commit offsets and configure idempotence/transactions, you can tune the whole pipeline anywhere from at-most-once up to exactly-once."

That's the whole system in one breath — everything else is detail on top of that.
