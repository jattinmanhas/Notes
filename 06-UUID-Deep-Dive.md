# UUID — Complete Deep Dive

> **Why this gets asked so much:** UUIDs sit at the intersection of distributed systems, database internals, and security — three areas interviewers want to probe. And almost everyone has *used* them while almost nobody knows *why UUIDv4 as a primary key can halve your write throughput*. That gap is exactly what the question is testing.
>
> **Spec note:** the current standard is **RFC 9562 (May 2024)**, which obsoletes RFC 4122 (2005). It adds versions 6, 7 and 8. If you cite "RFC 4122" you're citing a superseded document — citing 9562 instead is a cheap, real signal.

---

## Table of Contents

1. [What a UUID Actually Is](#1-what-a-uuid-actually-is)
2. [Anatomy — the 128 Bits](#2-anatomy--the-128-bits)
3. [Every Version, v1 to v8](#3-every-version-v1-to-v8)
4. [Collision Math](#4-collision-math)
5. [The Database Problem (the big one)](#5-the-database-problem-the-big-one)
6. [Fixing It — UUIDv7, ULID, Byte Reordering](#6-fixing-it--uuidv7-ulid-byte-reordering)
7. [Storage — How to Actually Persist a UUID](#7-storage--how-to-actually-persist-a-uuid)
8. [UUID vs Auto-Increment vs Snowflake vs ULID](#8-uuid-vs-auto-increment-vs-snowflake-vs-ulid)
9. [Java Internals](#9-java-internals)
10. [Implementing UUIDv7 in Java](#10-implementing-uuidv7-in-java)
11. [Security Considerations](#11-security-considerations)
12. [Architecture Patterns](#12-architecture-patterns)
13. [Interview Q&A — 30 Questions](#13-interview-qa--30-questions)
14. [Cheat Sheet](#14-cheat-sheet)

---

## 1. What a UUID Actually Is

A **128-bit number** intended to be unique across space and time **without any central coordinator**. That last clause is the entire point — it's the reason UUIDs exist and the answer to "why not just use an auto-increment ID?"

```
550e8400-e29b-41d4-a716-446655440000
└──8───┘ └4─┘ └4─┘ └4─┘ └────12────┘

36 characters: 32 hex digits + 4 hyphens
128 bits = 16 bytes
Format: 8-4-4-4-12
```

**GUID** (Microsoft's term) is the same thing. The only wrinkle is that some Microsoft tooling historically serialised the first three fields in **little-endian** byte order, which is why a GUID's byte array can look "scrambled" compared to its string form.

### What problem does it solve?

| Without coordination you cannot… | UUID lets you… |
|---|---|
| Generate an ID on a client that's offline | Create records offline, sync later, no conflicts |
| Merge two databases | Merge without renumbering everything |
| Insert into a sharded system without a round trip | Generate on the app server, write straight to the shard |
| Know the ID before the insert | Build object graphs in memory, insert in one batch |
| Avoid leaking business volume | Not expose "we have 4,732 customers" in a URL |

**The core trade:** you buy zero coordination and pay with 16 bytes (vs 4 or 8), no natural ordering (unless you pick v7), and worse index locality. Every UUID question is ultimately about that trade.

---

## 2. Anatomy — the 128 Bits

Not all 128 bits are random. **Six bits are reserved**, and knowing which ones is a standard question.

```
 550e8400 - e29b - 41d4 - a716 - 446655440000
                    ▲      ▲
                    │      └── VARIANT (first 1–3 bits of this group)
                    └───────── VERSION (first hex digit of the 3rd group)
```

### The version nibble

The **13th hex digit** (first character of the third group) is the version: `1`–`8`.

```
550e8400-e29b-41d4-a716-446655440000
              ▲
              └── '4' → UUIDv4 (random)

019535d9-3df7-79fb-b466-fa907fa17f9e
              ▲
              └── '7' → UUIDv7 (time-ordered)
```

### The variant bits

The **17th hex digit** (first character of the fourth group) encodes the variant in its leading bits:

| Leading bits | Variant | Possible hex digit |
|---|---|---|
| `0xx` | NCS (legacy Apollo) | `0`–`7` |
| **`10x`** | **RFC 4122 / 9562 — the one everyone uses** | **`8`, `9`, `a`, `b`** |
| `110` | Microsoft GUID (legacy) | `c`, `d` |
| `111` | Reserved | `e`, `f` |

> **Quick trick worth knowing:** if the 17th character isn't `8`, `9`, `a` or `b`, it isn't a standard RFC-variant UUID. Interviewers sometimes hand you a string and ask "is this valid, and what version?" — you read digit 13 for the version and digit 17 for the variant.

### Bit budget by version

| Version | Fixed/structured bits | Random bits |
|---|---|---|
| v4 | 6 (4 version + 2 variant) | **122** |
| v7 | 54 (48 timestamp + 4 version + 2 variant) | **74** |
| v1 | 6 + 48-bit node + 14-bit clock seq | ~0 (timestamp-driven) |

**So "a UUID has 128 bits of randomness" is wrong** — v4 has 122, and v7 has 74. Saying the right number is a small but real credibility marker.

---

## 3. Every Version, v1 to v8

| Ver | Basis | Sortable | Coordination | Verdict |
|---|---|---|---|---|
| **v1** | Timestamp (100 ns since 1582) + MAC address + clock sequence | ❌ (fields are in the wrong order) | MAC uniqueness | **Legacy.** Leaks the MAC address |
| **v2** | DCE Security — v1 with a POSIX UID/GID substituted | ❌ | — | **Effectively dead.** Barely implemented; skip it |
| **v3** | **MD5** hash of (namespace + name) | ❌ | None | Deterministic. Use v5 instead |
| **v4** | **Random** (122 bits) | ❌ | None | **The default everyone knows.** Great ID, bad primary key |
| **v5** | **SHA-1** hash of (namespace + name) | ❌ | None | **Deterministic** — same input always gives the same UUID |
| **v6** | v1 with the timestamp fields **reordered** to be sortable | ✅ | MAC/node | A migration path for existing v1 systems. New systems should use v7 |
| **v7** | **Unix ms timestamp + random** | ✅ | None | **The modern default.** Sortable *and* coordination-free |
| **v8** | Vendor/experimental — you define the layout | depends | depends | Escape hatch. Only version/variant bits are mandated |

### v1 in detail

```
 60-bit timestamp (100 ns intervals since 1582-10-15) + 14-bit clock sequence + 48-bit node (MAC)
```

**Two problems:** it leaks the machine's MAC address (this is how the Melissa virus author was traced in 1999), and the timestamp is split across fields **low-first**, so lexicographic sorting doesn't match chronological order. That second flaw is exactly what v6 and v7 fix.

### v3 / v5 — deterministic UUIDs

```
uuid = hash(namespace_uuid + name)
```

Same namespace + same name → **same UUID, every time, on every machine**. Predefined namespaces exist for DNS, URL, OID and X.500.

**Real uses, worth naming:**
- Deriving a stable ID for an external entity (`v5(NAMESPACE_URL, "https://example.com/products/123")`) so two services independently compute the same ID.
- **Idempotency keys** derived from request content.
- Deterministic test fixtures.

**Never for anything secret** — the input is guessable, so the UUID is guessable.

### v4 — the one everyone uses

122 random bits from a CSPRNG. Simple, no coordination, no information leak. **The problem is purely at the database layer** (§5).

### v7 — the modern answer (RFC 9562 §5.7)

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                           unix_ts_ms                          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          unix_ts_ms           |  ver  |       rand_a          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|var|                        rand_b                             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                            rand_b                             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

unix_ts_ms : 48 bits, big-endian, ms since the Unix epoch  → chronologically sortable
ver        :  4 bits = 0b0111
rand_a     : 12 bits — random, OR a sub-ms fraction / counter for intra-ms monotonicity
var        :  2 bits = 0b10
rand_b     : 62 bits — random, OR partly a counter
```

Key properties:

- **Big-endian timestamp in the most significant bits** → sorting the bytes sorts by time. This is the whole design.
- 74 bits of randomness — plenty (a v4 collision needs ~2.7×10¹⁸ UUIDs; 74 bits *per millisecond* is far beyond any real generation rate).
- 48 ms bits last until the year **10889**.
- The RFC explicitly says implementations **SHOULD use v7 instead of v1 and v6**.
- Optional **monotonic counter** in `rand_a`/`rand_b` guarantees ordering even for UUIDs generated within the same millisecond — important if you rely on the ID for ordering, not just index locality.

### v8 — custom

Only the version and variant bits are fixed; the other 122 are yours. Use it if you need to embed a tenant ID, shard key or region into the ID. **Uniqueness is entirely your responsibility** — the RFC says it "MUST NOT be assumed".

---

## 4. Collision Math

### The birthday problem

For a 50% chance of at least one collision among `N` possible values, you need roughly `1.1774 × √N` draws.

**For UUIDv4** (122 random bits, `N = 2¹²² ≈ 5.3 × 10³⁶`):

```
√(2¹²²)         ≈ 2.3 × 10¹⁸
× 1.1774        ≈ 2.71 × 10¹⁸ UUIDs for a 50% chance of ONE collision
```

Put in human terms:

```
Generating 1 billion UUIDv4 per second, continuously:
    2.71 × 10¹⁸ / 10⁹  =  2.71 × 10⁹ seconds  ≈  86 years

For a one-in-a-billion chance of a single collision:
    ~103 trillion UUIDs  (roughly 10¹⁴)
```

**The answer to give:** *"Collisions are theoretically possible but practically irrelevant — you'd need to generate a billion a second for 86 years to reach a coin-flip chance of a single duplicate. In practice a 'UUID collision' in production is always a bug in the generator, not a probability event."*

### The failure modes that *actually* cause duplicate UUIDs

This is the follow-up that separates a memorised answer from a real one:

| Cause | Why it happens |
|---|---|
| **A non-cryptographic RNG** | `Random` seeded from the clock — two JVMs starting in the same millisecond produce identical sequences |
| **Forked processes / cloned VMs** | Both inherit the same PRNG state and emit identical UUIDs |
| **Containers with poor entropy at boot** | Historically a real problem; a low-entropy seed narrows the effective space enormously |
| **A broken v1 implementation** | Duplicate MAC addresses (virtualised NICs), or a clock that jumped backwards without incrementing the clock sequence |
| **Truncating a UUID** | Storing it in `CHAR(32)`, or using only the first 8 hex characters — now you have 32 bits, and the birthday bound is ~77,000 |

> **The line:** *"The randomness of v4 is fine. What breaks is the entropy source — a seeded PRNG, a forked process, or someone truncating the value. That's where real-world 'collisions' come from."*

---

## 5. The Database Problem (the big one)

**This is the section that matters.** If you only prepare one thing about UUIDs, prepare this — it's the question behind most of the X/Twitter threads.

### Why random primary keys hurt

Indexes are **B+ trees**, kept sorted. Insert position is determined by the key's value.

**Sequential key (auto-increment):**

```
Every insert goes to the RIGHTMOST leaf page:

   [1..99] [100..199] [200..299] [300..___]  ← all inserts land here
                                      ▲
   • Only ONE page is "hot" → it stays in the buffer pool
   • Pages fill to ~100% before a new one is allocated
   • No page splits
   • Sequential I/O
```

**Random key (UUIDv4):**

```
Every insert goes to a RANDOM leaf page:

   [a3..] [7f..] [c1..] [09..] [e5..] [22..] ... thousands of pages
      ▲              ▲                  ▲
   • Every insert touches a DIFFERENT page → must be read from disk if not cached
   • Working set = the ENTIRE index, not just the tail
   • Inserting into a full middle page causes a PAGE SPLIT:
        one full page → two pages ~50% full
   • Result: index is ~2× larger, cache hit rate collapses, random I/O
```

### The four costs, named

| Cost | Mechanism |
|---|---|
| **Page splits** | Inserting into a full interior page splits it into two half-full pages. Fill factor drops from ~100% to ~50–70% → **the index is roughly 2× bigger than it needs to be** |
| **Buffer pool thrashing** | With sequential keys the hot set is one page. With random keys it's the whole index. Once the index exceeds RAM, **every insert becomes a disk read** and throughput falls off a cliff |
| **Write amplification** | Page splits rewrite whole pages. In Postgres, touching a page for the first time after a checkpoint writes a **full-page image** to the WAL — random access maximises how many distinct pages you touch, inflating WAL volume |
| **Fragmentation** | Logical order no longer matches physical order, so range scans do random I/O |

### MySQL/InnoDB — where it's worst

InnoDB uses a **clustered index**: the primary key *is* the table. Rows are physically stored inside the PK B+ tree, in PK order.

Two consequences:

1. **A random PK randomises the physical layout of your actual table data**, not just an index. Every insert lands somewhere different in the main data structure.
2. **Secondary indexes store the primary key as the row pointer.** So every secondary index carries a 16-byte UUID per entry instead of a 4-byte `INT` or 8-byte `BIGINT`.

```
Table with 5 secondary indexes, 100 M rows:
  BIGINT PK : 5 × 100M × 8 B  =  4 GB of PK copies inside secondary indexes
  UUID PK   : 5 × 100M × 16 B =  8 GB
  As CHAR(36): 5 × 100M × 36 B = 18 GB   ← and this is the common mistake
```

### Postgres — bad, but less catastrophic

Postgres uses **heap tables**: the table is separate from its indexes, so a random PK doesn't randomise row placement. But you still get random *index* insertion, so page splits, buffer thrash and WAL inflation all still apply — just to the index rather than the table.

Postgres-specific extras: index bloat requires `VACUUM`, and **B-tree deduplication** (Postgres 13+) works poorly on high-cardinality random values.

### Orders of magnitude

Published benchmarks vary a lot with hardware and dataset size, so don't quote a precise figure. The honest, defensible framing:

> *"While the index fits in memory the difference is modest. Once it exceeds RAM, random-key inserts degrade sharply — commonly reported as several-fold lower throughput — because every insert becomes a random disk read. The crossover point is what matters, and it arrives much sooner with UUIDv4 because the index is roughly twice as large to begin with."*

That answer is better than a memorised "it's 3× slower", because it explains *when* and *why*.

### What is NOT the problem

Be precise — half the internet is wrong about this:

- **It's not the 16 bytes.** 16 vs 8 bytes matters for storage, but it isn't what tanks insert throughput.
- **It's not "UUIDs are slow to generate".** `randomUUID()` costs well under a microsecond.
- **It's the randomness of the insertion position.** A sequential 16-byte key (UUIDv7) behaves almost exactly like an auto-increment key. **That distinction is the whole answer.**

---

## 6. Fixing It — UUIDv7, ULID, Byte Reordering

Three approaches, in order of preference.

### 1. Use UUIDv7 ✅

Time-ordered in the most significant bits, so inserts go to the right-hand edge of the index just like an auto-increment key — while keeping zero coordination.

```
UUIDv4 insert positions:  ▓ ░ ▓ ░░ ▓ ░ ▓░ ▓  ░▓   (scattered)
UUIDv7 insert positions:  ░░░░░░░░░░░░░░░░░▓▓▓▓   (right edge, like AUTO_INCREMENT)
```

Support today: **Postgres 18** has native `uuidv7([shift])`, `uuidv4()`, plus `uuid_extract_timestamp()` and `uuid_extract_version()`. Most language ecosystems have libraries; **the JDK does not ship v7**, so you use a library or the ~20 lines in §10.

**The one trade-off to acknowledge:** a v7 UUID **discloses its creation time** to anyone who has it. Usually fine; occasionally a leak (see §11).

### 2. ULID — the alternative

```
01ARZ3NDEKTSV4RRFFQ69G5FAV
└─10 chars─┘└───16 chars───┘
 48-bit ms     80-bit random

26 characters, Crockford base32 (no I, L, O, U — avoids visual ambiguity)
Lexicographically sortable as a STRING (unlike a UUID's hex form)
```

**ULID vs UUIDv7:** essentially the same idea; ULID has more randomness (80 vs 74 bits) and a shorter, case-insensitive text form, but it is **not** an IETF standard and is not a `uuid` database type. **Prefer UUIDv7** now that it's standardised — the interoperability is worth more.

### 3. Byte reordering (for legacy v1)

MySQL's `UUID_TO_BIN(uuid, swap_flag)` with `swap_flag = 1` swaps the time-low and time-high fields of a **v1** UUID so the bytes become chronologically sortable.

```sql
INSERT INTO t (id) VALUES (UUID_TO_BIN(UUID(), 1));   -- MySQL UUID() is v1
SELECT BIN_TO_UUID(id, 1) FROM t;
```

Only helps for **v1**. It does nothing for v4 — there's no time in a v4 to reorder.

### 4. The composite / dual-key pattern

Keep a `BIGINT` auto-increment as the internal clustered primary key, and a UUID as a separate **unique, externally exposed** column.

```sql
CREATE TABLE users (
  id          BIGINT AUTO_INCREMENT PRIMARY KEY,   -- internal: joins, FKs, clustering
  public_id   BINARY(16) NOT NULL UNIQUE,          -- external: URLs, APIs
  ...
);
```

**Pros:** optimal index behaviour, small foreign keys, no enumeration leak externally.
**Cons:** two identity concepts to keep straight, an extra unique index, and lookups by `public_id` need that index (fine — it's one extra hop).

> **When to choose which:** if you're on a single database and can tolerate a round trip to get an ID, the dual-key pattern is excellent. If IDs must be generated by clients or before the insert (distributed, offline-first, event-sourced), use **UUIDv7 as the primary key** and be done.

---

## 7. Storage — How to Actually Persist a UUID

**The most common real-world mistake in this whole topic:** storing a UUID as `VARCHAR(36)`.

| Storage | Bytes | Notes |
|---|---|---|
| `CHAR(36)` / `VARCHAR(36)` | **36–37** | ❌ **2.25× the space**, slower comparisons (string vs 128-bit integer), bigger indexes at every level |
| `BINARY(16)` (MySQL) | **16** | ✅ Correct. Convert with `UUID_TO_BIN` / `BIN_TO_UUID` |
| `uuid` (Postgres) | **16** | ✅ Native type — always use it, never `text` |
| Two `BIGINT` columns | 16 | Works, but loses type safety and readability |

```
100 M rows, primary key + 3 secondary indexes:
  CHAR(36):  100M × 36 B × 4  ≈  14.4 GB
  BINARY(16):100M × 16 B × 4  ≈   6.4 GB
                                 ────────
                       saved:      8 GB — and, more importantly, far more of the
                                   index now fits in the buffer pool
```

### Postgres specifics

```sql
CREATE TABLE orders (
    id          uuid PRIMARY KEY DEFAULT uuidv7(),   -- PG 18+; use gen_random_uuid() for v4
    created_at  timestamptz NOT NULL DEFAULT now()
);

SELECT uuid_extract_version('019535d9-3df7-79fb-b466-fa907fa17f9e'::uuid);   -- → 7
SELECT uuid_extract_timestamp('019535d9-3df7-79fb-b466-fa907fa17f9e'::uuid); -- → the creation time
```

Note that with a v7 primary key you can *derive* the creation timestamp from the ID — sometimes letting you drop a `created_at` column and an index.

### JDBC / JPA

```java
// Postgres: maps cleanly to the native uuid type
@Id
@Column(columnDefinition = "uuid")
private UUID id;

// MySQL: store as BINARY(16) — do NOT let Hibernate default to a 36-char string
@Id
@Column(columnDefinition = "BINARY(16)")
private UUID id;
```

```java
// Manual conversion, both directions.
public static byte[] toBytes(UUID uuid) {
    return ByteBuffer.allocate(16)
            .putLong(uuid.getMostSignificantBits())
            .putLong(uuid.getLeastSignificantBits())
            .array();
}

public static UUID fromBytes(byte[] bytes) {
    ByteBuffer bb = ByteBuffer.wrap(bytes);
    return new UUID(bb.getLong(), bb.getLong());
}
```

---

## 8. UUID vs Auto-Increment vs Snowflake vs ULID

| | **Auto-increment** | **UUIDv4** | **UUIDv7** | **Snowflake** | **ULID** |
|---|---|---|---|---|---|
| Size | 4/8 B | 16 B | 16 B | **8 B** | 16 B |
| Coordination | Central DB | **None** | **None** | Needs unique worker IDs | **None** |
| Sortable by time | ✅ | ❌ | ✅ | ✅ | ✅ |
| Index locality | ✅ Best | ❌ Worst | ✅ Good | ✅ Good | ✅ Good |
| Generate before insert | ❌ | ✅ | ✅ | ✅ | ✅ |
| Guessable / enumerable | ❌ **Leaks volume** | ✅ Safe | ⚠️ Leaks the timestamp | ⚠️ Leaks time + node | ⚠️ Leaks the timestamp |
| Text form | short | 36 chars | 36 chars | ~19 digits | **26 chars** |
| Works offline | ❌ | ✅ | ✅ | ✅ (with a worker ID) | ✅ |
| Merge two databases | ❌ Painful | ✅ | ✅ | ✅ | ✅ |

### The decision path

```
Single database, IDs never generated by clients, don't mind a round trip?
    → BIGINT auto-increment (+ a UUID public_id if URLs must not be enumerable)

Distributed / client-generated / offline-first / event-sourced?
    → UUIDv7

Need a compact 64-bit key AND strict ordering, and can run coordination?
    → Snowflake

Need IDs that reveal nothing at all, including creation time?
    → UUIDv4  (accept the index cost, or use the dual-key pattern)

Same-input-same-ID (deterministic)?
    → UUIDv5
```

> **The senior answer to "which would you pick?"**: *"UUIDv7 by default for anything distributed — it's the only option that's both coordination-free and index-friendly. I'd use a BIGINT internal key with a UUID public ID if I'm on a single database and want the tightest indexes. I'd avoid UUIDv4 as a clustered primary key specifically, though it's perfectly fine as an external identifier."*

---

## 9. Java Internals

### `java.util.UUID` is two longs

```java
public final class UUID implements Serializable, Comparable<UUID> {
    private final long mostSigBits;
    private final long leastSigBits;
}
```

16 bytes of payload; ~32 bytes on the heap with the object header. Immutable and thread-safe.

### What the JDK gives you — and what it doesn't

| Method | Version produced | Notes |
|---|---|---|
| `UUID.randomUUID()` | **v4** | Uses a lazily-initialised static `SecureRandom` |
| `UUID.nameUUIDFromBytes(byte[])` | **v3 (MD5)** | ⚠️ Frequently mistaken for v5 — **there is no v5 in the JDK** |
| `UUID.fromString(String)` | parse | |
| `new UUID(long, long)` | raw | How you build v7 yourself |
| **v1, v5, v6, v7** | — | **Not in the JDK.** Use `java-uuid-generator` (JUG), `uuid-creator`, or ~20 lines of your own |

### `randomUUID()` and `SecureRandom` contention — the real gotcha

```java
public static UUID randomUUID() {
    SecureRandom ng = Holder.numberGenerator;     // lazily initialised singleton
    byte[] randomBytes = new byte[16];
    ng.nextBytes(randomBytes);
    randomBytes[6]  &= 0x0f;  randomBytes[6]  |= 0x40;   // set version to 4
    randomBytes[8]  &= 0x3f;  randomBytes[8]  |= 0x80;   // set variant to IETF
    return new UUID(randomBytes);
}
```

Three things worth knowing:

1. **It's a single shared `SecureRandom`.** `SecureRandom` is thread-safe, but many providers synchronise internally — so under heavy multithreaded generation it can become a contention point. Fix: a `ThreadLocal<SecureRandom>`, or a pooled generator. Only worth doing if you've *measured* it.
2. **Entropy source.** On Linux the default `NativePRNG` reads `/dev/urandom`, which is non-blocking. The classic "my app hangs on startup" story comes from `/dev/random` blocking on a low-entropy VM; you can force non-blocking with `-Djava.security.egd=file:/dev/./urandom`. Largely historical on modern kernels, but it's a great war story to have.
3. **You can see the version/variant bits being set** in those four lines — a nice concrete thing to point at when explaining §2.

### Performance notes

- `randomUUID()` is on the order of **hundreds of nanoseconds** — irrelevant next to any database call. **Never** claim generation cost is the reason to avoid UUIDs.
- `toString()` has a fast path since JDK 9. `fromString()` is comparatively slow — avoid parsing in hot loops; keep `UUID` objects or `byte[16]`.
- **Never use `Math.random()` or `new Random()`** to build a UUID. `Random` has a 48-bit seed, so you get at most 2⁴⁸ distinct values — a birthday collision at ~2×10⁷ UUIDs, which is entirely reachable.

---

## 10. Implementing UUIDv7 in Java

Two versions: the simple one, and the monotonic one you'd actually ship.

```java
import java.security.SecureRandom;
import java.util.UUID;

/** Minimal RFC 9562 §5.7 UUIDv7: 48-bit ms timestamp + 74 random bits. */
public final class UuidV7 {

    private static final SecureRandom RANDOM = new SecureRandom();

    public static UUID generate() {
        long timestampMs = System.currentTimeMillis();   // 48 bits are plenty until year 10889

        byte[] rand = new byte[10];
        RANDOM.nextBytes(rand);

        // ---- most significant 64 bits: unix_ts_ms(48) | ver(4) | rand_a(12) ----
        long msb = (timestampMs & 0xFFFF_FFFF_FFFFL) << 16;   // timestamp into the top 48 bits
        msb |= (long) (rand[0] & 0x0F) << 8;                  // rand_a high nibble
        msb |= (rand[1] & 0xFFL);                             // rand_a low byte
        msb &= ~(0xFL << 12);                                 // clear the version nibble
        msb |= (0x7L << 12);                                  // set version = 7

        // ---- least significant 64 bits: var(2) | rand_b(62) ----
        long lsb = 0;
        for (int i = 2; i < 10; i++) lsb = (lsb << 8) | (rand[i] & 0xFFL);
        lsb &= 0x3FFF_FFFF_FFFF_FFFFL;                        // clear the top 2 bits
        lsb |= 0x8000_0000_0000_0000L;                        // set variant = 0b10

        return new UUID(msb, lsb);
    }

    /** Recover the creation time — one of v7's nicest properties. */
    public static long timestampOf(UUID uuid) {
        return uuid.getMostSignificantBits() >>> 16;
    }
}
```

### The monotonic version (what you'd actually ship)

Within a single millisecond, plain v7 UUIDs are randomly ordered relative to each other. If you rely on the ID for ordering — not just index locality — use a counter in `rand_a`, as the RFC's Method 1 permits.

```java
import java.security.SecureRandom;
import java.util.UUID;

/**
 * Monotonic UUIDv7. Within the same millisecond, a 12-bit counter in rand_a
 * guarantees strictly increasing IDs. On counter overflow (>4096 IDs in one
 * millisecond on one instance) we spin to the next millisecond rather than
 * risk emitting an out-of-order value.
 */
public final class MonotonicUuidV7 {

    private static final SecureRandom RANDOM = new SecureRandom();
    private static final int MAX_COUNTER = 0xFFF;   // 12 bits

    private long lastTimestampMs = -1L;
    private int counter = 0;

    public synchronized UUID generate() {
        long now = System.currentTimeMillis();

        if (now > lastTimestampMs) {
            lastTimestampMs = now;
            counter = RANDOM.nextInt(MAX_COUNTER >>> 1);   // seed low, leaving room to climb
        } else if (now == lastTimestampMs) {
            if (++counter > MAX_COUNTER) {
                now = waitForNextMillis(lastTimestampMs);  // overflow → next millisecond
                lastTimestampMs = now;
                counter = RANDOM.nextInt(MAX_COUNTER >>> 1);
            }
        } else {
            // ⚠️ CLOCK WENT BACKWARDS (NTP step). Never emit a smaller timestamp —
            // hold the previous value and keep incrementing the counter instead.
            if (++counter > MAX_COUNTER) {
                throw new IllegalStateException("clock moved backwards and counter exhausted");
            }
            now = lastTimestampMs;
        }

        long msb = (now & 0xFFFF_FFFF_FFFFL) << 16;
        msb |= (0x7L << 12);                     // version 7
        msb |= (counter & MAX_COUNTER);          // monotonic counter in rand_a

        long lsb = RANDOM.nextLong();
        lsb &= 0x3FFF_FFFF_FFFF_FFFFL;
        lsb |= 0x8000_0000_0000_0000L;           // variant

        return new UUID(msb, lsb);
    }

    private long waitForNextMillis(long last) {
        long now = System.currentTimeMillis();
        while (now <= last) { Thread.onSpinWait(); now = System.currentTimeMillis(); }
        return now;
    }
}
```

> **Points this scores:** handling **clock-backwards** explicitly (the same failure that breaks Snowflake), handling **counter overflow**, and knowing that `synchronized` here is fine because generation is nanoseconds — but that a `ThreadLocal` generator per thread scales better if you measure contention.

---

## 11. Security Considerations

### UUIDv4 is *usually* safe as a token — but be careful

122 bits of CSPRNG output is more entropy than a typical session token. **However:**

- It's only secure if the source is a **CSPRNG**. `UUID.randomUUID()` uses `SecureRandom` ✅. A hand-rolled UUID from `Random` ❌ — that's 48 bits of state and is predictable.
- **UUIDv1, v3, v5, v6, v7 are all guessable or partly guessable.** v1/v6 embed a MAC and a timestamp; v3/v5 are a hash of known input; v7 embeds the creation time. **Never use these as secrets.**
- Best practice remains a purpose-built token (32+ bytes from `SecureRandom`, base64url) rather than overloading an identifier as a credential. Even with v4, an ID tends to leak into logs, URLs and analytics in ways a token shouldn't.

### What each version leaks

| Version | Leaks |
|---|---|
| v1 / v6 | **MAC address** of the generating machine + precise creation time |
| v3 / v5 | Nothing directly, but is **fully predictable** if the input is guessable |
| v4 | Nothing |
| v7 | **Creation timestamp to the millisecond** |
| v8 | Whatever you put in it |

**Where the v7 timestamp leak actually matters:** it lets an outsider infer sign-up times, order volume over a period, or correlate records across systems. If a user-facing ID must reveal nothing, use v4 externally (or the dual-key pattern) and v7 internally.

### Enumeration

The strongest argument *for* UUIDs in public URLs: `/orders/1042` invites `/orders/1043`, and it advertises your volume. `/orders/019535d9-3df7-79fb-b466-fa907fa17f9e` does neither.

> ⚠️ **But an unguessable ID is not authorisation.** IDOR (insecure direct object reference) is still a vulnerability — always check that the caller is allowed to see the object. "Unguessable" is defence in depth, never the access-control mechanism. Saying this unprompted is a strong security signal.

---

## 12. Architecture Patterns

### Client-generated IDs

Let the client generate the UUID and send it with the create request:

```
POST /orders   { "id": "0195...", "items": [...] }
```

**Wins:** the request becomes naturally **idempotent** (a retry with the same ID hits a unique constraint and can be resolved to the existing record), the client can build an object graph offline, and there's no round trip to obtain an ID.

**Caveat:** never trust a client-supplied ID for anything but identity — validate the format, and never let it be a security-relevant value.

### UUID as an idempotency key

The payment-retry design in these notes uses exactly this: `paymentId` (a UUID) becomes the gateway's idempotency key, stable across every retry. A UUID is ideal here because it's unique without coordination and can be generated before any work begins.

### Deterministic IDs with v5

```java
// Both services independently derive the same UUID for the same external entity —
// no shared table, no lookup service.
UUID productId = uuidV5(NAMESPACE_URL, "https://partner.com/products/SKU-1234");
```

Useful for reconciling entities across systems and for idempotent imports.

### Sharding on a UUID

A UUIDv4 hashes evenly, so `hash(uuid) % N` distributes perfectly. **A UUIDv7 does not** — the leading bits are a timestamp, so range-sharding on a v7 sends *all current writes to one shard*. If you shard on v7, hash the whole value first, or shard on a different key.

> This is a genuinely good follow-up to be ready for: *"UUIDv7 fixes index locality but creates a hot shard if you range-partition on it. Hash-partition instead, or shard on the tenant/user."*

### Exposed vs internal identity

| Layer | Identifier | Why |
|---|---|---|
| Public API / URLs | UUID | Non-enumerable, stable, safe to share |
| Internal joins / FKs | `BIGINT` (dual-key) or the same UUID | Smaller indexes, faster joins |
| Logs / traces | UUID | Correlatable across services |

---

## 13. Interview Q&A — 30 Questions

### Fundamentals

**1. What is a UUID and how big is it?**
128 bits / 16 bytes, rendered as 36 characters (32 hex + 4 hyphens) in the 8-4-4-4-12 format. It's designed to be unique across space and time **without central coordination** — that last part is the reason it exists.

**2. How many bits are actually random in a UUIDv4?**
**122**, not 128 — 4 bits are the version and 2 are the variant.

**3. How do you tell which version a UUID is?**
The 13th hex digit (first character of the third group) is the version. The 17th hex digit encodes the variant; for standard RFC UUIDs it's `8`, `9`, `a` or `b`.

**4. Difference between v1 and v4?**
v1 = timestamp + MAC address + clock sequence; deterministic-ish, leaks the machine identity, and isn't sortable because the timestamp fields are stored low-order first. v4 = 122 random bits; leaks nothing, no ordering at all.

**5. What are v3 and v5, and what's the difference?**
Both are deterministic hashes of (namespace + name): v3 uses **MD5**, v5 uses **SHA-1**. Same input → same UUID on every machine. Prefer v5. Note `UUID.nameUUIDFromBytes()` in Java is **v3**, not v5.

**6. What's UUIDv6, and why does it exist?**
v1 with the timestamp fields reordered so the value sorts chronologically. It exists as a **migration path** for systems already on v1. New systems should use v7 — the RFC says so explicitly.

**7. What is UUIDv8 for?**
Vendor/experimental. Only version and variant bits are mandated; the other 122 are yours (embed a tenant ID, shard key, region). Uniqueness becomes entirely your responsibility.

**8. Which RFC?**
**RFC 9562 (May 2024)**, which obsoletes RFC 4122 and added v6, v7 and v8.

### Collisions

**9. Can two UUIDs collide?**
Theoretically yes, practically no. For v4 you need ~2.71 × 10¹⁸ UUIDs for a 50% chance of a single collision — a billion per second for ~86 years.

**10. Show me the maths.**
Birthday bound: `n ≈ 1.1774 × √N` for a 50% chance. With `N = 2¹²²`, `√N ≈ 2.3 × 10¹⁸`, so `n ≈ 2.7 × 10¹⁸`.

**11. So why do people report UUID collisions in production?**
Not probability — **broken generators**. A non-cryptographic `Random` seeded from the clock, forked processes or cloned VMs sharing PRNG state, containers booting with no entropy, duplicate MACs in v1 on virtualised NICs, or someone truncating the UUID to fit a column.

**12. Do you need a unique constraint on a UUID primary key?**
A primary key is a unique constraint. The real point: **never rely on probability alone for correctness.** The database constraint is what actually guarantees uniqueness; the UUID just makes violations vanishingly unlikely.

### Database performance — expect several of these

**13. Why is UUIDv4 a bad primary key?**
Indexes are sorted B+ trees. A random key means every insert targets a random leaf page: page splits (index ~2× bigger), buffer-pool thrashing (the working set becomes the whole index instead of one hot page), write amplification, and fragmentation. Once the index exceeds RAM, throughput falls sharply.

**14. Is it the 16 bytes that cause the problem?**
No — that's the common misconception. **It's the randomness of the insertion position.** A sequential 16-byte key (UUIDv7) performs close to an auto-increment key.

**15. Why is it worse in MySQL than Postgres?**
InnoDB uses a **clustered index** — the primary key *is* the table, so a random PK randomises your actual row storage. Also, every **secondary index stores the PK** as its row pointer, so a 16-byte UUID inflates every secondary index. Postgres uses heap tables, so only the index suffers.

**16. How do you fix it?**
Best: **UUIDv7** — time-ordered, so inserts append at the right edge of the index. Alternatives: ULID, byte-swapping for legacy v1 (`UUID_TO_BIN(u, 1)`), or the **dual-key pattern** (BIGINT internal PK + UUID public column).

**17. How would you store a UUID in MySQL?**
`BINARY(16)` with `UUID_TO_BIN` / `BIN_TO_UUID`. **Never `VARCHAR(36)`** — that's 2.25× the space in the PK and in every secondary index, plus slower string comparisons.

**18. And in Postgres?**
The native `uuid` type (16 bytes). Postgres 18 has `uuidv7()` and `uuidv4()` built in, plus `uuid_extract_timestamp()` and `uuid_extract_version()`.

**19. Can you drop `created_at` if you use v7?**
You can derive the creation time from the ID, which sometimes lets you drop the column and its index. Be careful: the timestamp is when the **ID was generated**, not necessarily when the row was committed, and it exposes creation time to anyone holding the ID.

### Design and trade-offs

**20. UUID vs auto-increment — when would you pick each?**
Auto-increment for a single database where you can afford a round trip: smallest indexes, best locality. UUID when IDs must be generated without coordination — distributed writes, client/offline generation, database merges — or when public IDs must not be enumerable.

**21. What does an auto-increment ID leak?**
Volume and growth rate. `/orders/1042` tells a competitor roughly how many orders you've had, and inviting `/orders/1043` makes enumeration trivial.

**22. UUIDv7 vs Snowflake?**
Snowflake is 8 bytes (half the size, better for indexes and network) and strictly ordered per node, but it **requires coordinated worker IDs** and breaks if the clock steps backwards. UUIDv7 is 16 bytes but needs zero coordination. Pick Snowflake if you already have coordination infrastructure and want a compact key; UUIDv7 otherwise.

**23. UUIDv7 vs ULID?**
Nearly identical designs — 48-bit ms timestamp plus randomness. ULID has 80 random bits vs 74 and a shorter, case-insensitive 26-character base32 form. UUIDv7 is an **IETF standard** and maps to native database `uuid` types. Prefer v7 for interoperability.

**24. Can you shard on a UUID?**
On v4, yes — it hashes evenly. On **v7, be careful**: the leading bits are a timestamp, so range-partitioning sends all current writes to one shard. Hash the whole value, or shard on tenant/user instead.

**25. How do you generate IDs offline / on a mobile client?**
UUIDv7 on the client. The record is created locally with its final ID and syncs later with no renumbering and no conflicts. This is also what makes the create request idempotent on retry.

### Java specifics

**26. Is `UUID.randomUUID()` thread-safe? Is it a bottleneck?**
Thread-safe, yes — it uses a shared `SecureRandom`. It **can** become a contention point under heavy multithreaded generation because many `SecureRandom` providers synchronise internally. Fix with a `ThreadLocal<SecureRandom>` — but only after measuring; generation is only hundreds of nanoseconds.

**27. Does `UUID.nameUUIDFromBytes()` give you v5?**
No — **v3 (MD5)**. The JDK has no v5, and no v7 either. Use a library (JUG, uuid-creator) or implement it.

**28. Why not build a UUID from `new Random()`?**
`Random` has a **48-bit seed**, so at most 2⁴⁸ distinct streams. The birthday bound drops to ~2×10⁷ — collisions become reachable. Always use `SecureRandom`.

### Security

**29. Can you use a UUID as a session token or password-reset link?**
A **v4** from `SecureRandom` has 122 bits of entropy, which is enough entropy-wise. But v1/v3/v5/v6/v7 are guessable or partly guessable, and even v4 tends to leak into logs, URLs and analytics. Best practice is a purpose-built token (32 bytes from `SecureRandom`, base64url) rather than reusing an identifier as a credential.

**30. Is an unguessable UUID a substitute for authorisation?**
**No.** That's IDOR. Always check the caller is entitled to the object. Unguessability is defence in depth, never the access-control mechanism.

---

## 14. Cheat Sheet

```
SIZE          128 bits = 16 bytes = 36 chars (8-4-4-4-12)
SPEC          RFC 9562 (May 2024), obsoletes RFC 4122
VERSION       13th hex digit          VARIANT   17th hex digit (8/9/a/b)
RANDOM BITS   v4 → 122      v7 → 74

VERSIONS
  v1  time + MAC          leaks MAC, not sortable        legacy
  v2  DCE security        effectively dead
  v3  MD5(ns + name)      deterministic
  v4  random              the classic; bad clustered PK
  v5  SHA-1(ns + name)    deterministic — prefer over v3
  v6  reordered v1        migration path only
  v7  unix_ms + random    ⭐ modern default: sortable, no coordination
  v8  custom              vendor/experimental

COLLISIONS
  50% chance after ~2.71 × 10¹⁸ v4 UUIDs  (1 B/sec for ~86 years)
  Real "collisions" = bad RNG, forked PRNG state, or truncation

THE DB PROBLEM
  Random key → random leaf page → page splits + buffer thrash + WAL bloat
  NOT the 16 bytes — the randomness of the INSERT POSITION
  InnoDB worst: clustered index + PK copied into every secondary index

FIXES
  UUIDv7  |  ULID  |  UUID_TO_BIN(u,1) for v1  |  BIGINT PK + UUID public_id

STORAGE
  MySQL     BINARY(16)     ← never VARCHAR(36) (2.25× space)
  Postgres  uuid           ← native, 16 bytes, uuidv7() in PG 18

JAVA
  randomUUID()         → v4, shared SecureRandom (possible contention)
  nameUUIDFromBytes()  → v3 (MD5), NOT v5
  v1/v5/v6/v7          → not in the JDK; library or ~20 lines
  never use Random()   → 48-bit seed, collisions at ~2 × 10⁷

SECURITY
  v4 safe (CSPRNG) · v1/v6 leak MAC + time · v3/v5 predictable · v7 leaks time
  Unguessable ≠ authorised. IDOR is still IDOR.
```

### The three sentences that carry the whole topic

1. *"UUIDv4's problem isn't its size, it's that random keys insert into random B+ tree pages — page splits and buffer-pool thrashing, and it's worst in InnoDB because the clustered index means the primary key is the table."*
2. *"UUIDv7 puts a 48-bit millisecond timestamp in the most significant bits, so it sorts chronologically and appends at the right edge of the index like an auto-increment key — while still needing zero coordination."*
3. *"Collisions aren't a probability problem, they're a generator problem — a seeded `Random`, forked process state, or someone truncating the value."*

---

*End of UUID deep dive.*

**Sources:** [RFC 9562 — Universally Unique IDentifiers (UUIDs)](https://www.rfc-editor.org/rfc/rfc9562.txt) · [PostgreSQL 18 — UUID Functions](https://www.postgresql.org/docs/18/functions-uuid.html)
