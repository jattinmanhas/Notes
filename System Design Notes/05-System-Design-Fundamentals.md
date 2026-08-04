# System Design Fundamentals — The Complete Reference

> **How to use this:** these are the building blocks, not the questions. Every design question (URL shortener, news feed, chat, Uber) is a *recombination* of the pieces below. Learn the pieces and their trade-offs and you can derive any design; memorise designs and you fall apart on the first follow-up.
>
> **The single most important habit:** every section here has a "when NOT to use it". Interviewers grade on *trade-off reasoning*, not vocabulary. Anyone can say "add a cache" — the signal is knowing what it costs you.

---

## Table of Contents

**Part 0 — Running the interview**
- [0. The 6-Step Framework](#0-the-6-step-framework)

**Part 1 — Foundations**
- [1. Back-of-the-Envelope Estimation](#1-back-of-the-envelope-estimation)
- [2. Non-Functional Vocabulary](#2-non-functional-vocabulary)

**Part 2 — Distributed systems theory**
- [3. CAP and PACELC](#3-cap-and-pacelc)
- [4. Consistency Models](#4-consistency-models)
- [5. Consensus, Quorums and Leader Election](#5-consensus-quorums-and-leader-election)
- [6. Failure Detection](#6-failure-detection)
- [7. Time, Clocks and Ordering](#7-time-clocks-and-ordering)
- [8. Idempotency and Delivery Semantics](#8-idempotency-and-delivery-semantics)

**Part 3 — Scaling primitives**
- [9. Scaling Basics](#9-scaling-basics) · [10. Load Balancing](#10-load-balancing) · [11. Consistent Hashing](#11-consistent-hashing) · [12. Sharding](#12-sharding) · [13. Replication](#13-replication) · [14. Caching](#14-caching) · [15. CDN](#15-cdn)

**Part 4 — Data**
- [16. SQL vs NoSQL](#16-sql-vs-nosql) · [17. Index Internals](#17-index-internals) · [18. Normalisation](#18-normalisation-vs-denormalisation) · [19. ACID vs BASE](#19-acid-vs-base) · [20. Distributed Transactions](#20-distributed-transactions) · [21. Blob Storage](#21-blob-storage)

**Part 5 — Communication**
- [22. Sync vs Async](#22-sync-vs-async) · [23. Protocols](#23-protocols) · [24. Queues vs Streams](#24-queues-vs-streams) · [25. API Design](#25-api-design)

**Part 6 — Reliability**
- [26. Rate Limiting](#26-rate-limiting) · [27. Resilience Patterns](#27-resilience-patterns) · [28. Observability](#28-observability) · [29. Deployment](#29-deployment) · [30. Disaster Recovery](#30-disaster-recovery)

**Part 7 — Specialised**
- [31. Security](#31-security) · [32. Unique ID Generation](#32-unique-id-generation) · [33. Geospatial](#33-geospatial-indexing) · [34. Probabilistic Structures](#34-probabilistic-data-structures) · [35. Search](#35-search)

**Part 8 — [The Interview Itself](#part-8--the-interview-itself)**

---

# Part 0 — Running the Interview

## 0. The 6-Step Framework

45 minutes. Do not start drawing boxes at minute 2 — that's the most common way to fail.

| Step | Time | What you do |
|---|---|---|
| **1. Requirements** | 5–8 min | Functional scope, then non-functional (scale, latency, consistency). **Write them down.** Explicitly cut things: "I'll treat analytics as out of scope." |
| **2. Estimation** | 3–5 min | QPS, storage, bandwidth. Only compute the numbers that will *change a decision*. |
| **3. API + Data model** | 5 min | A handful of endpoints and the core entities. This forces concreteness before architecture. |
| **4. High-level design** | 10 min | Boxes and arrows. Client → LB → service → cache → DB, plus async paths. Walk one request end to end. |
| **5. Deep dive** | 10–15 min | The interviewer picks, or you offer: "the interesting part here is the fan-out — shall I go deeper there?" |
| **6. Bottlenecks & wrap** | 5 min | Single points of failure, hot spots, what breaks at 10×, what you'd monitor. |

### The questions to always ask in step 1

- **Scale:** DAU? Reads vs writes ratio? Peak vs average (assume peak ≈ 2–3× average)?
- **Latency:** p99 target? Is this user-facing or a background job?
- **Consistency:** Is stale data acceptable, and for how long? *(This one question shapes the entire design.)*
- **Availability:** How many nines? What's the cost of downtime?
- **Read or write heavy?** Determines whether you optimise with caching/replicas or with sharding/queues.
- **Retention:** How long is data kept? Changes storage math by orders of magnitude.

> **The reframe that scores points:** don't ask "what's the scale?" — ask "**what's the read:write ratio and what's the consistency requirement?**" Those two answers determine almost every subsequent decision, and asking them shows you know that.

### What loses points

| Anti-pattern | What to do instead |
|---|---|
| Jumping to architecture before requirements | Spend the first 5 minutes on scope. Always. |
| Buzzword soup ("Kafka, Redis, Cassandra, K8s") | Name the problem *first*, then the tool that solves it |
| No trade-offs — every choice presented as obviously correct | "I'd pick X over Y because we're read-heavy and can tolerate staleness" |
| Designing for a billion users when they said 10k | Match the design to the stated scale; mention how you'd evolve it |
| Silence while thinking | Narrate. The interviewer is grading your reasoning, and they can't grade silence |
| Ignoring the interviewer's hints | "How would that behave if the queue backs up?" is not curiosity — it's a rescue attempt |

---

# Part 1 — Foundations

## 1. Back-of-the-Envelope Estimation

### The numbers to memorise

**Time and scale:**

```
1 day       ≈ 86,400 s   ≈ 10⁵ s        ← the single most useful approximation
1 month     ≈ 2.6 × 10⁶ s
1 year      ≈ 3.15 × 10⁷ s

1 million writes/day    ≈ 12 QPS
1 billion writes/day    ≈ 12,000 QPS
1 billion  = 10⁹        1 trillion = 10¹²

Peak QPS ≈ 2–3 × average QPS      ← always state this multiplier
```

**Storage:**

```
char/ASCII byte  1 B      UUID (string)     36 B     UUID (binary)   16 B
int              4 B      long / double      8 B     timestamp        8 B
Typical row      ~1 KB    Tweet (text+meta) ~1 KB    Photo         ~1–5 MB
1 KB = 10³ B     1 MB = 10⁶ B    1 GB = 10⁹ B    1 TB = 10¹² B    1 PB = 10¹⁵ B
```

**Latency (order of magnitude — nobody expects exact figures):**

| Operation | Time | Intuition |
|---|---|---|
| L1 cache reference | ~1 ns | |
| Branch mispredict | ~3 ns | |
| L2 cache reference | ~4 ns | |
| Mutex lock/unlock | ~20 ns | |
| Main memory reference | ~100 ns | **100× slower than L1** |
| Read 1 MB sequentially from RAM | ~3 µs | |
| SSD random read | ~16 µs | **~100× faster than a disk seek** |
| Read 1 MB from SSD | ~50 µs | |
| Round trip within a datacenter | ~500 µs | **~0.5 ms** |
| HDD seek | ~2 ms | |
| Read 1 MB from HDD | ~2 ms | |
| Round trip CA ↔ Netherlands | ~150 ms | **speed of light is the floor** |

**The three ratios that matter more than the absolute numbers:**

1. **Memory is ~100× faster than SSD; SSD is ~100× faster than an HDD seek.** → why caching works.
2. **A cross-continent round trip (~150 ms) is ~300× a same-DC round trip (~0.5 ms).** → why CDNs and regional deployments exist.
3. **Sequential I/O is orders of magnitude faster than random I/O**, on both disk and SSD. → why LSM trees, append-only logs and Kafka are designed the way they are.

### Worked example — "Design Twitter"

```
Assumptions (STATE THEM, then move on):
  300 M MAU, 50% daily active        → 150 M DAU
  Each user posts 2 tweets/day       → 300 M tweets/day
  Read:write ratio 100:1             → 30 B timeline reads/day

Write QPS:  300 M / 10⁵ s   = 3,000 QPS       peak ≈ 9,000 QPS
Read  QPS:  30 B  / 10⁵ s   = 300,000 QPS     peak ≈ 900,000 QPS
                                    ↑ THIS is the number that drives the design

Storage:    300 M tweets/day × 1 KB = 300 GB/day
            × 365 × 5 years          ≈ 550 TB      → sharding is mandatory
Media:      10% of tweets have a 1 MB image → 30 TB/day → blob storage + CDN

Bandwidth:  read 300 K QPS × 1 KB    = 300 MB/s egress for text alone
```

**The conclusion you draw from this**, and the only reason to do the maths: reads outnumber writes 100:1, so **precompute the timeline on write (fan-out on write)** rather than assembling it on read — unless the user has 100 M followers, in which case you fall back to fan-out on read. That hybrid *is* the answer, and the estimate is what justifies it.

> **Interview technique:** round aggressively (300 M/day → 3,000 QPS, not 3,472). Say "roughly" a lot. The point is the order of magnitude and the decision it drives, not arithmetic.

---

## 2. Non-Functional Vocabulary

### Availability — the nines

| Availability | Downtime/year | Downtime/month | Typical use |
|---|---|---|---|
| 99% ("two nines") | 3.65 days | 7.2 hours | Internal tools |
| 99.9% ("three nines") | 8.77 hours | 43 min | Standard SaaS SLA |
| 99.95% | 4.38 hours | 22 min | Paid tiers |
| 99.99% ("four nines") | 52.6 min | 4.3 min | Serious infrastructure |
| 99.999% ("five nines") | 5.26 min | 26 s | Telecom, payments core |

**Two rules interviewers probe:**

1. **Components in series multiply.** A request through LB → service → DB, each 99.9%, gives `0.999³ ≈ 99.7%` — *worse* than any single component. More hops = less available.
2. **Redundancy in parallel adds nines.** Two independent 99% replicas give `1 - 0.01² = 99.99%` — *if* failures are truly independent. They usually aren't (same rack, same AZ, same bad deploy), which is why you spread across availability zones.

### Latency: percentiles, not averages

**Never quote an average latency.** With 1% of requests at 5 s and 99% at 10 ms, the mean is ~60 ms — a number that describes *no actual user's experience*.

| Metric | Meaning | Why it matters |
|---|---|---|
| p50 (median) | Typical request | Ignores the tail entirely |
| p95 / p99 | The slow ones | **This is what users complain about** |
| p999 | The very tail | Matters at scale — 0.1% of 1 B requests is 1 M unhappy users |

**Tail amplification** — the killer follow-up: if one page issues 100 parallel sub-requests, and each has a p99 of 100 ms, then the probability that *all 100* are fast is `0.99¹⁰⁰ ≈ 37%`. So **~63% of page loads hit at least one p99 request.** The service's p99 becomes the page's typical case. Mitigations: fewer fan-out calls, hedged requests (send a duplicate after a delay, take the first response), and tighter timeouts.

### Latency vs throughput

- **Latency** = time for one request. **Throughput** = requests per second.
- They trade off: batching raises throughput and raises latency. Adding queues raises throughput and raises latency under load.
- **Little's Law:** `Concurrency = Throughput × Latency`. To serve 1,000 QPS at 200 ms each you need ~200 concurrent slots. This is exactly how you size thread pools and connection pools — a great thing to drop into a capacity discussion.

### SLI / SLO / SLA

| Term | Meaning | Example |
|---|---|---|
| **SLI** | The *measurement* | "Fraction of requests served in < 200 ms" |
| **SLO** | Your internal *target* | "99.9% of requests < 200 ms over 30 days" |
| **SLA** | The *contract* with money attached | "99.9% uptime or you get a 10% credit" |

**Error budget:** a 99.9% SLO gives you 0.1% of requests to fail — 43 minutes a month. That budget is *meant to be spent* on deploys and risk. If you're not consuming it, you're shipping too slowly. This framing is a strong senior-level signal.

### Durability vs availability

Distinct properties, often conflated:

- **Durability** — once acknowledged, data is not lost. S3 advertises 11 nines of durability.
- **Availability** — you can reach it right now. S3's availability SLA is far lower than its durability.

A system can be durable but unavailable (data safe, service down). The reverse — available but not durable — is far worse and is the failure mode of "we acknowledged the write before it was replicated".

---

# Part 2 — Distributed Systems Theory

## 3. CAP and PACELC

### CAP, stated precisely

Among **Consistency** (every read sees the most recent write), **Availability** (every request gets a non-error response), and **Partition tolerance** (the system keeps working when the network drops messages), you can guarantee only two.

**The correction that scores points:** *"Partitions aren't a choice — they're a fact of networks. So CAP isn't 'pick two', it's 'when a partition happens, do you sacrifice consistency or availability?' In practice every distributed system is either CP or AP."*

| Choice | Behaviour during a partition | Examples |
|---|---|---|
| **CP** | Refuse requests on the minority side rather than serve stale data | ZooKeeper, etcd, HBase, MongoDB (default), Spanner |
| **AP** | Serve possibly-stale data on both sides, reconcile later | Cassandra, DynamoDB (tunable), Riak, DNS |

**Choosing in an interview:** ask what a wrong answer costs. Bank balance → CP. Like count → AP. Shopping cart → AP (Amazon's original Dynamo paper: a cart that can't accept an item loses money; a cart that occasionally resurrects a deleted item does not).

### PACELC — the extension nobody mentions, so mention it

> **If Partition** → trade **A**vailability vs **C**onsistency; **E**lse (normal operation) → trade **L**atency vs **C**onsistency.

This is the more useful framing because partitions are rare and **the latency-vs-consistency trade-off is happening every single millisecond**. Synchronous replication to three replicas is consistent and slow; asynchronous is fast and stale.

| System | PACELC |
|---|---|
| Cassandra / Dynamo | PA/EL — available under partition, low latency otherwise |
| Spanner | PC/EC — consistent always, pays in latency (TrueTime commit-wait) |
| MongoDB | PC/EC by default, tunable |

> **The line to use:** *"CAP only tells you about the rare partition case. PACELC points out that even with a healthy network you're trading latency against consistency on every request — and that's the trade-off that actually shapes the design."*

---

## 4. Consistency Models

A spectrum, strongest to weakest. Know at least the top and bottom plus read-your-writes.

| Model | Guarantee | Cost | Example use |
|---|---|---|---|
| **Linearizable** (strong) | Every read sees the latest committed write; the system behaves as if there's one copy | Coordination on every op; high latency; unavailable under partition | Account balances, locks, leader election |
| **Sequential** | All nodes see operations in the same order, not necessarily real time | Cheaper than linearizable | |
| **Causal** | Operations causally related are seen in order; concurrent ones may differ | Needs vector clocks / dependency tracking | Comment threads (reply never appears before its parent) |
| **Read-your-writes** | *You* always see your own writes; others may lag | Route the user's reads to the primary, or pin by session | Post a tweet → you must see it in your own timeline |
| **Monotonic reads** | You never see time move backwards | Sticky sessions to one replica | Refresh doesn't lose a comment you just saw |
| **Eventual** | If writes stop, replicas converge — eventually | Cheapest, most available | View counts, DNS, follower counts |

**Two more distinctions worth knowing:**

- **Linearizability vs serializability.** Linearizability is about *recency* on a single object (real-time ordering). Serializability is about *transactions* over multiple objects behaving as if run one at a time. **Strict serializability** = both. Interviewers love this because most candidates use the words interchangeably.
- **Conflict resolution in eventually consistent systems:** Last-Write-Wins (simple, silently loses data — needs synchronised clocks), vector clocks (detects concurrency, pushes resolution to the app), or **CRDTs** (data types that mathematically always converge — counters, sets, sequences). CRDTs are the right answer for collaborative editing and distributed counters.

> **The practical answer for most designs:** *"Strong consistency where money or identity is involved, read-your-writes for anything the user just did, eventual everywhere else."* That sentence handles 90% of consistency follow-ups.

---

## 5. Consensus, Quorums and Leader Election

### Why consensus is hard

**FLP impossibility:** in an asynchronous network with even one faulty process, no deterministic algorithm can guarantee consensus. Real systems dodge this with timeouts and randomised elections — they give up *guaranteed* termination for termination *with probability 1*.

### Quorums — the practical tool

With `N` replicas, a write needs `W` acks and a read needs `R` responses:

```
W + R > N   ⇒   read and write quorums overlap ⇒ reads see the latest write
W > N/2     ⇒   no two writes can succeed concurrently (prevents split brain)
```

| Config (N=3) | Behaviour |
|---|---|
| W=3, R=1 | Fast reads, slow writes, no write availability if any node is down |
| W=1, R=3 | Fast writes, slow reads — good for write-heavy logging |
| **W=2, R=2** | **Balanced; tolerates one node failure.** The usual answer |
| W=1, R=1 | Fast and eventually consistent — no overlap guarantee |

This is exactly Cassandra/DynamoDB's tunable consistency, and being able to derive it live is a strong signal.

### Raft in 60 seconds

Know Raft, not Paxos — it's designed to be explainable, and that's why interviewers ask about it.

1. **Leader election.** Nodes are follower/candidate/leader. A follower that hears no heartbeat within a **randomised** election timeout becomes a candidate and requests votes. Majority wins. Randomisation is what prevents repeated split votes.
2. **Log replication.** All writes go to the leader, which appends to its log and replicates. Once a **majority** has persisted an entry, it's committed and applied.
3. **Safety.** A candidate can only win if its log is at least as up to date as the voter's — which guarantees committed entries are never lost.

**Where you'll actually meet it:** etcd, Consul, ZooKeeper (ZAB, similar), Kafka's KRaft controller, CockroachDB, TiKV. In a design, the honest answer is *"I'd use etcd/ZooKeeper for leader election rather than implement consensus myself"* — and interviewers respect that.

### Split brain

Two nodes both believe they're the leader after a partition, both accept writes, data diverges. Defences: **majority quorum** (a minority partition can't elect a leader), **fencing tokens** (monotonic epoch number; storage rejects writes from a stale leader), and **STONITH** (forcibly kill the old leader).

> This is the same fencing-token argument as the distributed-lock critique in the seat-booking notes — a lock with a TTL is not mutual exclusion without one.

---

## 6. Failure Detection

You cannot distinguish "crashed" from "slow" from "network partitioned". Every failure detector is a heuristic trading detection speed against false positives.

| Mechanism | How | Trade-off |
|---|---|---|
| **Heartbeats** | Periodic "I'm alive"; miss N in a row → dead | Simple; short timeout = fast detection + false positives, long = the reverse |
| **Phi-accrual** | Outputs a *suspicion level* from the history of arrival times, not a boolean | Adapts to network conditions. Used by Cassandra and Akka |
| **Gossip** | Nodes randomly exchange membership state | Scales to thousands of nodes, no central coordinator; converges in O(log N) rounds |

**The consequence that matters for design:** because a false positive is always possible, the system must be **safe when it's wrong**. That means fencing tokens, idempotent operations, and leases rather than locks. A design that assumes failure detection is accurate is a design that will double-process.

---

## 7. Time, Clocks and Ordering

### Never trust wall-clock time across machines

- Clock drift is real (~10–100 ppm without NTP; seconds per day).
- NTP sync itself has error (~1–100 ms) and can step the clock **backwards**.
- **Therefore:** never use wall-clock comparison across machines to order events or decide who wins a conflict.
- In Java: use `System.nanoTime()` for durations (monotonic) and `System.currentTimeMillis()` only for wall-clock display. `nanoTime` is not comparable across machines.

**Last-Write-Wins is a data-loss bug in disguise** when clocks are skewed — the write with the further-ahead clock wins regardless of actual order. Say this if someone proposes LWW.

### Logical clocks

| Clock | What it gives you | Cost |
|---|---|---|
| **Lamport timestamp** | A total order consistent with causality. `a → b` implies `L(a) < L(b)` — but **not** the converse | One counter per node |
| **Vector clock** | Detects *concurrency*: you can tell "a before b", "b before a", or "concurrent" | O(N) space per event |
| **Hybrid Logical Clock** | Logical ordering that stays close to physical time | Used by CockroachDB |
| **TrueTime** (Spanner) | An *interval* `[earliest, latest]` from GPS + atomic clocks; commit-wait until the uncertainty passes | Requires special hardware; gives external consistency |

**The one-liner:** *"Lamport clocks give you a total order but can't tell you whether two events were concurrent. Vector clocks can, at O(N) space per event. Spanner buys real-time ordering with atomic clocks and pays for it by waiting out the uncertainty window on every commit."*

---

## 8. Idempotency and Delivery Semantics

### The three semantics

| Semantic | Meaning | Reality |
|---|---|---|
| **At-most-once** | Fire and forget; may be lost | Metrics, non-critical logs |
| **At-least-once** | Retried until acked; **may be duplicated** | **The default for nearly every real system** |
| **Exactly-once** | Delivered precisely once | **Impossible at the network layer.** Achieved as *effectively-once* = at-least-once delivery + idempotent processing |

> **The line interviewers want to hear:** *"Exactly-once delivery is impossible — the two generals problem. What you can build is exactly-once **processing**: at-least-once delivery plus an idempotent consumer, usually via a dedup key stored transactionally with the effect."*

Kafka's "exactly-once semantics" is precisely this: idempotent producers (sequence numbers per partition) plus transactional writes that tie the output and the consumer offset into one atomic commit. It works *within* Kafka; the moment you write to an external system, you're back to needing idempotency.

### How to make an operation idempotent

1. **Natural idempotency** — `SET x = 5` is idempotent; `x += 5` is not. Prefer absolute over relative operations.
2. **Idempotency key** — the client supplies a unique key; the server stores `key → result` and replays the stored result on a repeat. *(This is exactly the payment-retry design.)*
3. **Unique constraint** — let the database reject the duplicate. The most reliable option, because it can't be bypassed.
4. **Conditional update / CAS** — `UPDATE … WHERE version = ?`. Applying twice is a no-op the second time.
5. **Dedup window** — store recent request IDs in Redis with a TTL. Cheap, but only correct within the window.

### The outbox pattern — the dual-write problem

You cannot atomically write to the database *and* publish to Kafka. If you write then publish, a crash in between loses the event; publish then write, and you can emit an event for a transaction that rolled back.

**Fix:** write the event to an `outbox` table **in the same transaction** as the business data. A separate relay (or CDC via Debezium reading the WAL) publishes from the outbox and marks rows sent. At-least-once, so consumers must be idempotent — but nothing is ever lost or phantom-published.

```
BEGIN;
  INSERT INTO orders (...);
  INSERT INTO outbox (event_type, payload, created_at) VALUES ('OrderCreated', ..., now());
COMMIT;
-- relay: SELECT unsent FROM outbox → publish to Kafka → mark sent
```

This is one of the highest-value patterns to know. It comes up in any design involving a database and a message broker, which is most of them.

---

# Part 3 — Scaling Primitives

## 9. Scaling Basics

| | **Vertical** (bigger machine) | **Horizontal** (more machines) |
|---|---|---|
| Complexity | Trivial — no code changes | High — distribution, consistency, coordination |
| Ceiling | Hard physical limit | Effectively unlimited |
| Failure | **Single point of failure** | Redundancy built in |
| Cost curve | Superlinear (big machines cost disproportionately more) | Roughly linear |
| When | Early stage; databases, where it's often the *right* first move | Stateless services; anything past one machine's limit |

**Say this:** *"Vertical scaling is underrated — a modern box does a lot, and it avoids an enormous amount of distributed-systems complexity. I'd scale up before scaling out, and scale out the stateless tier before the stateful one."*

### Stateless vs stateful — the fundamental split

**Make services stateless.** A stateless service can be load-balanced trivially, autoscaled, and killed at will. Push state into: databases (durable), caches (ephemeral shared), object storage (blobs), or the client (JWT, cursor tokens).

State is what makes scaling hard. Every hard problem in Part 3 and 4 is a consequence of state.

### Scaling ladder — the order you actually do it

```
1. One server (app + DB)
2. Split app and DB onto separate machines
3. Add a load balancer + multiple stateless app servers
4. Add a cache (biggest single win for read-heavy systems)
5. Add read replicas
6. Add a CDN for static assets
7. Move heavy work off the request path → message queue + workers
8. Shard the database                        ← the expensive, irreversible one
9. Split into services / regional deployment
```

**Steps 8 and 9 are where complexity explodes.** Delay them as long as possible, and say so — knowing when *not* to distribute is a senior signal.

---

## 10. Load Balancing

### L4 vs L7

| | **L4 (transport)** | **L7 (application)** |
|---|---|---|
| Sees | IP + port | Full HTTP: path, headers, cookies, body |
| Can do | Forward the connection | Route by URL, TLS termination, compression, rewrite, WAF |
| Speed | Faster, less CPU | Slower, far more capable |
| Examples | AWS NLB, IPVS | AWS ALB, NGINX, Envoy, HAProxy |

Use L7 for HTTP microservices (path-based routing, canaries, header-based routing). Use L4 for raw TCP, extreme throughput, or when you need to preserve the client IP without headers.

### Algorithms

| Algorithm | Behaviour | Use when |
|---|---|---|
| **Round robin** | Rotate through servers | Homogeneous servers, uniform requests |
| **Weighted round robin** | Proportional to capacity | Mixed instance sizes |
| **Least connections** | Send to the least busy | **Variable request durations — usually the best default** |
| **Least response time** | Combines connections + observed latency | Latency-sensitive services |
| **IP hash / consistent hash** | Same client → same server | Sticky sessions, cache locality |
| **Random with two choices** | Pick 2 at random, take the less loaded | **Near-optimal with almost no coordination — great answer** |

> **"Power of two choices"** is worth naming explicitly: sampling two servers and choosing the less loaded gives exponentially better load distribution than pure random, without the global state that least-connections needs. It's how many modern proxies work.

### Health checks

- **Passive** — observe real traffic for errors. Zero overhead, slow to react.
- **Active** — periodic probe to `/health`. Fast, costs traffic.
- **Shallow vs deep:** shallow returns 200 if the process is up; deep checks the DB and dependencies. **Deep checks cause cascading failures** — if the DB blips, every instance reports unhealthy and the LB removes the entire fleet. Prefer shallow liveness + separate readiness, and never fail a health check for a *downstream* problem you could degrade around.

### Sticky sessions

Route a user to the same server via cookie or IP hash. **Generally an anti-pattern** — it breaks even load distribution, complicates deploys, and loses state on failure. Prefer stateless services with shared session storage (Redis) or signed tokens. Legitimate uses: WebSocket connections and in-memory caches with strong locality.

### Avoiding the LB as a single point of failure

Active-passive pair with a floating/virtual IP, or DNS round robin across multiple LBs, or anycast. Mention that the LB itself needs redundancy — many candidates draw one box and stop.

---

## 11. Consistent Hashing

### The problem it solves

With `N` cache servers and `server = hash(key) % N`, adding one server changes the modulus and **almost every key remaps**. Cache hit rate collapses to ~0 and the origin gets crushed. Same problem for shard assignment.

### The mechanism

1. Map the hash output onto a **ring** (0 … 2³² − 1).
2. Place each **server** on the ring at `hash(serverId)`.
3. Place each **key** on the ring at `hash(key)`.
4. A key belongs to the **first server clockwise** from its position.

Now adding or removing a server only affects the keys between it and its neighbour: **only ~K/N keys move**, not all of them.

```
          hash ring (0 ─────────────────► 2³²)
                    ┌──────────────────┐
             S1 ────┤                  ├──── S2
                    │   k1  k2    k3   │
             S4 ────┤                  ├──── S3
                    └──────────────────┘
     k1 → first server clockwise → S2
     Remove S2 → only k1, k2, k3 move to S3. Everything else is untouched.
```

### Virtual nodes — the part that's always asked

With few servers, the ring is unevenly divided and load is lopsided. Also, when a server dies, **all** its load lands on exactly one neighbour.

**Fix:** each physical server gets `V` positions on the ring (typically 100–200), hashed as `hash(serverId + "#" + i)`.

- Load evens out (standard deviation shrinks as ~1/√V).
- When a server fails, its load spreads across **many** neighbours instead of one.
- Heterogeneous capacity is easy: give a bigger machine more virtual nodes.

### Where it's used, and where it isn't

**Used by:** Memcached/Redis client-side sharding, Cassandra and DynamoDB partitioning, Envoy's ring-hash balancer, CDN request routing.

**Don't reach for it when:** you have a fixed shard count with an explicit lookup table (a directory-based approach is simpler and lets you move individual shards deliberately), or when your data store already handles rebalancing for you.

> **Complexity:** with a sorted structure over the ring (a `TreeMap` in Java — `ceilingEntry` then wrap to `firstEntry`), lookup is **O(log(N·V))**.

---

## 12. Sharding

Splitting one dataset across multiple databases so it fits and so writes scale. **This is the step you delay as long as possible** — it's operationally expensive and hard to reverse.

### Strategies

| Strategy | Key | Pros | Cons |
|---|---|---|---|
| **Range** | `user_id 1–1M → shard 1` | Range queries work; simple | **Hot spots** — sequential IDs or timestamps all land on the newest shard |
| **Hash** | `hash(user_id) % N` | Even distribution | Range queries need scatter-gather; resharding is painful |
| **Consistent hash** | ring | Even + cheap rebalancing | More moving parts |
| **Directory / lookup** | explicit shard map service | Total flexibility; move individual tenants | The lookup service is a SPOF and an extra hop |
| **Geographic** | region | Data residency (GDPR), low latency | Uneven population; cross-region queries are slow |

### Choosing the shard key — the actual decision

The shard key determines everything. Get it wrong and you either get hot spots or every query becomes a scatter-gather.

**Rules:**

1. **High cardinality** — enough distinct values to spread across shards.
2. **Even distribution** — no value dominates.
3. **Matches your access pattern** — the key should be in the `WHERE` clause of your most common query, so it hits one shard.
4. **Avoid monotonic keys** (timestamps, auto-increment IDs) — all writes go to one shard. Fix by prefixing with a hash or a random bucket.

**Worked example:** for a chat app, sharding messages by `message_id` spreads writes evenly but makes "load this conversation" hit every shard. Sharding by `conversation_id` makes the common read hit one shard, at the cost of a hot shard for an extremely busy group. **`conversation_id` is right** — optimise for the dominant access pattern, then handle the outlier specially.

### Problems sharding creates (be ready for all four)

| Problem | Mitigation |
|---|---|
| **Cross-shard joins** | Denormalise, or fetch and join in the application layer. Accept it |
| **Cross-shard transactions** | Avoid by design (keep a transaction inside one shard); otherwise saga (§20) |
| **Hot spots / celebrity problem** | Sub-shard the hot key, add a cache in front, or dedicate a shard |
| **Resharding** | Consistent hashing, or over-provision logical shards (e.g. 1024 logical shards mapped onto 8 physical nodes — you move logical shards, never re-hash) |

> **The "logical shards" trick is worth volunteering:** create far more shards than machines up front and map many logical shards to each physical node. Growing means reassigning logical shards, which is a data move, not a re-hash. This is how Vitess, Citus and many production systems work.

### Celebrity / hot key problem

One key (a celebrity's follower list, a viral tweet, a flash-sale product) gets orders of magnitude more traffic than the rest.

Mitigations: **cache the hot key aggressively** (often enough on its own), **split the key** (`celebrity_123_bucket_0..9` and aggregate), give it a **dedicated shard**, or **switch strategy for that key** (fan-out on read for celebrities, fan-out on write for everyone else — the Twitter hybrid).

---

## 13. Replication

Copies of the same data on multiple nodes for **availability**, **read scaling**, and **durability**. Distinct from sharding: replication copies the *same* data, sharding splits *different* data. Real systems do both.

### Topologies

| Topology | How | Pros | Cons |
|---|---|---|---|
| **Single leader** | All writes to the leader, reads from followers | Simple; no write conflicts | Leader is a write bottleneck and a failover risk |
| **Multi-leader** | Multiple writable nodes, usually one per region | Low write latency per region; survives region loss | **Write conflicts** — needs resolution (LWW, CRDT, app logic) |
| **Leaderless** | Write to any node, quorum-based (Dynamo style) | Highly available; no failover step | Needs read repair, anti-entropy, quorum tuning |

### Synchronous vs asynchronous

| | **Sync** | **Async** | **Semi-sync** |
|---|---|---|---|
| Write acked when | All (or a quorum of) replicas confirm | Leader alone confirms | ≥1 replica confirms |
| Latency | High (slowest replica) | Low | Moderate |
| Data loss on leader failure | None | **Possible** — unreplicated writes are lost | Bounded |
| Used by | Financial systems | Most read-scaling setups | **Common default (MySQL semi-sync)** |

### Replication lag — the thing they'll actually ask about

Async replication means a follower can be behind by milliseconds to minutes. Three classic anomalies and their fixes:

| Anomaly | Symptom | Fix |
|---|---|---|
| **Read-your-writes violated** | User posts a comment, refreshes, it's gone | Route that user's reads to the **leader** for N seconds after a write, or pin by session |
| **Monotonic reads violated** | Refresh and data goes *backwards* (two replicas at different lags) | **Sticky routing** — same user always hits the same replica |
| **Causal violation** | You see a reply before the message it replies to | Causal consistency / version vectors, or write related data to one partition |

> **A concrete answer that lands well:** *"After a write I'd set a short-lived cookie or session flag and route that user's reads to the primary for a few seconds. It's cheap and it fixes the only lag anomaly users actually notice."*

### Failover

Detect the leader is down → elect a new one → redirect writes. Hazards: **split brain** (fence the old leader), **data loss** (unreplicated async writes), **timeout tuning** (too short → spurious failovers under load), and **cascading failure** (new leader gets the full load cold, with an empty cache).

---

## 14. Caching

The single highest-leverage optimisation in read-heavy systems, and the one with the most follow-up questions.

### Where caches live

```
Browser cache  →  CDN  →  API gateway/reverse-proxy cache  →  Application in-memory
                                                           →  Distributed cache (Redis/Memcached)
                                                           →  Database buffer pool
```

Each layer is cheaper and faster than the next one down. "Cache higher up the stack" is almost always the right instinct.

### Caching patterns

| Pattern | Read | Write | Notes |
|---|---|---|---|
| **Cache-aside** (lazy loading) | App checks cache, on miss loads from DB and populates | App writes DB, then **invalidates** cache | **The default.** Resilient (cache down ≠ system down), but every miss costs a round trip and stale data is possible |
| **Read-through** | Cache itself loads from DB on miss | — | Cleaner app code; needs cache-provider support |
| **Write-through** | — | Write to cache **and** DB synchronously | Cache always fresh; writes are slower |
| **Write-behind** (write-back) | — | Write to cache, flush to DB asynchronously | Very fast writes; **risk of data loss** on cache failure |
| **Refresh-ahead** | Proactively refresh entries about to expire | — | Avoids miss latency for hot keys; wasteful for cold ones |

### Eviction policies

| Policy | Rule | Best for |
|---|---|---|
| **LRU** | Evict least recently used | General purpose — the default |
| **LFU** | Evict least frequently used | Stable popularity distributions |
| **FIFO** | Evict oldest inserted | Rarely optimal |
| **TTL** | Time-based expiry | **Combine with any of the above** — bounds staleness |
| **Random** | Evict at random | Surprisingly decent; O(1) and no bookkeeping |

Redis offers `allkeys-lru`, `volatile-lru`, `allkeys-lfu` etc. Being able to name the actual policy names is a nice touch.

### Invalidation — "one of the two hard problems"

| Strategy | Trade-off |
|---|---|
| **TTL only** | Simplest. Stale for up to the TTL. Usually good enough — say so |
| **Explicit invalidation on write** | Fresher, but you must find every affected key; races are possible |
| **Write-through** | Always fresh, slower writes |
| **Versioned keys** (`user:123:v7`) | No invalidation at all — bump the version, old entries age out naturally. **Elegant and worth mentioning** |
| **CDC-driven** | Database change stream invalidates the cache. Decoupled, eventually consistent |

### The four cache failure modes — know all four by name

| Problem | What happens | Fix |
|---|---|---|
| **Cache stampede / thundering herd** | A hot key expires; 10,000 concurrent requests all miss and hit the DB at once | **Request coalescing** (one loader, others wait), a short lock per key, probabilistic early expiry, or `stale-while-revalidate` |
| **Cache penetration** | Requests for keys that *don't exist* bypass the cache every time (often malicious) | **Cache the negative result** with a short TTL, or a **Bloom filter** of existing keys in front |
| **Cache avalanche** | Many keys expire simultaneously (e.g. all warmed at deploy time) | **Jitter the TTLs** (`ttl + random(0, 10%)`) — same reasoning as retry jitter |
| **Hot key** | One key gets so much traffic it saturates a single cache node | Replicate the key across nodes, add a local in-process cache in front, or split the key |

> **Cache stampede is the most-asked of the four.** The crisp answer: *"Single-flight — the first request to miss takes a per-key lock and loads; concurrent requests wait for that result rather than each hitting the database. Combined with serving stale data while revalidating, the origin sees exactly one request per key."*

### When NOT to cache

- Write-heavy data with low read reuse — you pay invalidation cost for nothing.
- Data that must be strictly consistent (balances, inventory counts).
- When the cache hit rate would be low — a 20% hit rate adds latency and complexity for very little.
- **Always ask what happens when the cache is empty** (cold start after deploy) or completely down. If the answer is "the database dies", the design is fragile: you need graceful degradation, request coalescing, and possibly a warm-up step.

---

## 15. CDN

Geographically distributed edge servers that cache content close to users.

- **Why:** a cross-continent round trip is ~150 ms vs ~10 ms to a nearby PoP. Also offloads bandwidth from your origin — often the bigger win financially.
- **Push vs pull:** *pull* (origin-fetch on first miss) is the default and self-managing; *push* is for large files you know will be needed (video releases) and avoids a slow first request.
- **Cache-control:** `max-age`, `s-maxage` (shared caches only), `stale-while-revalidate` (serve stale while fetching fresh — excellent for availability), `ETag`/`If-None-Match` for conditional requests.
- **Invalidation:** purging a CDN is slow and rate-limited. **Use versioned/fingerprinted URLs** (`app.a1b2c3.js`) so you never invalidate — you just reference a new URL. This is the standard answer.
- **Dynamic content:** modern CDNs also do edge compute, TLS termination, DDoS absorption and request routing — not just static assets.

---

# Part 4 — Data

## 16. SQL vs NoSQL

**The framing that scores:** the question isn't "which is better", it's *"what access patterns do I have, and what am I willing to give up?"* Start relational and justify deviating — that ordering signals judgement.

| | **Relational** | **NoSQL** |
|---|---|---|
| Schema | Fixed, enforced | Flexible / schema-on-read |
| Joins | First class | Usually absent — denormalise instead |
| Transactions | Full ACID across tables | Often single-item/partition only |
| Scaling | Vertical + read replicas; sharding is bolted on | Horizontal by design |
| Query flexibility | Ad-hoc SQL, any dimension | **You must know your access pattern up front** |
| Consistency | Strong by default | Usually tunable / eventual |

### The NoSQL families

| Type | Model | Good at | Examples |
|---|---|---|---|
| **Key-value** | `key → opaque blob` | Sessions, caches, feature flags. Fastest possible lookups | Redis, DynamoDB, Memcached |
| **Document** | `key → JSON` | Nested objects, varied schemas, content | MongoDB, Couchbase, Firestore |
| **Wide-column** | `row key → sparse columns`, sorted | Massive write throughput, time series, feeds | Cassandra, HBase, ScyllaDB, Bigtable |
| **Graph** | nodes + edges | Multi-hop relationship traversal (friends-of-friends, fraud rings) | Neo4j, Neptune |
| **Time-series** | timestamp-optimised, compressed | Metrics, IoT, monitoring | InfluxDB, TimescaleDB, Prometheus |
| **Search** | inverted index | Full-text, relevance ranking, faceting | Elasticsearch, OpenSearch, Solr |

### Choosing — a decision path you can say out loud

```
Need ad-hoc queries / joins / multi-row transactions?      → Relational (Postgres)
Simple key lookups at enormous scale, known access pattern? → Key-value (DynamoDB/Redis)
Write-heavy, time-ordered, need linear write scaling?       → Wide-column (Cassandra)
Nested documents, evolving schema, few joins?               → Document (MongoDB)
Relationship traversal is the core query?                   → Graph
Full-text relevance search?                                 → Search index (alongside your DB, not instead)
```

> **Polyglot persistence** is often the honest answer: Postgres for orders, Redis for sessions, Elasticsearch for search, S3 for media, Cassandra for the event log. Say it — but also note the operational cost of running five datastores, because that cost is real and mentioning it shows maturity.

**Modern nuance worth raising:** Postgres now does JSONB, full-text search and partitioning well, and often removes the need for a second store at moderate scale. "Just use Postgres until it hurts" is a defensible and increasingly common position.

---

## 17. Index Internals

Interviewers push here because it separates people who've *used* a database from people who understand one.

### B+ tree (Postgres, MySQL InnoDB, most OLTP)

- Balanced tree, all data in the leaves, leaves linked in a sorted list.
- Height ~3–4 for millions of rows → **O(log n)**, only a few disk seeks.
- **Great at:** point lookups, range scans (`BETWEEN`, `ORDER BY`, prefix `LIKE 'abc%'`), and reads generally.
- **Cost:** random writes cause page splits and write amplification; the index must be updated in place.

### LSM tree + SSTables (Cassandra, RocksDB, LevelDB, HBase)

- Writes go to an in-memory **memtable** + a write-ahead log → flushed as immutable sorted files (**SSTables**) → merged by background **compaction**.
- **Great at:** very high write throughput (all writes are sequential), and compression.
- **Cost:** reads may check multiple SSTables (mitigated by **Bloom filters** per file), and compaction consumes I/O in the background — causing latency spikes.

| | **B+ tree** | **LSM tree** |
|---|---|---|
| Write path | Random, in place | Sequential, append-only |
| Read amplification | Low | Higher (several files) |
| Write amplification | Moderate | Higher (compaction), but sequential |
| Space | Fragmentation from splits | Better compression; temporary duplicate space |
| Choose for | Read-heavy, OLTP, range queries | Write-heavy, time series, logs |

> **The one-liner:** *"B+ trees optimise for reads by keeping data sorted in place; LSM trees optimise for writes by never doing random I/O and paying for it later in compaction."*

### Index design rules

1. **Composite index column order matters** — an index on `(a, b, c)` serves `WHERE a=?`, `WHERE a=? AND b=?`, and `(a,b,c)`, but **not** `WHERE b=?` alone. It's a phone book sorted by last name then first name.
2. **Covering index:** if the index contains every column the query needs, the DB never touches the table (an "index-only scan"). Huge win.
3. **Selectivity:** indexing a boolean column is usually pointless — the planner will just scan. High-cardinality columns benefit most.
4. **Every index slows writes** and consumes space. Index for the queries you actually run.
5. **Read the query plan.** `EXPLAIN ANALYZE`. Saying "I'd check the plan rather than guess" is a strong, practical signal.
6. **Clustered vs secondary:** InnoDB stores rows *inside* the primary-key B+ tree (clustered), so secondary indexes store the PK and require a second lookup. This is why a random UUID primary key hurts InnoDB — it causes page splits everywhere. Use an ordered ID (§32) or a surrogate auto-increment key.

---

## 18. Normalisation vs Denormalisation

| | **Normalised** | **Denormalised** |
|---|---|---|
| Data duplication | None | Deliberate |
| Writes | One place to update — consistent | Multiple places — must keep in sync |
| Reads | Requires joins | Single fetch, fast |
| Fits | OLTP, write-heavy, strong consistency | Read-heavy, analytics, NoSQL, distributed |

**The rule:** normalise until it's slow, then denormalise deliberately with a plan for keeping copies in sync (events, CDC, or a nightly job). In a sharded or NoSQL world you often denormalise *first*, because joins across shards don't exist.

**Materialised views** are the structured middle ground: keep the normalised source of truth, maintain a precomputed read model, refresh on write or on a schedule. This is CQRS in miniature and it's exactly what a precomputed news feed is.

---

## 19. ACID vs BASE

**ACID** — Atomicity (all or nothing), Consistency (invariants preserved), Isolation (concurrent transactions don't interfere), Durability (committed survives crashes).

**BASE** — **B**asically **A**vailable, **S**oft state, **E**ventually consistent. Not a rigorous definition; it's the pragmatic opposite pole.

### Isolation levels — the table to have memorised

| Level | Dirty read | Non-repeatable read | Phantom |
|---|---|---|---|
| READ UNCOMMITTED | ✅ possible | ✅ | ✅ |
| **READ COMMITTED** | ❌ | ✅ | ✅ | ← *Postgres/Oracle default* |
| **REPEATABLE READ** | ❌ | ❌ | ✅ (❌ in InnoDB) | ← *MySQL default* |
| SERIALIZABLE | ❌ | ❌ | ❌ |

**Key point that gets missed:** READ COMMITTED does **not** prevent lost updates. Two transactions read a value, both write, one is silently overwritten. You need `SELECT … FOR UPDATE`, an optimistic version check, or SERIALIZABLE.

**MVCC** (multi-version concurrency control) is how Postgres/InnoDB deliver isolation without read locks: each transaction sees a snapshot; writers create new row versions instead of blocking readers. Trade-off: version bloat and vacuum/purge overhead. *"Readers don't block writers and writers don't block readers"* is the sentence to have ready.

---

## 20. Distributed Transactions

Once data spans services or shards, ACID across the boundary is no longer free.

### Two-phase commit (2PC)

A coordinator asks all participants to **prepare**; if all vote yes, it tells them to **commit**.

**Why it's rarely the answer:** it's **blocking** — if the coordinator dies after the prepare phase, participants hold locks indefinitely and cannot decide unilaterally. It also couples availability (any participant down = no commit) and scales badly. Know it, name its flaw, then propose sagas.

### Saga pattern ✅

Break the distributed transaction into a sequence of **local** transactions, each with a **compensating action** to undo it.

```
Order Saga:
  1. Create order          ⟲ compensate: cancel order
  2. Reserve inventory     ⟲ compensate: release inventory
  3. Charge payment        ⟲ compensate: refund payment
  4. Schedule shipment     ⟲ compensate: cancel shipment

Step 3 fails → run compensations for 2 and 1, in reverse order.
```

| | **Choreography** | **Orchestration** |
|---|---|---|
| Control | Each service reacts to events | A central orchestrator drives the steps |
| Coupling | Loose | Orchestrator knows the whole flow |
| Visibility | Hard to trace the overall state | **Easy — one place to look** |
| Best for | 2–3 simple steps | 4+ steps, or complex compensation logic |

**Critical caveats to raise unprompted:**

- Sagas give **atomicity but not isolation** — intermediate states are visible to other transactions. Handle with semantic locks (an `PENDING` status), commutative updates, or by accepting it.
- **Compensations aren't perfect rollbacks.** You can't un-send an email. Design compensations that are semantically meaningful (send a cancellation notice), not literal undos.
- Every step must be **idempotent**, because retries are guaranteed.

### Related patterns

- **Outbox** (§8) — reliably publish events with the transaction.
- **CDC** (Debezium) — read the database WAL and emit change events. Zero application code, at-least-once.
- **Event sourcing** — store the events as the source of truth and derive state. Perfect audit trail; more complexity, and schema evolution is genuinely hard.
- **TCC (Try-Confirm-Cancel)** — a reservation-style variant of saga; the seat-booking hold is exactly this.

---

## 21. Blob Storage

**Never store large binaries in your database.** Bloats backups, kills the buffer pool, expensive.

**The standard flow — know it, it comes up in any design with images or video:**

1. Client asks your API for a **pre-signed upload URL**.
2. Client uploads **directly to S3/GCS**, bypassing your servers entirely.
3. S3 emits an event → a worker generates thumbnails/transcodes.
4. Reads are served from the **CDN** in front of the bucket.
5. Your database stores only the **key/URL and metadata**.

Why it matters: your API servers never handle file bytes, so they don't need bandwidth or memory for uploads. Add **storage classes/lifecycle rules** (hot → infrequent access → archive) for cost, and mention **multipart upload** for large files with resumability.

---

# Part 5 — Communication

## 22. Sync vs Async

**The most useful question in any design:** *"does the user need to wait for this?"*

| | **Synchronous** | **Asynchronous** |
|---|---|---|
| Caller | Blocks for the result | Fires and continues |
| Coupling | Temporal — both must be up | Decoupled via a broker |
| Failure | Propagates immediately | Absorbed by the queue, retried |
| Latency | Sum of all hops | Request path stays fast |
| Complexity | Low | Higher — ordering, duplicates, eventual consistency |

**Move work off the request path** whenever it's not needed for the response: sending emails, generating thumbnails, updating search indexes, analytics, fan-out to followers, webhooks. Return `202 Accepted` with a status URL and do the work in a worker.

**Keep it synchronous** when the caller needs the result to proceed (auth checks, payment authorisation, reading data to render a page) or when the operation must fail visibly and immediately.

---

## 23. Protocols

| Protocol | Model | Best for | Watch out for |
|---|---|---|---|
| **REST/HTTP** | Request/response, resource-oriented | Public APIs, CRUD, caching via HTTP semantics | Over/under-fetching; chatty for nested data |
| **gRPC** | RPC over HTTP/2, Protobuf | **Internal service-to-service** — fast, typed, streaming, small payloads | Not browser-native without a proxy; binary is harder to debug |
| **GraphQL** | Client-specified queries | Many clients with different data needs; kills over-fetching | N+1 resolvers, hard to cache at HTTP level, query-complexity attacks |
| **WebSocket** | Full-duplex, persistent | Chat, multiplayer, live collaboration | Stateful connections complicate load balancing and scaling |
| **SSE** | Server → client, one way, over plain HTTP | Notifications, live feeds, progress | One direction only; browser connection limits on HTTP/1.1 |
| **Long polling** | Client holds a request open | Fallback when WebSockets are blocked | Wasteful; high connection churn |
| **Webhooks** | Server → server callback | Third-party event delivery | Must be idempotent, signed, and retried with backoff |

### HTTP versions

| | HTTP/1.1 | HTTP/2 | HTTP/3 |
|---|---|---|---|
| Transport | TCP | TCP | **QUIC over UDP** |
| Multiplexing | No (head-of-line blocking; hence domain sharding) | Yes, one TCP connection | Yes, **no TCP head-of-line blocking** |
| Headers | Plain text | Compressed (HPACK) | Compressed (QPACK) |
| Notable | Keep-alive, pipelining (broken in practice) | Server push (largely deprecated) | 0-RTT resumption, survives network switching |

**The realtime decision path** — a common follow-up:

```
Server → client only, updates are frequent?      → SSE (simplest, plain HTTP, auto-reconnect)
Bidirectional, low latency (chat, games)?        → WebSocket
Occasional updates, simplicity matters?          → Polling with a sensible interval
Mobile with unreliable networks?                 → Push notifications (APNs/FCM) + fetch on open
```

---

## 24. Queues vs Streams

**The distinction interviewers probe:** a **queue** deletes a message once it's consumed; a **log/stream** retains it and each consumer tracks its own offset. That single difference drives everything else.

| | **Message queue** (SQS, RabbitMQ) | **Event log/stream** (Kafka, Pulsar, Kinesis) |
|---|---|---|
| After consumption | Message is removed | **Retained** for a configured period |
| Consumers | Compete for messages | **Independent** — each has its own offset |
| Replay | No | **Yes — rewind the offset** |
| Ordering | Usually best-effort (FIFO queues available) | Strict **per partition** |
| Throughput | High | Very high (sequential disk I/O) |
| Use for | Task distribution, work queues, RPC decoupling | Event sourcing, multiple subscribers, analytics, CDC |

### Kafka concepts you'll be asked about

- **Topic → partitions.** A partition is the unit of parallelism *and* of ordering. **Ordering is guaranteed only within a partition**, so if you need per-user ordering, use `user_id` as the partition key.
- **Consumer group:** each partition is consumed by exactly one consumer in the group. **Consumers > partitions means idle consumers** — the partition count caps your parallelism, so choose it generously up front (increasing it later breaks key→partition mapping).
- **Offsets:** committed by the consumer. Commit *after* processing → at-least-once. Commit *before* → at-most-once.
- **Retention** by time or size; **compaction** keeps only the latest value per key (useful for changelog topics).
- **Replication factor** + `min.insync.replicas` + `acks=all` = durable writes. `acks=1` is fast and can lose data on leader failure.

### Delivery patterns

- **Point-to-point:** one message → one consumer. Task queues.
- **Pub/sub:** one message → all subscribers. Fan-out.
- **Competing consumers:** many workers on one queue for horizontal throughput.
- **Dead-letter queue (DLQ):** after N failed attempts, move the message aside so it stops blocking the queue, and alert. **Always mention the DLQ** — a poison message that retries forever is a classic outage, and most candidates forget it.

### Backpressure

When producers outrun consumers: the queue grows, latency grows, memory or disk fills. Responses, in increasing severity: **autoscale consumers**, **bound the queue and reject** (`429`), **shed load** (drop low-priority work), or **rate-limit the producer**. An unbounded queue is not a solution — it just converts a throughput problem into an out-of-memory problem later.

---

## 25. API Design

### Principles

- **Resource-oriented, plural nouns:** `/users/123/orders`, not `/getUserOrders`.
- **Correct verbs and semantics:** `GET` (safe, cacheable), `POST` (create, not idempotent), `PUT` (full replace, **idempotent**), `PATCH` (partial), `DELETE` (idempotent).
- **Status codes that mean something:** `200`, `201 Created`, `202 Accepted` (async), `204 No Content`, `400`, `401` (who are you), `403` (I know who you are, no), `404`, `409 Conflict`, `422`, `429 Too Many Requests`, `500`, `503`.
- **Idempotency keys on unsafe operations** — `Idempotency-Key` header on `POST`, so a retry doesn't double-charge.

### Pagination — always asked

| | **Offset** (`?page=3&size=20`) | **Cursor / keyset** (`?after=xyz&limit=20`) |
|---|---|---|
| Query | `LIMIT 20 OFFSET 40` | `WHERE (created_at, id) < (?, ?) LIMIT 20` |
| Deep pages | **O(n)** — the DB scans and discards every skipped row | **O(log n)** — index seek, constant cost |
| Stability | **Broken** — an insert shifts everything, you see duplicates or skips | Stable |
| Jump to page N | ✅ | ❌ |

> **The answer:** *"Cursor-based for infinite scroll and any large dataset — offset pagination degrades linearly and gives inconsistent results when data is being written. Offset is fine only for small, admin-style tables where users need to jump to a specific page."*

### Versioning

URL path (`/v1/users`) is the most explicit and easiest to route. Alternatives: `Accept` header (purer REST, harder to test) or a query param. Prefer **additive, backward-compatible changes** so you rarely need a new version. Have a documented deprecation window.

---

# Part 6 — Reliability

## 26. Rate Limiting

Protects against abuse, accidental overload, and cost blowouts. **Know all five algorithms and their trade-offs** — this is one of the most commonly asked topics, and it's also a standalone design question.

| Algorithm | How | Memory | Burst handling | Downside |
|---|---|---|---|---|
| **Fixed window** | Count per fixed interval; reset at the boundary | O(1) | Poor | **2× burst at the boundary** — 100 requests at 0:59 + 100 at 1:00 |
| **Sliding window log** | Store a timestamp per request, count those in the window | **O(n)** per user | Exact | Memory-heavy at scale |
| **Sliding window counter** | Weighted blend of current + previous window | O(1) | Good approximation | Slightly inexact |
| **Token bucket** | Tokens refill at rate `r`, capacity `b`; each request takes one | O(1) | **Allows controlled bursts** | — |
| **Leaky bucket** | Requests queue and drain at a constant rate | O(queue) | **Smooths output** — no bursts | Adds latency; queue can fill |

**Choosing:** **token bucket** is the usual answer — O(1) memory, allows legitimate bursts, simple to reason about. Use **leaky bucket** when the *downstream* needs a strictly smooth rate (e.g. calling a third-party API with a hard rate cap).

```java
// Token bucket — lazy refill, no background timer. This is the whole algorithm.
long now = clock.millis();
double refill = (now - lastRefillMs) / 1000.0 * refillRatePerSec;
tokens = Math.min(capacity, tokens + refill);
lastRefillMs = now;

if (tokens >= 1) { tokens -= 1; return ALLOW; }
return REJECT;   // respond 429 with a Retry-After header
```

### Distributed rate limiting

Per-instance limits don't work — 10 instances with a 100/s limit each is a 1000/s system limit. Options:

- **Centralised counter in Redis** — `INCR` + `EXPIRE`, or a Lua script for atomicity. Accurate; adds a network hop and a dependency.
- **Sticky routing** by user so one instance owns each user's counter. No shared state, but breaks under rebalancing.
- **Local limits with a share of the budget** — each instance gets `limit/N`. Approximate and unfair under uneven routing, but zero coordination.
- **Async reconciliation** — count locally, sync to Redis periodically. Fast, briefly over-permissive.

**Practical details worth mentioning:** return `429` with `Retry-After` and `X-RateLimit-Remaining` headers; rate limit by **user/API key** rather than IP where possible (NAT means many users share an IP); apply limits at the **edge/gateway** so bad traffic never reaches your services; and consider **tiered limits** per plan.

---

## 27. Resilience Patterns

### Timeouts

**Every network call needs a timeout.** A call with no timeout is a resource leak waiting for a bad day. Set it from the p99 of the dependency, not from a guess — and make sure your timeout budget shrinks as you go deeper in the call chain, or an inner retry will outlive an outer timeout.

### Retries — with backoff and jitter

Retry only **idempotent** operations and only **retriable** failures (5xx, timeouts, 429 — never 400 or 401). Use exponential backoff with **jitter** to avoid a synchronised retry storm.

**Add a retry budget:** cap retries at ~10% of the request rate globally (Google SRE's guidance). Without it, a partial outage becomes a full one as retries multiply load exactly when the system is weakest. *(Full treatment in the payment-retry notes.)*

### Circuit breaker

Stop calling a failing dependency so it can recover — and so your threads aren't all parked waiting on it.

```
CLOSED ──(failure rate > threshold)──► OPEN ──(after cooldown)──► HALF_OPEN
   ▲                                                                  │
   └──────────────(probe succeeds)────────────────────────────────────┘
                                    (probe fails → back to OPEN)
```

**Subtlety worth stating:** a *business* failure (card declined, 404) is **not** a health signal. Only count infrastructure failures — timeouts, 5xx, connection errors — or a spike in legitimate declines will trip your breaker.

### Bulkhead

Isolate resources so one slow dependency can't consume the whole thread pool. Separate connection pools/thread pools per downstream. Named after ship compartments: one flooded compartment doesn't sink the vessel.

### Load shedding & graceful degradation

When overloaded, **reject early and cheaply** rather than degrading everything. Prioritise: drop analytics before checkout. Return `503` with `Retry-After`.

**Graceful degradation** examples worth having ready: recommendations service down → show a static popular list; personalisation down → show generic content; search down → show browse. **Design each feature so it can fail independently** — that sentence alone is a strong answer.

### Idempotency + retries together

Retries are only safe if the operation is idempotent. This is why §8 and §27 are inseparable: **timeouts force retries, retries force idempotency.**

---

## 28. Observability

**Three pillars** (plus a fourth worth mentioning):

| Pillar | Answers | Tools |
|---|---|---|
| **Metrics** | "Is something wrong?" — aggregated, cheap, alertable | Prometheus, Datadog, CloudWatch |
| **Logs** | "What exactly happened?" — high detail, expensive at volume | ELK, Loki, Splunk |
| **Traces** | "Where did the time go?" — one request across many services | Jaeger, Zipkin, OpenTelemetry |
| *Profiles* | "Which code is burning CPU/memory?" | continuous profilers |

### What to measure

- **RED** (for services): **R**ate, **E**rrors, **D**uration.
- **USE** (for resources): **U**tilisation, **S**aturation, **E**rrors.
- **The four golden signals** (Google SRE): latency, traffic, errors, saturation.

### Practical points that score

- **Correlation/trace IDs** propagated through every hop — without them, debugging a distributed system is guesswork.
- **Alert on symptoms, not causes.** Alert on "checkout error rate > 1%", not "CPU > 80%". CPU at 80% might be fine; a failing checkout never is.
- **Structured logging** (JSON), never string concatenation, so logs are queryable.
- **Sample traces** (1–10%) — tracing every request is expensive and rarely more informative. Always sample errors.
- **Alert fatigue is a real failure mode.** Every alert must be actionable and have a runbook, or people learn to ignore the pager.

---

## 29. Deployment

| Strategy | How | Trade-off |
|---|---|---|
| **Rolling** | Replace instances a few at a time | No extra capacity needed; both versions live simultaneously (**schema must be compatible**) |
| **Blue-green** | Two full environments, flip traffic | Instant rollback; **2× infrastructure cost** |
| **Canary** | 1% → 5% → 25% → 100%, watching metrics | **Safest.** Catches problems with limited blast radius; slower |
| **Feature flags** | Deploy dark, enable at runtime | Decouples deploy from release; flag debt accumulates if not cleaned up |

**Schema migrations — the expand/contract pattern**, which interviewers love:

```
1. EXPAND:  add the new column (nullable), deploy code that writes BOTH
2. BACKFILL: migrate existing rows in batches
3. MIGRATE: deploy code that reads the new column
4. CONTRACT: stop writing the old column, then drop it
```

Never rename or drop a column in the same deploy as the code change — during a rolling deploy, old and new code run simultaneously, and one of them will break.

---

## 30. Disaster Recovery

| Term | Meaning | Question it answers |
|---|---|---|
| **RPO** (Recovery Point Objective) | Acceptable **data loss** window | "How much data can we lose?" |
| **RTO** (Recovery Time Objective) | Acceptable **downtime** | "How fast must we be back?" |

| Strategy | RTO | RPO | Cost |
|---|---|---|---|
| Backup & restore | Hours | Hours | $ |
| Pilot light (minimal standby) | ~10s of minutes | Minutes | $$ |
| Warm standby (scaled-down live) | Minutes | Seconds | $$$ |
| **Active-active multi-region** | ~0 | ~0 | $$$$ |

**Multi-region trade-offs:**

- **Active-passive:** simpler, one write region, failover is a deliberate operation. Cheaper, non-zero RTO.
- **Active-active:** lowest latency globally and survives a region loss, but you now have **multi-leader writes** — conflict resolution, and cross-region replication lag become your problem.
- **Data residency** (GDPR, India's DPDP) may force regional partitioning regardless of engineering preference. Mentioning compliance as a design driver is a nice senior touch.

**Test your backups.** An untested backup is a hypothesis, not a recovery plan. Regular restore drills and game days.

---

# Part 7 — Specialised Topics

## 31. Security

### AuthN vs AuthZ

**Authentication** = who are you. **Authorisation** = what may you do. `401` vs `403`.

| Mechanism | How | Trade-off |
|---|---|---|
| **Session cookie** | Server stores session, client holds an ID | **Revocable instantly**; needs shared session storage |
| **JWT** | Signed, self-contained token | Stateless, scales well; **hard to revoke before expiry** |
| **OAuth 2.0** | Delegated authorisation to a third party | Standard for "log in with X"; multiple flows to know |
| **OIDC** | Identity layer on top of OAuth 2.0 | OAuth is authorisation; OIDC adds authentication |
| **API keys / mTLS** | Static secret / mutual certs | Service-to-service |

**The JWT revocation problem** — a guaranteed follow-up. A JWT is valid until it expires; you can't un-issue it. Mitigations: **short-lived access tokens (5–15 min) + a long-lived refresh token** that *is* checked against a revocable store, plus a denylist of revoked JTIs for emergencies. Say the short-lived-access-token answer first.

### Other essentials

- **TLS everywhere**, including internal traffic (zero trust). TLS 1.3 has a 1-RTT handshake, 0-RTT on resumption.
- **Encryption at rest** for databases and blobs; manage keys in a KMS, rotate them.
- **Never store plaintext passwords.** Use bcrypt/scrypt/Argon2 with a per-user salt — deliberately slow hashes, not SHA-256.
- **Injection defences:** parameterised queries always; output encoding for XSS; CSRF tokens or `SameSite` cookies.
- **Secrets** in a vault (AWS Secrets Manager, HashiCorp Vault), never in code or environment files in the repo.
- **Defence in depth:** WAF, DDoS protection, rate limiting, least-privilege IAM, network segmentation, audit logs.
- **PII:** encrypt, minimise collection, define retention, support deletion (GDPR right to erasure) — which is genuinely hard in an event-sourced or heavily backed-up system. Worth flagging as a real design constraint.

---

## 32. Unique ID Generation

A favourite because it's small enough to design fully in 10 minutes.

**Requirements to state:** globally unique, ideally **time-sortable** (so it works as a database primary key and for pagination), high throughput, no single point of failure.

| Approach | Sortable | Coordination | Notes |
|---|---|---|---|
| **DB auto-increment** | ✅ | Single DB | Simple; SPOF and a write bottleneck |
| **UUIDv4** (random) | ❌ | None | 128 bits, no coordination — but **random order destroys B+ tree locality** (page splits, poor insert performance) |
| **UUIDv7** | ✅ | None | Time-ordered UUID — **the modern default**; keeps UUID's zero-coordination property *and* index locality |
| **ULID** | ✅ | None | 48-bit timestamp + 80-bit randomness, lexicographically sortable, base32 |
| **Snowflake** | ✅ | Needs unique worker IDs | The classic answer — see below |
| **Ticket server / range allocation** | ✅ | Central, but batched | Each node claims a block of 10,000 IDs; cheap and simple |

### Snowflake (64 bits)

```
┌─┬───────────────────────────────┬────────────┬──────────────┐
│0│  timestamp (41 bits, ms)      │ node (10)  │ sequence (12)│
└─┴───────────────────────────────┴────────────┴──────────────┘
 sign         ~69 years from epoch   1024 nodes   4096 per ms per node
```

- Fits in a `long`. **Time-sortable**, so it doubles as a good clustered primary key.
- 4096 IDs per millisecond per node = ~4M/s per node.
- **The two questions they'll ask:** *(1) how do nodes get unique IDs?* — ZooKeeper/etcd assignment, or derive from the pod/instance ID. *(2) what about clock skew / NTP moving the clock backwards?* — refuse to issue IDs until the clock catches up (a brief stall), and alert; never issue a duplicate.

> **The pragmatic modern answer:** *"UUIDv7 unless I need the compactness of a 64-bit key or the strict per-node ordering of Snowflake. It gives me zero coordination and time-ordering, which is what made UUIDv4 painful as a primary key."*

---

## 33. Geospatial Indexing

For "design Uber / Yelp / find nearby": *"give me everything within 5 km."* A plain B-tree on `(lat, lng)` can't do this efficiently — it's a 2-D range problem and a 1-D index only helps on one axis.

| Technique | Idea | Trade-off |
|---|---|---|
| **Geohash** | Interleave lat/lng bits into a base32 string; shared prefix = nearby | Simple, works with any string index (Redis, Postgres). **Boundary problem:** two adjacent points can have very different hashes, so you must query the cell **plus its 8 neighbours** |
| **Quadtree** | Recursively split space into 4; split further where density is high | Adapts to density (dense cities, sparse countryside); in-memory, needs rebuilding |
| **S2 (Google)** | Project the sphere onto a cube, Hilbert curve ordering | Excellent locality, handles spherical geometry correctly. Used by Uber and Google Maps |
| **H3 (Uber)** | Hexagonal grid | Uniform neighbour distances (hexagons have 6 equidistant neighbours; squares have 4+4 at different distances) |
| **PostGIS / R-tree** | Native spatial index in the DB | Just use it if you're already on Postgres — least effort by far |

**The standard interview answer:** *"Geohash with prefix matching, querying the target cell and its eight neighbours to handle boundaries. Precision level chosen from the search radius. At Uber's scale I'd move to S2 or H3 for better spherical locality, and keep driver locations in Redis with a short TTL since they're updated every few seconds and don't need durability."*

---

## 34. Probabilistic Data Structures

Trade exactness for enormous space savings. Interviewers love these because they show you know that "approximately right, in 1 KB" often beats "exactly right, in 10 GB".

| Structure | Answers | Error | Space |
|---|---|---|---|
| **Bloom filter** | "Have I seen X?" | **False positives possible, false negatives impossible** | ~10 bits per element for 1% FPR |
| **Counting Bloom / Cuckoo filter** | Same, but supports deletion | Same | Slightly more |
| **HyperLogLog** | "How many *distinct* items?" | ~2% standard error | **~12 KB for billions of items** |
| **Count-Min Sketch** | "How often did X appear?" | Overestimates only | Sublinear |
| **Top-K / Space-Saving** | "What are the heaviest hitters?" | Approximate | Sublinear |

**Where they show up in designs:**

- **Bloom filter** in front of a cache or database to avoid lookups for keys that don't exist (**cache penetration**, §14); inside every LSM-tree SSTable to skip files during reads (§17); in a crawler to check "already visited".
- **HyperLogLog** for unique-visitor counts — Redis has `PFADD`/`PFCOUNT` built in. Counting 1 B unique users exactly needs gigabytes; HLL needs 12 KB with 2% error, and nobody cares about 2% on a dashboard.
- **Count-Min Sketch** for detecting hot keys and heavy hitters in a stream.

> **The framing:** *"A Bloom filter can tell you 'definitely not present' or 'probably present'. That asymmetry is exactly what you want in front of an expensive lookup — a false positive costs one wasted query, and false negatives can't happen, so you never miss real data."*

---

## 35. Search

### Inverted index — the core idea

Instead of `document → words`, store `word → list of documents`:

```
"database"  → [doc1, doc7, doc12, ...]
"system"    → [doc3, doc7, doc19, ...]
Query "database system" → intersect the two posting lists
```

**Pipeline:** tokenise → lowercase → remove stop words → stem/lemmatise ("running" → "run") → build postings. At query time, apply the same analysis to the query, intersect posting lists, then **rank**.

**Ranking:** TF-IDF (term frequency × inverse document frequency) or **BM25** (the modern default — TF-IDF with saturation and length normalisation). Increasingly combined with vector/embedding search for semantic matching, then re-ranked — worth a sentence if you want to sound current.

### In a design

- Elasticsearch/OpenSearch is a **secondary** store, not your source of truth. Keep the authoritative data in your primary database and sync via CDC or events — say this explicitly, because treating a search index as a database is a common mistake.
- The sync is **eventually consistent**; a document may be searchable a second or two after it's written. Usually fine — confirm it with the interviewer.
- **Autocomplete** is a different problem: use a **trie** (or an FST/completion suggester), not full-text search, and precompute the top-N completions per prefix.
- Sharding + replicas: shards for index size and write throughput, replicas for query throughput and availability.

---

# Part 8 — The Interview Itself

## The reusable architecture skeleton

Almost every answer starts here, then you add what the requirements demand:

```
                         ┌─── CDN (static + media) ───┐
                         │                            ▼
  Client ──► DNS ──► Load Balancer ──► API Gateway ──► Service(s)
                                        (auth,           │
                                         rate limit)     ├──► Cache (Redis)
                                                         ├──► Primary DB ──► Read replicas
                                                         ├──► Blob store (S3)
                                                         └──► Message queue ──► Workers
                                                                                   │
                                                                                   ├──► Search index
                                                                                   └──► Analytics/warehouse
```

## Answering "how would you scale this?"

Walk the ladder, in order, and justify each step from the numbers:

1. **Cache** the hot reads (biggest win per unit of effort in read-heavy systems)
2. **Read replicas** for further read scaling
3. **CDN** for static and media
4. **Async** — move non-essential work off the request path
5. **Shard** the database (only when a single primary genuinely can't hold the writes)
6. **Denormalise / precompute** read models
7. **Regional deployment** for global latency

## The follow-ups you should expect

| Question | What they're testing | Your angle |
|---|---|---|
| "What if this component fails?" | Failure thinking | Redundancy, graceful degradation, blast radius |
| "What's your bottleneck?" | Whether you understand your own design | Name it *before* they ask — usually the DB or a hot key |
| "How would you handle 10× traffic?" | Scaling reasoning | The ladder above; what breaks first |
| "How do you keep the cache consistent?" | Depth on caching | TTL, invalidation strategy, versioned keys, and the accepted staleness window |
| "What if two users do X simultaneously?" | Concurrency | Optimistic locking, conditional update, unique constraint |
| "How do you monitor this?" | Operational maturity | RED/golden signals, alert on symptoms, trace IDs |
| "What would you do differently with more time?" | Self-awareness | Have two honest answers ready |
| "How do you deploy without downtime?" | Practical experience | Rolling/canary + expand-contract migrations |

## The trade-off sentences to have ready

- "Read-heavy, so I'd optimise reads with caching and replicas, and accept eventual consistency on non-critical fields."
- "This is a write-heavy, time-ordered workload, so an LSM-based store fits better than a B-tree one."
- "I'd precompute on write because reads outnumber writes 100:1 — but I'd fall back to compute-on-read for celebrity accounts."
- "Strong consistency for money, read-your-writes for anything the user just did, eventual everywhere else."
- "I'd start with a single Postgres instance. At this scale it's plenty, and sharding is a decision I'd rather defer than undo."
- "Exactly-once delivery isn't achievable; I'd do at-least-once plus an idempotent consumer."
- "That's a real risk. I'd bound it with a timeout and a circuit breaker, and degrade to a cached response."

## Numbers cheat sheet

```
1 day ≈ 10⁵ s          1 M/day ≈ 12 QPS         1 B/day ≈ 12,000 QPS
Peak ≈ 2–3× average    Row ≈ 1 KB               Read:write is the first thing to ask

Memory ~100× SSD ~100× HDD seek
Same-DC round trip ~0.5 ms      Cross-continent ~150 ms
99.9% = 43 min/month down       99.99% = 4.3 min/month

W + R > N  ⇒ quorum overlap ⇒ consistent reads
Concurrency = Throughput × Latency        (Little's Law)
Series availability multiplies; parallel redundancy adds nines
```

## Final checklist before you finish

- [ ] Did I state assumptions and get them confirmed?
- [ ] Did I do estimation and *use* the numbers to justify a decision?
- [ ] Did I name at least one trade-off for every major choice?
- [ ] Did I identify the bottleneck myself?
- [ ] Did I cover failure modes — what breaks, and what happens when it does?
- [ ] Did I mention monitoring?
- [ ] Did I say what I'd cut or defer, and why?

---

*End of fundamentals. Next: applying these to specific design questions.*
