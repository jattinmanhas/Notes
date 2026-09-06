# Cache in Java — complete LLD walkthrough

Study order: requirements → model and flow → project tree → each complete file and explanation → patterns → walkthrough → limitations and exercises. This is a runnable single-process interview reference, with no external services or omitted source files.

## 1. Requirements and scope

Build a capacity-bounded, thread-safe LRU cache with per-entry TTL. Support put, get and delete. Get promotes a live entry to most recently used. Updating a key replaces its value and expiry. Reject nonpositive capacity and TTL. Use an injected clock so expiration tests need no sleeps. This is an in-process generic cache, with no remote storage, loading or persistence.

## 2. Design and responsibilities

```text
Main -> Cache<K,V>
        -> map key -> linked-list node (Go) / access-ordered map (Java)
        -> recency order: least used -> most used
        -> Clock for TTL
put -> remove expired -> update/insert -> evict least recently used
get -> expire if needed -> promote -> return
```

## 3. Project structure and running

Create each file at the shown relative path. All imports and package declarations are included.

```text
cache-java/
  src/study/CacheEntry.java
  src/study/Cache.java
  src/study/Main.java
  src/study/BehaviorTest.java
```

JDK 17 or newer; no Maven, Gradle or external libraries required. From the project root:

```bash
javac --release 17 -d out src/study/*.java
java -cp out study.Main
java -cp out study.BehaviorTest
```

Each top-level type has its own file. `Main` and `BehaviorTest` are public entry points. Other types are package-private in `study`; private fields protect state, while package methods expose the intended operations. A larger repository can split packages and deliberately make selected APIs public. Tests throw AssertionError explicitly and do not need `-ea`.

## 4. Every file, its code, and explanation

### 4.1. `src/study/CacheEntry.java`

```java
package study;

import java.time.Instant;

record CacheEntry<V>(V value, Instant expires) {
}
```

**How it works and why it belongs here:** Immutable wrapper stores value plus deadline. The wrapped value itself can still be mutable.

### 4.2. `src/study/Cache.java`

```java
package study;

import java.util.Objects;
import java.util.Optional;
import java.util.LinkedHashMap;
import java.time.*;
import java.util.function.Supplier;

final class Cache<K, V> {
    private final int capacity;
    private final Supplier<Instant> clock;
    private final LinkedHashMap<K, CacheEntry<V>> items = new LinkedHashMap<>(16, 0.75f, true);
    Cache(int capacity, Supplier<Instant> clock) {
        if (capacity<1) throw new IllegalArgumentException("capacity");
        this.capacity = capacity;
        this.clock = Objects.requireNonNull(clock);
    }

    synchronized void put(K key, V value, Duration ttl) {
        Objects.requireNonNull(key);
        Objects.requireNonNull(value);
        if (ttl.isZero() || ttl.isNegative()) throw new IllegalArgumentException("TTL");
        Instant now = clock.get();
        items.entrySet().removeIf(e -> !now.isBefore(e.getValue().expires()));
        items.put(key, new CacheEntry<>(value, now.plus(ttl)));
        if (items.size()> capacity) {
            var it = items.keySet().iterator();
            it.next();
            it.remove();
        }
    }
    synchronized Optional<V> get(K key) {
        var e = items.get(key);
        if (e == null) return Optional.empty();
        if (!clock.get().isBefore(e.expires())) {
            items.remove(key);
            return Optional.empty();
        }
        return Optional.of(e.value());
    }

    synchronized void delete(K key) {
        items.remove(key);
    }
}
```

**How it works and why it belongs here:** LinkedHashMap with accessOrder=true maintains LRU order internally. get changes that order and therefore must be synchronized too. Null values are forbidden so Optional.empty unambiguously means a miss. Supplier is the standard functional interface for the clock; a custom class hierarchy would add no value.

### 4.3. `src/study/Main.java`

```java
package study;

public class Main {

    public static void main(String[]args) {
        var c = new Cache<String, String>(2, java.time.Instant::now);
        c.put("A", "alpha", java.time.Duration.ofMinutes(1));
        c.put("B", "beta", java.time.Duration.ofMinutes(1));
        c.get("A");
        c.put("C", "gamma", java.time.Duration.ofMinutes(1));
        System.out.println("B present: "+c.get("B").isPresent());
        System.out.println("A: "+c.get("A").orElseThrow());
    }
}
```

**How it works and why it belongs here:** The composition root creates the collaborators explicitly and runs the example. The public entry point lives in its own file; domain collaborators remain package-private within study.

### 4.4. `src/study/BehaviorTest.java`

```java
package study;

public class BehaviorTest {

    static void check(boolean value, String message) {
        if (!value) throw new AssertionError(message);
    }

    public static void main(String[] args) throws Exception {
        var now = new java.util.concurrent.atomic.AtomicReference<>(java.time.Instant.EPOCH);
        var c = new Cache<String, Integer>(2, now::get);
        c.put("a", 1, java.time.Duration.ofSeconds(1));
        c.put("b", 2, java.time.Duration.ofHours(1));
        c.get("a");
        c.put("c", 3, java.time.Duration.ofHours(1));
        check(c.get("b").isEmpty(), "LRU");
        now.set(now.get().plusSeconds(1));
        check(c.get("a").isEmpty(), "TTL boundary");
        c.put("c", 4, java.time.Duration.ofHours(1));
        check(c.get("c").orElseThrow() == 4, "update");
        c.delete("c");
        check(c.get("c").isEmpty(), "delete");
        {
            var concurrent = new Cache<Integer, Integer>(8, java.time.Instant::now);
            var threads = new java.util.ArrayList<Thread>();
            for (int i = 0; i<8; i++) {
                int key = i;
                var thread = new Thread(() -> {
                    for (int n = 0; n<100; n++) {
                        concurrent.put(key, n, java.time.Duration.ofHours(1));
                        concurrent.get(key);
                        concurrent.delete(key);
                    }
                });
                threads.add(thread);
                thread.start();
            }
            for (var thread:threads)thread.join();
            concurrent.put(1, 42, java.time.Duration.ofHours(1));
            check(concurrent.get(1).orElseThrow() == 42, "concurrent cache consistency");
        }
        System.out.println("All behavior tests passed");
    }
}
```

**How it works and why it belongs here:** Uses an injected atomic clock for deterministic TTL tests, plus eviction, update and deletion checks. A concurrent put/get/delete smoke test exercises the shared recency structures; the Go run also uses race detection.

## 5. Patterns and principles: where and why

| Pattern or principle | Concrete location | Reason |
|---|---|---|
| LRU eviction algorithm | Cache get/put | Recency determines which live entry leaves at capacity. |
| Dependency injection | Clock / Supplier<Instant> | Makes TTL deterministic in tests. |
| Encapsulation | Cache mutex / synchronized operations | Lookup, expiry, promotion and eviction are one atomic action. |
| Composition | Map plus recency structure | Combines fast lookup with ordered eviction; this sample does not invent an unnecessary eviction Strategy interface. |

A mutex or synchronized method is a concurrency mechanism, not a GoF design pattern. An enum is a state representation, not automatically the State pattern. Interfaces are justified by interchangeable behavior or a useful boundary; inheritance is not required to demonstrate OOP.

### Applying SOLID without unnecessary abstractions

**Single responsibility:** the main entry point assembles dependencies; the domain service owns state invariants; policy collaborators own the behavior named in the pattern table. The tests exercise behavior through the public operations rather than depending on implementation maps.

**Open/closed:** inspect `Cache get/put` as the primary variation point. Where a policy interface exists, supply a new implementation without changing the state-transition algorithm. Where this example implements a specific data structure, do not claim its algorithm is interchangeable until you deliberately extract that boundary.

**Liskov substitution:** a replacement collaborator must preserve the documented contract, including invalid-input behavior, clock units, ownership rules and callback failure behavior. Merely matching a method signature is not enough.

**Interface segregation and dependency inversion:** interfaces expose the small question the caller needs answered. Constructors receive policies and clocks where tests or alternative behavior benefit. Simple records, enum values and internal containers remain concrete. A dedicated repository interface is useful when persistence is in scope; these samples do not pretend in-memory mutations automatically translate to database transactions.

## 6. Follow the example and the invariants

With capacity 2, put A and B, then get A. B becomes the least recently used live entry. Put C and B is evicted. Advancing the injected clock to A's expiry causes get A to miss. Expiry uses now >= expiresAt, so the exact boundary is expired. Put removes all expired entries first so an expired MRU entry does not cause an unrelated live LRU entry to be evicted. Get miss does not create a cache entry.

### Verified execution

The complete project above was compiled and its behavior tests executed. The following is actual validation output (paths, timing and identifiers can vary):

```text
B present: false
A: alpha

All behavior tests passed
```

## 7. Best practices, edge cases and production extensions

Get and delete are O(1) average; put is O(N) because it scans for expired entries before insertion. Do not advertise every operation as O(1). The explicit scan makes expiration behavior easy to reason about. A min-heap or timing wheel can support more efficient expiration at greater implementation cost. Expired entries may retain memory until an operation removes them, but physical entries never exceed capacity.

Returning a generic value does not deep-copy mutable objects. Cache callers should use immutable values or own their synchronization. The clock must be nondecreasing for intuitive TTL semantics; production time sources require care around wall-clock adjustments. LRU is a policy choice, not automatically best for scan-heavy workloads. Loading on misses requires a separate single-flight mechanism to prevent stampedes. Distributed caches need serialization, consistent key definitions and failure semantics; none are implicit in this mutex-protected implementation.

## 8. Presenting this in an interview

Start by agreeing on the scope in section 1. Draw the responsibility flow, identify the state that must remain consistent, and name the operation that owns that invariant. Implement the core model and service, then wire the collaborators in main and run a concrete example. Show at least one rejected or boundary case from the tests. Explain the pattern at the point where it solves a problem, rather than starting with a list of pattern names.

For a distributed follow-up, distinguish thread safety inside this process from coordination across replicas. In-memory objects do not survive restarts. Agree on consistency, failure recovery and storage requirements before replacing them with remote infrastructure.

## 9. Practice next

1. Add a size/weight limit instead of only entry count.
2. Implement a separate loading-cache wrapper with one in-flight load per key.
3. Replace the expiration scan with a heap and compare complexity.
4. Add hit/miss/eviction metrics without exposing internal mutable state.
