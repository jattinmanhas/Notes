# HLD 01 — URL Shortener (bit.ly / TinyURL)

> **Why this question exists:** it's the "hello world" of system design, and that's exactly why interviewers use it. The happy path is trivial, so the whole interview is about *depth*. Can you derive the key length from maths, generate unique IDs without a bottleneck, and serve 100K+ redirects/sec at low latency?
>
> **The one hard problem:** **generating short, unique, non-guessable codes at scale without a coordination bottleneck.** Everything else (caching, sharding) is standard and comes straight from the fundamentals notes. If you only master one section here, master §5.
>
> **Fundamentals used:** estimation (§1), caching and its failure modes (§14), consistent hashing/sharding (§11–12), ID generation (§32), idempotency/unique constraints (§8), Bloom filters (§34), stream processing for analytics (§37).

---

## Table of Contents

1. [Requirements](#1-requirements)
2. [Estimation — and the decisions it forces](#2-estimation--and-the-decisions-it-forces)
3. [API design](#3-api-design)
4. [Data model and database choice](#4-data-model-and-database-choice)
5. [The core problem — generating short codes](#5-the-core-problem--generating-short-codes)
6. [High-level design](#6-high-level-design)
7. [Deep dives](#7-deep-dives)
8. [Failure modes and bottlenecks](#8-failure-modes-and-bottlenecks)
9. [How the design evolves with scale](#9-how-the-design-evolves-with-scale)
10. [Interviewer follow-ups — with answers](#10-interviewer-follow-ups--with-answers)
11. [The 45-minute script](#11-the-45-minute-script)
12. [Intuition takeaways](#12-intuition-takeaways)

---

## 1. Requirements

### Functional — what the system does

| Requirement | In scope? | Why it matters to the design |
|---|---|---|
| Given a long URL, return a short URL | ✅ Core | The write path |
| Visiting the short URL redirects to the long URL | ✅ Core | The read path, **99% of traffic** |
| Custom aliases (`bit.ly/jattin-resume`) | ✅ Usually | Creates a uniqueness problem separate from generated codes |
| Expiry (link dies after a date) | ✅ Usually | Needs a cleanup strategy |
| Click analytics (count, geo, referrer) | ⚠️ Ask | Changes the redirect status code (§3) and adds a pipeline. Good to offer as an extension |
| User accounts, edit/delete links | ⚠️ Mention, then cut | Standard CRUD, nothing interesting to design |

> **Say it out loud:** *"I'll focus on shorten + redirect + custom alias + expiry. I'll treat analytics as an extension and come back to it if we have time, because it changes one decision I'd rather make explicitly."*

### Non-functional — the ones that shape the design

| Property | Target | Consequence |
|---|---|---|
| **Read:write ratio** | ~100:1 | Optimise the read path first: cache, replicas, possibly CDN |
| **Redirect latency** | p99 < ~50 ms server-side | The redirect must be a single cache/KV lookup, with nothing else on the hot path |
| **Availability** | Redirect ≫ creation | A broken redirect breaks *every link ever shared*. Creation can degrade more gracefully |
| **Consistency** | A new link should work almost immediately; stale-for-seconds is fine elsewhere | Read-your-writes for the creator, eventual for everyone else |
| **Durability** | A created link must never be lost | Links are printed on posters and in PDFs. Losing one is permanent damage |
| **Uniqueness** | **Strict.** Two long URLs must never share a code | This is the correctness core. It is non-negotiable |
| **Unguessability** | Codes shouldn't be enumerable | Otherwise someone can crawl `aaaaaa1, aaaaaa2…` and scrape every private link |

> **The intuition:** this is a system where **reads are cheap and hot, writes are rare but must be exactly unique.** Almost every decision follows from that asymmetry.

---

## 2. Estimation — and the decisions it forces

**Assumptions (state them, get a nod, move on):**

```
New URLs:            100 M / day
Read:write:          100 : 1        →  10 B redirects / day
Retention:           10 years
Avg record size:     ~500 B         (code 7 B + long URL ~100–200 B + metadata + index/storage overhead)
```

**QPS:**

```
Writes:   100 M / 10⁵ s   ≈ 1,000 /s        peak ≈ 3,000 /s
Reads:    10 B  / 10⁵ s   ≈ 100,000 /s      peak ≈ 300,000 /s     ← the design driver
```

**Storage:**

```
Total URLs over 10 years:  100 M × 365 × 10   ≈ 365 B URLs
Raw storage:               365 B × 500 B      ≈ 180 TB
With replication ×3:                          ≈ 550 TB
```

**Key length, derived rather than guessed** (the most important calculation in this question):

```
Alphabet: base62 = [a–z A–Z 0–9]   (URL-safe, no special characters)

62⁶ ≈ 56.8 B     < 365 B needed    ✗  runs out in ~1.5 years
62⁷ ≈ 3.5 T      > 365 B needed    ✓  ~10× headroom
62⁸ ≈ 218 T                        overkill, and longer URLs
```

**→ 7 characters.** And now you can *defend* it: "6 runs out in about 18 months; 7 gives ~10× headroom over 10 years."

**Cache sizing.** Size the cache by the **working set** (the links people are actually clicking), not by total data:

```
Clicks concentrate on RECENT links: people click a link in the days after it's shared.
Links created in the last 30 days:     100 M × 30        = 3 B links
Assume ~20% of those are "hot":        0.2 × 3 B         = 600 M links
Memory:                                600 M × 500 B     ≈ 300 GB
→ ~5 Redis nodes at 64 GB, call it 8–10 with headroom and replicas
```

Compare that with caching *everything*: 180 TB. Caching the hot set covers most traffic at about 0.2% of the storage. **That ratio is why caching works at all.**

*(A common shortcut is "cache 20% of daily requests × 500 B = 1 TB". It over-counts, because the same hot link is requested millions of times but only needs to be cached once.)*

**Bandwidth:** `100K reads/s × ~500 B ≈ 50 MB/s`. That's trivial, so bandwidth isn't a concern here, and it's worth saying so. *Ruling things out is also a signal.*

### What the numbers decide

| Number | Decision it forces |
|---|---|
| 100K–300K reads/s | A cache in front of the DB is mandatory. You *could* serve this with ~10+ DB replicas, but a handful of Redis nodes does it cheaper and at sub-millisecond latency |
| ~1–3K writes/s | Writes are *not* the problem. A single well-provisioned DB could almost absorb them. **Don't over-engineer the write path** |
| ~550 TB with replication | Doesn't fit on one machine, so you need a horizontally scalable store or sharding |
| 365 B keys vs 62⁷ space | 7-char codes. It also tells you the **collision rate** for hash-based approaches (§5) |

---

## 3. API design

```
POST /api/v1/urls
  Body:     { "long_url": "https://...", "custom_alias": "optional", "expires_at": "optional" }
  Headers:  Authorization, Idempotency-Key (optional)
  → 201 Created   { "short_url": "https://sho.rt/aZ3kQ9x", "code": "aZ3kQ9x", "expires_at": ... }
  → 409 Conflict  if custom_alias is taken
  → 400           if long_url is invalid / blocked

GET /{code}
  → 301 or 302 with   Location: <long_url>
  → 404           if the code doesn't exist
  → 410 Gone      if expired  (more honest than 404, a nice touch)

DELETE /api/v1/urls/{code}   → 204   (owner only)
```

### 301 vs 302: the question that's always asked

| | **301 Moved Permanently** | **302 Found** (or 307) |
|---|---|---|
| Browser behaviour | **Caches the redirect.** Repeat visits never reach your servers | Asks your server every time |
| Server load | Lower | Higher |
| Analytics | **Lost** after the first visit per browser | Every click is counted |
| Can you change or expire the link later? | **Not reliably.** Browsers keep the cached redirect | Yes |

**Middle ground:** 301 with an explicit `Cache-Control: max-age=3600`. The browser re-checks after an hour, so you get most of the load savings and still keep control within an hour.

> **The answer:** *"If analytics or expiry matter, I'd use 302, because every click has to reach me. If it's purely about cost and links are permanent, 301 offloads repeat traffic to the browser. Since we said expiry is in scope, 302."*
>
> **The intuition:** the status code is a **caching decision**. 301 means "let the client cache this forever", and that's the same trade-off as any cache: less load, less control.

**Idempotency on POST:** if the client retries a create after a timeout, you don't want two codes for one intent. An `Idempotency-Key` header, claimed with a unique insert (fundamentals §8), solves it. This is a small but mature detail.

---

## 4. Data model and database choice

```
urls
  code         VARCHAR(7)   PRIMARY KEY      ← the only lookup that matters
  long_url     TEXT
  user_id      BIGINT       (nullable for anonymous)
  created_at   TIMESTAMP
  expires_at   TIMESTAMP    (nullable)

users            (standard; not interesting)
```

### Which database? Derive it from the access pattern

Ask "what queries do I run?":

1. `GET by code` → **point lookup by primary key.** This is ~99% of traffic.
2. `INSERT if code not exists` → **conditional write** (the uniqueness guarantee).
3. `List my links` → by `user_id`, rare, and can be a secondary index.

There are **no joins, no multi-row transactions, and no complex queries.** That profile is exactly what a key-value store is for.

| Option | Case for it | Case against |
|---|---|---|
| **DynamoDB / Cassandra** (KV / wide-column) | Built for PK lookups at huge scale; horizontal by design; built-in TTL for expiry; conditional writes (`attribute_not_exists` / `IF NOT EXISTS`) | Cassandra's `IF NOT EXISTS` uses Paxos (lightweight transactions), which is slower. That's fine at ~1K writes/s |
| **Postgres, sharded by code** | Familiar, strong constraints (`PRIMARY KEY` gives uniqueness for free), rich tooling | You operate the sharding yourself at 550 TB (Citus/Vitess help) |

> **A defensible answer:** *"The access pattern is a pure PK lookup with no joins, so a KV store like DynamoDB or Cassandra fits naturally and gives me horizontal scaling and TTL for free. Postgres would also work. Honestly, at 1K writes/s the store is not the hard part; the cache in front of it is what serves the traffic."*

### Should the same long URL always get the same code (dedup)?

Tempting, but usually **no**:

- Different users want **their own** link, with their own analytics, expiry and ability to delete.
- Dedup needs a second index `long_url → code`, and long URLs are big, so that's a large index for little saving.
- Storage is cheap. 365 B rows is fine.

Offer it only as a per-user option: "if *the same user* shortens the same URL, return the existing code." That needs an index on `(user_id, hash(long_url))`.

---

## 5. The core problem — generating short codes

This is where the interview is won or lost. There are four approaches. Understand *why* each one fails before you accept the one that works.

**The whole section as one chain of problem → fix.** This is also the order you should say it out loud in the interview:

```
Hash the URL            → problem: collisions (and they grow)         → fix: make collisions impossible
Global counter          → problem: one counter = bottleneck + SPOF    → fix: hand out RANGES, not single IDs
Counter ranges          → problem: sequential codes are guessable     → fix: SCRAMBLE with a reversible function
Scrambled counter       → unique by construction, unguessable, coordination only once per million IDs  ✓
```

Every step fixes the previous step's problem and keeps its benefits. If you can recite this chain, you can answer the question.

### Approach A — Hash the long URL

```
code = base62( SHA-256(long_url) mod 62⁷ )    →  7 chars
```

**Why it's attractive:** it's stateless, needs no coordination, and the same URL always gives the same code (free dedup).

**Why it breaks: collisions.** You're squeezing a hash into 7 characters (3.5 T slots) and filling 365 B of them.

```
Probability a NEW code collides with an existing one  ≈  used / space
  Year 1:   36 B / 3.5 T  ≈ 1%
  Year 10: 365 B / 3.5 T  ≈ 10%    ← one in ten writes collides
```

So you *must* handle collisions: attempt a **conditional insert**, and on conflict re-hash with a salt (`hash(url + counter)`) and retry. It works, but:

- Every write can need multiple DB round trips, and it gets *worse over time*.
- If you dedup by URL, two users shortening the same URL collide *intentionally*, so you need to tell "same URL" apart from "true collision".

> **The intuition (birthday paradox):** collisions arrive far earlier than people expect. With N slots, you expect your first collision after only ~√N insertions. √3.5 T ≈ 1.9 M, and you create 1.9 M links in about 30 minutes. Concretely, expected collisions among n items ≈ n² / 2N, so on **day one** (n = 100 M) that's `10¹⁶ / 7×10¹² ≈ 1,400 collisions`. Never assume a truncated hash is unique.
>
> **Don't confuse the two numbers.** "~1% of writes collide in year 1" is the chance that *one given write* collides. "First collision within 30 minutes" is the chance that *any pair* collides. Both are true. The second is why a collision check is mandatory from day one, and the first is why the retries stay cheap early on and get more expensive over time.

### Approach B — Random code + collision check

```
code = 7 random base62 characters
INSERT … IF NOT EXISTS   → retry on conflict
```

It has the same collision maths as hashing but no dedup benefit. It's simple and perfectly fine **at small scale**. Say it's your starting point and explain when it stops being good enough.

### Approach C — Global counter + base62 encoding

```
id   = next value of a counter      (1, 2, 3, … 365 B)
code = base62(id)
```

**Why it's attractive:** **unique by construction.** There are zero collisions and no check is needed. Each ID is used exactly once.

**Three problems, and each has a standard fix:**

| Problem | Fix |
|---|---|
| **A single counter is a bottleneck and a SPOF.** Every write has to hit it | **Range allocation:** each app server grabs a *block* of 1 M IDs at once from a ticket server (a single DB row: `UPDATE counter SET next = next + 1000000 RETURNING next`). It then issues IDs from memory. The ticket server is hit about once per block per server, so at ~1K writes/s cluster-wide that's roughly every 15 minutes. It's no longer a bottleneck, and a short outage doesn't matter because servers have buffered ranges |
| **Crashes waste ranges.** A server dies with 600K unused IDs | **Don't care.** 3.5 T space vs 365 B needed. Wasting IDs is free. *Say this explicitly; it shows you did the maths* |
| **Sequential codes are guessable.** `aZ3kQ9x` → `aZ3kQ9y` lets anyone enumerate every link | **Scramble the ID with a bijection** before encoding (below) |

**Scrambling: unique *and* unguessable.** The key insight in plain words:

> **Any reversible function is automatically collision-free.** If two different inputs produced the same output, you couldn't reverse it, because you wouldn't know which input to return. So if you pass unique counter values through a function you *could* undo, the outputs are guaranteed unique. No check needed.

That's why "encrypt the counter" works and "hash the URL" doesn't: encryption is reversible and a hash isn't. Two concrete choices:

- **Simple:** `scrambled = (id × P) mod 62⁷`, where P is a large number sharing no factor with 62⁷ (i.e. not divisible by 2 or 31). It's reversible (multiply by P's modular inverse), so there are no collisions. It hides the sequence from casual users, but someone who collects a few codes can work out P.
- **Proper:** a small **Feistel cipher** (format-preserving encryption) with a secret key. Think of it as "encrypt the number 1,000,042 into another number in the same range". It's a real permutation, and without the key the output looks random.

**Fixed length:** the scrambled value can be any number in `[0, 62⁷)`, including small ones, so **left-pad the base62 string to 7 characters** (e.g. with `0`). Every code is then exactly 7 characters.

### Approach D — Key Generation Service (KGS)

Pre-generate random unique codes **offline** and store them in a pool. The write path just takes one.

```
KGS (offline):   generate random 7-char codes → insert into `unused_keys` (unique)
App servers:     atomically take a batch of 1,000 keys (DELETE … RETURNING) into memory
                 → hand one to each create request
```

The `urls` table itself is the record of used codes, so a separate `used_keys` table isn't needed.

| Pros | Cons |
|---|---|
| No collisions on the write path, because uniqueness is resolved offline | An **extra service** with its own storage and availability concerns |
| Codes are random, so they're unguessable | Pool must be kept topped up; **concurrency on hand-out** (two servers must never get the same batch → atomic `DELETE … RETURNING` or `SELECT … FOR UPDATE SKIP LOCKED`) |
| Write path is a pure memory pop | A server crash loses its in-memory batch. That's fine for the same reason as C: the space is huge |

### Custom aliases: a separate uniqueness problem

The user picks `jattin-resume`. It must not collide with **generated** codes, now or in the future.

- **Conditional insert** with the unique constraint is the referee. Taken → `409 Conflict`.
- **Future collision risk:** your generator might later produce `jattin1`, which a user already took as an alias. Fixes: have the generator's conditional insert simply skip taken codes (with C and D that's a rare retry), **or** separate the namespaces (e.g. custom aliases must be ≥ 8 characters or contain a `-`, which generated codes never do). Namespace separation is the cleaner answer.
- Validate: a blocklist of reserved words (`api`, `admin`, `login`) and profanity.

### Choosing

| | Unique by construction | Unguessable | Coordination | Complexity |
|---|---|---|---|---|
| A. Hash | ❌ collisions grow over time | ✅ | None | Low, but retry logic |
| B. Random + check | ❌ same | ✅ | None | Lowest |
| **C. Counter ranges + scramble** | ✅ | ✅ (with Feistel) | Rare (once per range) | Medium |
| **D. KGS** | ✅ | ✅ | Batch hand-out | Medium-high (extra service) |

> **The answer to give:** *"I'd use range-based counters: each server leases a block of IDs from a ticket store, so the write path never coordinates. I'd run each ID through a keyed permutation before base62-encoding, so codes are unique by construction and not enumerable. Wasted ranges on crashes don't matter because we use about 10% of the 62⁷ space. I'd still keep a conditional insert as a safety net, because uniqueness is the one property I can't get wrong."*
>
**On the "safety net" conditional insert:** in DynamoDB a conditional write costs almost nothing, so keep it. In Cassandra, `IF NOT EXISTS` runs Paxos and costs about 4 round trips. When uniqueness is already guaranteed by construction, it's reasonable to skip it and rely on the design. Mention that you know the cost.

> **The intuition:** **"generate unique" is always cheaper than "generate random, then check".** If you can make collisions *impossible* instead of *detected*, do it. And coordination is only expensive when it's per-request, so batch it (ranges, KGS batches) and it disappears.

---

## 6. High-level design

```
                                   ┌──────────────── CDN / edge (optional, for 301s or short-TTL 302s)
                                   │
 Client ──► DNS ──► Load Balancer ─┼──► Redirect Service (stateless, many instances)
                                   │         │  1. GET code from Redis ──hit──► 302 Location
                                   │         │  2. miss → KV store (read replica) → populate cache → 302
                                   │         │  3. emit click event (async, fire-and-forget) ──► Kafka ──► analytics
                                   │         ▼
                                   │     ┌────────┐        ┌──────────────────────┐
                                   │     │ Redis  │ ◄────  │  KV store (sharded   │
                                   │     │ cache  │        │  by code, RF=3)      │
                                   │     └────────┘        └──────────────────────┘
                                   │                                ▲
                                   └──► Write Service (stateless) ──┘  conditional insert
                                             │
                                             └──► Ticket store (ID ranges) — hit once per 1M IDs
```

**Why separate Redirect and Write services?** Their profiles are opposite: 100× more reads than writes, very different latency needs, and different failure tolerance. Separating them lets you scale and deploy each independently, and a bug in link creation can't take down redirects. *This is the "design each feature to fail independently" idea from fundamentals §27.*

### Walk the write path

1. Client `POST`s a long URL. The gateway authenticates and **rate-limits** (abuse prevention).
2. Write service validates the URL: format, `http/https` only (no `javascript:` URLs), a max length (~2 KB), a blocklist, and optionally an async malware scan.
3. It takes the next ID from its **in-memory range** (refilling from the ticket store if the range is exhausted), scrambles it, and base62-encodes it.
4. **Conditional insert** into the KV store (`IF NOT EXISTS`), acknowledged by a quorum/majority, so it's durable.
5. **Write the new link to the cache with a short TTL** (say 1 hour), and delete any negative-cache entry for that code (see §7.1). The creator usually tests the link immediately, so that first click is a cache hit, and it also covers read-your-writes if replicas lag. The TTL is short because most links are clicked rarely or never, and you don't want 100 M cold links a day pushing hot ones out of the cache.
6. Return `201` with the short URL.

### Walk the read path (the one that matters)

1. `GET /aZ3kQ9x` → LB → any redirect instance.
2. **Redis lookup.** On a hit (the vast majority), return `302` immediately. Total server time is about 1–2 ms.
3. On a miss, read from the KV store (a replica is fine), populate the cache with a TTL, and return `302`.
4. Not found → `404` (and **cache the negative result** briefly, see §7).
5. Expired → `410`.
6. Fire a click event to Kafka **without waiting** for it. Analytics must never slow down or break a redirect.

---

## 7. Deep dives

### 7.1 Caching the redirect path

- **Pattern:** cache-aside with a TTL. Links are **effectively immutable** (they rarely change), so this is the easiest caching problem there is. There's almost no invalidation problem. Say so.
- **Eviction:** LRU (or LFU, since popularity is skewed and fairly stable for viral links). In Redis, `allkeys-lru` or `allkeys-lfu`.
- **Hot key (a viral link):** one code gets millions of hits per minute and saturates one Redis shard. Fixes: a small **in-process LRU cache** in each redirect instance (a few seconds' TTL), which absorbs almost everything, or replicate the hot key across nodes.
- **Stampede:** a viral link's cache entry expires and thousands of requests hit the DB at once. Fix: **single-flight** (one loader per key) and jittered TTLs.
- **Penetration (non-existent codes):** bots hammer random codes, every one misses the cache, and every one hits the DB. Fixes: **cache negative results** (`code → NOT_FOUND`, short TTL), and/or a **Bloom filter** of all existing codes checked before the DB. A "definitely not present" answer costs zero DB work.
  - **The subtle bug:** someone probes `jattin-cv`, gets a 404, and that 404 is negatively cached for 5 minutes. Then you create `jattin-cv` as a custom alias, and it returns 404 for up to 5 minutes. **Fix: the create path deletes the negative-cache entry** (§6, write path step 5). The Bloom filter has the same issue: **add the code on create**, and keep the filter shared (e.g. RedisBloom) or rebuilt periodically, so every redirect instance knows about new codes.
- **Edge caching:** with 301, or 302 plus a short `Cache-Control: max-age`, a CDN can serve redirects from the edge, so latency is near the user and origin load drops. This trades away analytics accuracy, as in §3.

> **The intuition:** immutable data is a caching dream. The only hard parts are the *extremes*: one key getting all the traffic (hot key/stampede) and keys that don't exist at all (penetration).

### 7.2 Sharding the store

- **Shard key = `code`.** Every hot query is `WHERE code = ?`, so every lookup hits exactly one shard. This is the textbook case of "the shard key matches the access pattern".
- **Hash vs range:** codes are already scrambled or random, so they distribute evenly under either. If you ever used **raw sequential** IDs, range sharding would put all new writes on the last shard, and *new links are also the hottest reads*. This is another reason the scramble matters.
- **`List my links` by user_id** doesn't match the shard key, so it's a scatter-gather. Fix: a secondary index (DynamoDB GSI), or a separate small `user_links (user_id, code)` table sharded by user. It's a rare query, so either is fine.
- **Resharding:** use consistent hashing or logical shards (fundamentals §11–12). Managed stores (DynamoDB) handle this for you, which is a legitimate reason to pick one.

### 7.3 Expiry and cleanup

Two separate jobs:

1. **Correctness (lazy):** on every read, check `expires_at`. If it has passed, return `410` regardless of whether the row still exists. Expiry is then correct to the second without any background job.
2. **Space reclamation (eager):** actually delete old rows. Use the store's **native TTL** (DynamoDB TTL, Cassandra TTL), or a background sweeper that deletes in batches during off-peak hours.
3. Cache TTL ≤ time remaining until the link expires, so the cache never serves an expired link.

**Should expired codes be reused?** **No.** The old code is still printed on posters and in old emails. Reusing it sends those people to someone else's URL, which is a safety and phishing issue. The space is large enough that you never need to reuse.

### 7.4 Analytics (the extension)

The rule: **analytics must never be on the redirect's critical path.**

```
Redirect service ──(async, non-blocking)──► Kafka topic "clicks" (spread evenly, NOT keyed by code)
                                                │
                              ┌─────────────────┴──────────────────┐
                              ▼                                    ▼
                  Stream job (Flink): per-code             Raw events → object storage
                  counts per minute, windowed               (recompute exactly in batch
                  by event time                              if billing depends on it)
                              ▼
                  OLAP store (ClickHouse/Druid) ──► dashboard "clicks per day, by country"
```

- **Why not key by code?** A viral link would send all its clicks to *one* partition, and one consumer would fall behind. That's the hot-key problem again, moved into Kafka. Spread events evenly across partitions, have each stream task **pre-aggregate** locally ("code X: 5,000 clicks this minute"), and then merge the partial counts. Counting doesn't need ordering, so there's no reason to pay for it.
- If Kafka is slow or down, **drop or buffer the event locally, and still redirect.** Losing a click count is acceptable; failing a redirect isn't.
- **Unique visitors** per link → HyperLogLog (fundamentals §34), not a set of IPs.
- This is where 302 vs 301 comes back: 301 means most repeat clicks never reach you, so the analytics are undercounted.

### 7.5 Abuse and security

- **Rate-limit creation** per user/API key/IP (token bucket at the gateway), because spammers mass-create links.
- **Malicious destinations:** check URLs against a threat list (e.g. Google Safe Browsing) asynchronously after creation, and disable flagged codes. Short links are a classic phishing vehicle.
- **Enumeration:** solved by the scrambled/random codes in §5. Also rate-limit `404`s per IP, since a scanner produces lots of them.
- **Open-redirect abuse:** optionally show an interstitial "you're leaving to X" page for flagged or anonymous links.

### 7.6 Multi-region

- Redirects are read-only and latency-sensitive, so serve them from **every region** with local cache and local read replicas.
- **Keep creation in one home region.** At ~1K writes/s there's no capacity reason for multi-region writes, and a single write region avoids any cross-region uniqueness question. The creator pays ~100–200 ms extra once per link, which nobody notices. *This is a good example of choosing the simpler design because the numbers allow it.*
- If multi-region writes are truly required (e.g. data residency), give each region **its own ID range space**, so uniqueness still holds by construction without coordination.
- Replication lag: a link created in the home region and clicked from another region a moment later may not have replicated yet. Fix: on a local miss, **check the home region before returning 404**. The home region always has the truth.

---

## 8. Failure modes and bottlenecks

| What fails | Impact | Mitigation |
|---|---|---|
| **Redis node dies** | Its keys all miss, and that traffic hits the DB | Redis replicas + automatic failover; DB read replicas sized to absorb a partial cache loss; in-process cache absorbs the hottest keys |
| **Entire cache cold** (after deploy/flush) | 100K+ reads/s on the DB, which can't cope | Warm the cache with the top-N links before cutting over; single-flight; rate-limit misses and shed if necessary |
| **Ticket store down** | New ranges can't be issued | Servers hold buffered ranges (minutes to hours of headroom); a replicated ticket store; creation degrades while **redirects are unaffected** |
| **KV shard down** | Its links can't be read on a cache miss | RF=3 with quorum reads; hot links still served from the cache |
| **Kafka down** | Click events lost | Redirect proceeds anyway; buffer or drop the events. Analytics is a best-effort path |
| **Viral link** | Hot key on one cache shard | In-process cache + key replication (§7.1) |
| **Bot scanning random codes** | Cache penetration, DB load | Negative caching + Bloom filter + rate-limit 404s |

**Name the bottleneck before they ask:** *"At this scale the bottleneck is the read path's cache tier, specifically hot keys and cold-start behaviour. The write path is almost idle by comparison: 1K writes/s is nothing."*

---

## 9. How the design evolves with scale

This is a strong way to show judgement: the *right* design depends on the numbers.

| Scale | Design |
|---|---|
| **Side project** (1K links/day) | One Postgres, `code` as PK, random 7-char code + retry on conflict. No cache. Done in an afternoon, and *correct* |
| **Startup** (1M links/day, 1K reads/s) | Add Redis cache-aside, read replica, stateless app servers behind an LB |
| **bit.ly scale** (100M/day, 100K+ reads/s) | Everything in §6: range-based IDs, sharded KV store, cache tier with hot-key protection, async analytics, multi-region reads |

> *"I wouldn't build the ticket-store design on day one. Random codes with a unique-constraint retry are fine until the collision rate or write volume says otherwise, and the migration path is clean because the code format doesn't change."*

---

## 10. Interviewer follow-ups — with answers

**"Why not just use an auto-increment ID from the database?"**
It's a single-writer bottleneck and SPOF for ID assignment, it's hard to scale across shards/regions, and the IDs are sequential and so enumerable. Range allocation keeps the "unique by construction" benefit without the per-request coordination.

**"Why not UUIDs?"**
They're 128 bits, which is ~22 base62 characters. That defeats the purpose of a *short* URL. You could truncate a UUID, but then you're back to Approach A's collision problem.

**"How do you guarantee two servers never produce the same code?"**
Disjoint ID ranges from an atomic ticket store (or disjoint KGS batches), plus a bijective scramble so distinct IDs always give distinct codes. The conditional insert is a belt-and-braces check. Uniqueness is guaranteed by construction, and the DB constraint verifies it.

**"What happens if the same user shortens the same URL twice?"**
It's a product decision. By default, two separate links. If the product wants dedup, look up `(user_id, hash(long_url))` first. The idempotency key covers *retries* of a single request, which is a different problem.

**"A link goes viral: 1M clicks/minute. What breaks?"**
One Redis shard gets hot. The in-process cache on each redirect instance absorbs it: 1M/min spread over, say, 50 instances is ~330 req/s each, all served from local memory. For stampede on expiry: single-flight plus jittered TTL.

**"How would you delete a link that's been cached everywhere?"**
Delete from the DB, `DEL` from Redis, and let the in-process caches' short TTLs (seconds) expire. With 302 the browser doesn't cache the redirect. With 301 you *can't* fully recall it from browsers, which is exactly why 301 is risky for links that might need to be revoked (e.g. a phishing link).

**"How long would it take to run out of codes?"**
At 100M/day: 3.5 T / 36.5 B per year ≈ 95 years. Move to 8 characters if needed. Old 7-char codes stay valid because length can grow without changing existing links.

**"Why a KV store over Postgres?"**
The only hot query is a primary-key lookup, and there are no joins or transactions. A KV store scales that horizontally with no operational sharding work and gives TTL natively. Postgres is equally correct; the choice is about operational cost at 550 TB, not capability.

---

## 11. The 45-minute script

| Time | What you say / do |
|---|---|
| 0–5 | Requirements: shorten, redirect, custom alias, expiry; analytics as an extension. **Ask the read:write ratio.** State uniqueness + unguessability + redirect availability as the key non-functionals |
| 5–10 | Estimation: 1K writes/s, 100K reads/s, 365 B URLs → **derive 7 characters** from 62⁶ vs 62⁷. ~550 TB → horizontal store. "Writes are easy, reads are the design driver" |
| 10–15 | API (+ **301 vs 302** decision), data model, KV store justification |
| 15–25 | HLD diagram; walk the write path and the read path end to end |
| 25–40 | **Deep dive on code generation** (hash → collisions via birthday maths → counter → ranges → scramble). Then caching: hot key, penetration, stampede |
| 40–45 | Failure modes, bottleneck (cache tier), analytics pipeline if time allows, how it evolves with scale |

---

## 12. Intuition takeaways

These are the lessons that carry over to *other* questions. That's why this problem is worth learning well.

1. **Derive constants from maths, don't pick them.** "7 characters" is a conclusion from 62⁷ vs 365 B, not a guess. Interviewers remember candidates who derive.
2. **Collisions come far earlier than intuition says (birthday paradox, ~√N).** Never trust a truncated hash to be unique.
3. **"Unique by construction" beats "random, then check".** Make the bad state impossible instead of detected.
4. **Coordination is only expensive per request.** Batch it (ranges, KGS batches) and it effectively disappears. The same idea shows up in Snowflake worker IDs, Kafka producer batching, and DB sequence caching.
5. **Wasting a cheap resource is fine.** Losing 1M IDs on a crash doesn't matter when you have 10× headroom. Knowing *what doesn't need protecting* is as important as knowing what does.
6. **Status codes are caching decisions.** 301 vs 302 is "who caches, and do I keep control?"
7. **Immutable data is a caching dream.** The hard parts are the extremes: hot keys and keys that don't exist.
8. **Keep side effects off the critical path.** Analytics is async and best-effort; the redirect must never wait for it or fail because of it.
9. **Separate paths with opposite profiles** (read vs write services) so each scales and fails independently.
10. **Reversible means collision-free.** Encrypting a unique counter gives unique outputs, and hashing doesn't. The same logic is behind format-preserving encryption, and it's why you can obfuscate IDs safely.
11. **Size caches by working set, not total data.** 300 GB of hot links vs 180 TB total. Ask "what's actually being accessed?"
12. **Every cache needs a story for when the truth changes, and that includes negative caches and Bloom filters.** A cached "not found" is still a cached value.
13. **Don't partition by a key you don't need ordering on.** Keying by a hot entity just moves the hot-key problem into Kafka.
14. **Say what's *not* a problem.** "Bandwidth is 50 MB/s, which is trivial" and "writes are only 1K/s" are signals of judgement, not filler.

---
