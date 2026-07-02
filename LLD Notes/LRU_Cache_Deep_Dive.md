# LRU Cache — Deep Dive (DSA → System Design)

> Covers everything from the raw data structure to a production-grade distributed cache.  
> Read top-to-bottom the first time; use as a reference after.

---

## Table of Contents

1. [What is LRU Cache?](#1-what-is-lru-cache)
2. [Why LRU? — Eviction Policy Comparison](#2-why-lru--eviction-policy-comparison)
3. [DSA Foundation — The Core Data Structure](#3-dsa-foundation--the-core-data-structure)
4. [Implementation 1 — Naive (LinkedHashMap cheat)](#4-implementation-1--naive-linkedhashmap-cheat)
5. [Implementation 2 — From Scratch (HashMap + DLL)](#5-implementation-2--from-scratch-hashmap--dll)
6. [Implementation 3 — Thread-Safe LRU Cache](#6-implementation-3--thread-safe-lru-cache)
7. [Complexity Analysis](#7-complexity-analysis)
8. [Edge Cases to Handle](#8-edge-cases-to-handle)
9. [System Design — Scaling LRU Cache](#9-system-design--scaling-lru-cache)
10. [Interview Q&A Cheat Sheet](#10-interview-qa-cheat-sheet)

---

## 1. What is LRU Cache?

**LRU = Least Recently Used.**

A cache with a fixed capacity that, when full, evicts the item that was accessed **least recently**.

**Real-world analogy:** Your browser's back button history. When it runs out of space, the oldest page you visited (least recently used) gets dropped.

**Where it's used:**
- CPU L1/L2/L3 cache
- Redis eviction policy (`allkeys-lru`)
- OS page replacement
- Database buffer pool (MySQL InnoDB)
- CDN edge node caching

**The two operations:**
- `get(key)` → return value if exists, else -1. **Mark as recently used.**
- `put(key, value)` → insert/update. If at capacity, **evict LRU item first.**

---

## 2. Why LRU? — Eviction Policy Comparison

| Policy | Evicts | Best for | Weakness |
|---|---|---|---|
| **LRU** | Least recently used | General-purpose, temporal locality | Doesn't account for frequency |
| **LFU** | Least frequently used | Skewed access patterns | New items evicted too fast |
| **FIFO** | Oldest inserted | Simple queues | Ignores recency entirely |
| **MRU** | Most recently used | Sequential scans | Counterintuitive for most apps |
| **Random** | Random item | Low overhead, approximations | Unpredictable |
| **ARC** | Adaptive (LRU + LFU hybrid) | Best of both | Complex to implement |

**LRU is the default choice** because most real workloads have temporal locality — recently accessed data is likely to be accessed again soon.

---

## 3. DSA Foundation — The Core Data Structure

### The Problem with Naive Approaches

| Approach | get() | put() | Eviction | Problem |
|---|---|---|---|---|
| Array | O(n) scan | O(n) shift | O(n) | Too slow |
| HashMap only | O(1) | O(1) | Don't know which is LRU | No ordering |
| Linked List only | O(n) scan | O(1) | O(1) | get() is O(n) |

### The Insight: Combine Both

```
HashMap<Key, Node>   →  O(1) lookup by key
Doubly Linked List   →  O(1) move-to-front and evict-from-tail
```

**Layout:**

```
HEAD (dummy) <-> [most recent] <-> ... <-> [least recent] <-> TAIL (dummy)
                      ↑                           ↑
                  just accessed              evict this next
```

**Rules:**
- On `get(key)`: find node via map → move it to front (just after HEAD)
- On `put(key, value)`:
  - If key exists: update value → move to front
  - If new + at capacity: remove node just before TAIL (LRU) → add new node at front
  - If new + not at capacity: just add at front

**Why doubly linked (not singly linked)?**  
To remove a node in O(1), you need a pointer to its **previous** node. Singly linked requires O(n) traversal to find prev.

---

## 4. Implementation 1 — Naive (LinkedHashMap cheat)

Good to know for quick interviews, but interviewers usually ask you to build from scratch.

```java
import java.util.LinkedHashMap;
import java.util.Map;

public class LRUCacheSimple {
    private final int capacity;
    private final LinkedHashMap<Integer, Integer> cache;

    public LRUCacheSimple(int capacity) {
        this.capacity = capacity;
        // accessOrder=true: iterates from least-recently-accessed to most
        this.cache = new LinkedHashMap<>(capacity, 0.75f, true) {
            @Override
            protected boolean removeEldestEntry(Map.Entry<Integer, Integer> eldest) {
                return size() > capacity;  // auto-evict when over capacity
            }
        };
    }

    public int get(int key) {
        return cache.getOrDefault(key, -1);
        // LinkedHashMap moves this entry to the end on access (accessOrder=true)
    }

    public void put(int key, int value) {
        cache.put(key, value);
        // removeEldestEntry() fires automatically after put
    }
}

// Usage
LRUCacheSimple lru = new LRUCacheSimple(3);
lru.put(1, 10);
lru.put(2, 20);
lru.put(3, 30);
lru.get(1);       // access key 1 → now MRU
lru.put(4, 40);   // capacity full → evicts key 2 (LRU)
System.out.println(lru.get(2)); // -1 (evicted)
System.out.println(lru.get(1)); // 10
```

**Why this works:** `LinkedHashMap` with `accessOrder=true` maintains insertion + access order. `removeEldestEntry` is a hook called after every `put`.

**Limitation:** Not thread-safe. Interviewers will ask you to build it from scratch next.

---

## 5. Implementation 2 — From Scratch (HashMap + DLL)

This is the answer interviewers want.

```java
import java.util.HashMap;
import java.util.Map;

public class LRUCache {

    // ─── Node (Doubly Linked List) ─────────────────────────────
    private static class Node {
        int key, value;
        Node prev, next;

        Node(int key, int value) {
            this.key   = key;
            this.value = value;
        }
    }

    // ─── Fields ───────────────────────────────────────────────
    private final int capacity;
    private final Map<Integer, Node> map;  // key → node (O(1) lookup)
    private final Node head, tail;         // dummy sentinels

    // ─── Constructor ──────────────────────────────────────────
    public LRUCache(int capacity) {
        this.capacity = capacity;
        this.map      = new HashMap<>();

        // Dummy sentinels eliminate null checks at boundaries
        head = new Node(0, 0);
        tail = new Node(0, 0);
        head.next = tail;
        tail.prev = head;
    }

    // ─── Public API ───────────────────────────────────────────

    public int get(int key) {
        if (!map.containsKey(key)) return -1;

        Node node = map.get(key);
        moveToFront(node);    // mark as recently used
        return node.value;
    }

    public void put(int key, int value) {
        if (map.containsKey(key)) {
            Node node = map.get(key);
            node.value = value;
            moveToFront(node);
        } else {
            if (map.size() == capacity) {
                evictLRU();   // remove least recently used
            }
            Node node = new Node(key, value);
            map.put(key, node);
            addToFront(node);
        }
    }

    // ─── Private Helpers ──────────────────────────────────────

    /** Remove node from its current position in the list */
    private void remove(Node node) {
        node.prev.next = node.next;
        node.next.prev = node.prev;
    }

    /** Insert node right after HEAD (most recently used position) */
    private void addToFront(Node node) {
        node.next      = head.next;
        node.prev      = head;
        head.next.prev = node;
        head.next      = node;
    }

    /** Move existing node to front */
    private void moveToFront(Node node) {
        remove(node);
        addToFront(node);
    }

    /** Remove node just before TAIL (least recently used) */
    private void evictLRU() {
        Node lru = tail.prev;   // node just before dummy tail
        remove(lru);
        map.remove(lru.key);
    }
}
```

### Dry Run — Trace Through

```
capacity = 3

put(1,10):  map={1→N1}    HEAD <-> [1] <-> TAIL
put(2,20):  map={1,2}     HEAD <-> [2] <-> [1] <-> TAIL
put(3,30):  map={1,2,3}   HEAD <-> [3] <-> [2] <-> [1] <-> TAIL

get(1):     move 1 to front
            HEAD <-> [1] <-> [3] <-> [2] <-> TAIL

put(4,40):  capacity full → evict TAIL.prev = [2]
            map={1,3,4}   HEAD <-> [4] <-> [1] <-> [3] <-> TAIL

get(2):     → -1  (was evicted)
get(3):     → 30  ✓
```

### Why Dummy Sentinels?

Without them, every `addToFront` and `evictLRU` needs null checks:

```java
// Without sentinels — messy null checks everywhere
if (head == null) {
    head = node; tail = node;
} else {
    node.next = head;
    head.prev = node;
    head = node;
}
```

With sentinels, `head` and `tail` are always valid nodes — no null checks needed.

---

## 6. Implementation 3 — Thread-Safe LRU Cache

In production, multiple threads hit the cache concurrently. Two approaches:

### Option A — `synchronized` (simple, coarse-grained)

```java
public class ThreadSafeLRUCache {
    private final LRUCache cache;

    public ThreadSafeLRUCache(int capacity) {
        this.cache = new LRUCache(capacity);
    }

    public synchronized int get(int key) {
        return cache.get(key);
    }

    public synchronized void put(int key, int value) {
        cache.put(key, value);
    }
}
```

**Problem:** Every operation blocks all other threads. Under high contention, this becomes a bottleneck.

### Option B — `ReentrantReadWriteLock` (better throughput)

```java
import java.util.concurrent.locks.ReentrantReadWriteLock;

public class RWLockLRUCache {
    private final LRUCache cache;
    private final ReentrantReadWriteLock lock = new ReentrantReadWriteLock();

    public RWLockLRUCache(int capacity) {
        this.cache = new LRUCache(capacity);
    }

    public int get(int key) {
        // NOTE: get() promotes to write lock because it mutates order
        lock.writeLock().lock();
        try {
            return cache.get(key);
        } finally {
            lock.writeLock().unlock();
        }
    }

    public void put(int key, int value) {
        lock.writeLock().lock();
        try {
            cache.put(key, value);
        } finally {
            lock.writeLock().unlock();
        }
    }
}
```

> **Important:** `get()` in LRU also mutates the list (moves node to front), so it needs a **write lock**, not a read lock. This is a common interview gotcha.

### Option C — Segmented / Sharded Cache (best throughput)

```java
import java.util.concurrent.locks.ReentrantLock;

public class ShardedLRUCache {
    private static final int NUM_SHARDS = 16;
    private final LRUCache[]      shards = new LRUCache[NUM_SHARDS];
    private final ReentrantLock[] locks  = new ReentrantLock[NUM_SHARDS];

    public ShardedLRUCache(int totalCapacity) {
        int shardCapacity = totalCapacity / NUM_SHARDS;
        for (int i = 0; i < NUM_SHARDS; i++) {
            shards[i] = new LRUCache(shardCapacity);
            locks[i]  = new ReentrantLock();
        }
    }

    private int shardFor(int key) {
        return Math.abs(key % NUM_SHARDS);
    }

    public int get(int key) {
        int shard = shardFor(key);
        locks[shard].lock();
        try {
            return shards[shard].get(key);
        } finally {
            locks[shard].unlock();
        }
    }

    public void put(int key, int value) {
        int shard = shardFor(key);
        locks[shard].lock();
        try {
            shards[shard].put(key, value);
        } finally {
            locks[shard].unlock();
        }
    }
}
```

**Why sharding helps:** Only 1/16 of keys contend for the same lock. Operations on different shards run truly in parallel.

---

## 7. Complexity Analysis

| Operation | Time | Space |
|---|---|---|
| `get(key)` | **O(1)** | — |
| `put(key, value)` | **O(1)** | — |
| Space (total) | — | **O(capacity)** |

Every operation is O(1) because:
- HashMap gives O(1) key lookup
- DLL gives O(1) insert/remove given a node pointer (no traversal)

---

## 8. Edge Cases to Handle

These come up in interviews — handle all of them:

```java
// 1. capacity = 1
LRUCache c = new LRUCache(1);
c.put(1, 1);
c.put(2, 2);  // evicts key 1
assert c.get(1) == -1;
assert c.get(2) == 2;

// 2. Update existing key — should NOT increase size
LRUCache c2 = new LRUCache(2);
c2.put(1, 1);
c2.put(2, 2);
c2.put(1, 10);  // update, not insert
c2.put(3, 3);   // should evict key 2, not key 1
assert c2.get(1) == 10;
assert c2.get(2) == -1;
assert c2.get(3) == 3;

// 3. get() counts as "use" — key shouldn't be evicted after get
LRUCache c3 = new LRUCache(2);
c3.put(1, 1);
c3.put(2, 2);
c3.get(1);      // key 1 is now MRU
c3.put(3, 3);   // should evict key 2 (LRU), not key 1
assert c3.get(1) == 1;
assert c3.get(2) == -1;

// 4. get() on missing key returns -1, doesn't add to cache
LRUCache c4 = new LRUCache(2);
assert c4.get(99) == -1;
assert c4.map.size() == 0;  // nothing added
```

---

## 9. System Design — Scaling LRU Cache

Once you've nailed the DSA part, the interviewer may ask: *"Now design this for 100M users."*

### Single Node → Distributed Cache

```
Single Node LRU
      ↓ (one server, limited RAM)
Sharded Cache (consistent hashing)
      ↓ (shard by key → each shard is one LRU cache)
Replicated Shards (leader + replicas per shard)
      ↓ (fault tolerance)
Redis Cluster (battle-tested, production-grade)
```

### Component Diagram

```
Clients
   │
   ▼
[Load Balancer]
   │
   ├──── [Cache Node 1]  ← shard A  (key % N == 0)
   ├──── [Cache Node 2]  ← shard B  (key % N == 1)
   └──── [Cache Node 3]  ← shard C  (key % N == 2)
              │
              ▼ (on cache miss)
         [Database]
```

### Key Design Decisions

**1. How to shard keys across nodes?**

Use **consistent hashing** (not simple `key % N`).

- `key % N` breaks when you add/remove nodes — almost every key remaps.
- Consistent hashing remaps only `K/N` keys on average when adding a node.

```
Ring:  0 ─── Node A ─── Node B ─── Node C ─── 360°

Key hash → position on ring → walk clockwise → first node = owner
```

**2. What happens on cache miss?**

```
Cache Miss Flow:
   1. Client requests key K
   2. Cache node doesn't have K → cache miss
   3. Cache node fetches K from DB
   4. Cache node stores K with TTL
   5. Returns value to client

Pattern: Cache-Aside (lazy loading)
```

**3. Write strategies**

| Strategy | How | Consistency | Complexity |
|---|---|---|---|
| **Write-through** | Write to cache + DB together | Strong | Medium |
| **Write-back** | Write to cache, async flush to DB | Eventual | Higher |
| **Write-around** | Write directly to DB, skip cache | Strong | Low |

For most LRU cache use cases, **cache-aside + write-through** is the safe default.

**4. TTL (Time-To-Live)**

Even with LRU eviction, stale data is a problem. Always set a TTL:

```
Short TTL (seconds) → high freshness, more DB load
Long TTL (minutes)  → low freshness, less DB load

Rule of thumb: TTL should be shorter than your data's staleness tolerance.
```

**5. Thundering Herd Problem**

When a hot key expires, 1000 requests simultaneously hit the DB:

```
Solution — Mutex/Lock on first miss:
   Thread 1: cache miss → acquires lock → fetches DB → populates cache → releases lock
   Thread 2-1000: cache miss → waits for lock → reads from cache (populated by Thread 1)

In Redis: use SET key value NX EX 30 (atomic set-if-not-exists with TTL)
```

**6. Hot Key Problem**

One key is so popular a single shard is overwhelmed:

```
Solutions:
  a) Local in-process cache on each app server (L1 cache) for top-N hot keys
  b) Replicate hot key across multiple shards; read from any replica
  c) Add random suffix to key: "user:42:0", "user:42:1" → spread across shards
```

### Capacity Estimation (back-of-envelope)

```
Problem: Design cache for 10M DAU, each user generates 100 reads/day

Read QPS = (10M × 100) / 86400 ≈ 12,000 reads/sec

Assume avg cached object = 1 KB
Cache for top 20% objects (Pareto): 10M objects × 20% × 1KB = 2 GB per shard

With 4 shards: 8 GB total RAM — fits on commodity servers.

At 12K QPS, even a single Redis node handles ~100K ops/sec comfortably.
Start with 1 node, shard only when you approach limits.
```

### Redis as Production LRU Cache

Redis implements LRU natively. Key config:

```bash
maxmemory 4gb
maxmemory-policy allkeys-lru   # evict any key LRU-style when full

# allkeys-lru   = evict least recently used among ALL keys
# volatile-lru  = evict least recently used among keys WITH a TTL set
# allkeys-lfu   = evict least frequently used (Redis 4.0+, often better)
```

> **Note:** Redis uses an **approximate LRU** — it samples N random keys and evicts the LRU among them (default N=5). This avoids the memory overhead of a true LRU pointer structure at scale.

---

## 10. Interview Q&A Cheat Sheet

**Q: Why HashMap + Doubly Linked List and not just a sorted map?**  
A: Sorted map (TreeMap) gives O(log n) for all ops. We need O(1). HashMap gives O(1) lookup; DLL gives O(1) reorder given a pointer. Combining them gets us O(1) for everything.

**Q: Why doubly linked, not singly linked?**  
A: To remove a node in O(1), you need a pointer to `node.prev`. Singly linked requires O(n) traversal to find it.

**Q: Why dummy head/tail sentinels?**  
A: Eliminates null checks at list boundaries. Every operation is uniform — no special cases for empty list or head/tail removal.

**Q: Does `get()` need a write lock in a concurrent LRU?**  
A: Yes. `get()` moves the node to the front of the DLL, which is a mutation. Read lock is not sufficient.

**Q: What's the difference between LRU and LFU?**  
A: LRU evicts based on *recency* (last access time). LFU evicts based on *frequency* (access count). LFU is better for stable hot keys; LRU is better for time-locality workloads. LFU is harder to implement (O(1) requires a frequency bucket + DLL per bucket).

**Q: How does Redis implement LRU if it doesn't maintain a global linked list?**  
A: Approximate LRU — each key stores a `lru_clock` timestamp (last access). On eviction, Redis samples N random keys and evicts the one with the oldest `lru_clock`. No DLL overhead, trades perfect accuracy for memory efficiency.

**Q: How do you handle cache stampede (thundering herd)?**  
A: Use a distributed mutex (Redis `SETNX`) so only one request fetches from DB while others wait. Or use probabilistic early expiration — refresh the key slightly before TTL expires.

**Q: When would you NOT use LRU?**  
A: Sequential scans (large table scans evict all your hot data — use MRU instead). Frequency-skewed access (use LFU). When you need strict TTL expiry (use TTL-based eviction).

---

*Last updated: June 2026*
