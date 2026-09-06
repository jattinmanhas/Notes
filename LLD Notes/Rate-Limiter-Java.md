# Rate Limiter in Java — complete LLD walkthrough

Study order: requirements → model and flow → project tree → each complete file and explanation → patterns → walkthrough → limitations and exercises. This is a runnable single-process interview reference, with no external services or omitted source files.

## 1. Requirements and scope

Implement a per-client token bucket. Each bucket starts full, refills continuously at a configured rate, and consumes one token for an allowed request. Denied requests consume no token. Return retryAfter for callers. Isolate different client keys, validate configuration, and inject a monotonic elapsed-time source. HTTP middleware, shared multi-server quotas and background key eviction are outside scope.

## 2. Design and responsibilities

```text
Main -> TokenBucket.Allow(client)
        -> retrieve/create bucket
        -> refill = elapsed * rate, capped at capacity
        -> enough tokens? consume one : compute retryAfter
One lock covers refill + decision + debit
```

## 3. Project structure and running

Create each file at the shown relative path. All imports and package declarations are included.

```text
rate-limiter-java/
  src/study/Decision.java
  src/study/RequestLimiter.java
  src/study/Bucket.java
  src/study/TokenBucket.java
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

### 4.1. `src/study/Decision.java`

```java
package study;

import java.time.Duration;

record Decision(boolean allowed, Duration retryAfter) {
}
```

**How it works and why it belongs here:** Immutable decision is safe to return across threads.

### 4.2. `src/study/RequestLimiter.java`

```java
package study;

interface RequestLimiter {
    Decision allow(String client);
}
```

**How it works and why it belongs here:** Consumers can choose another algorithm without depending on TokenBucket fields.

### 4.3. `src/study/Bucket.java`

```java
package study;

final class Bucket {
    double tokens;
    long last;
    Bucket(double tokens, long last) {
        this.tokens = tokens;
        this.last = last;
    }
}
```

**How it works and why it belongs here:** Mutable internal state belongs to the limiter lock; it is never exposed to callers.

### 4.4. `src/study/TokenBucket.java`

```java
package study;

import java.util.Map;
import java.util.HashMap;
import java.util.Objects;
import java.time.Duration;
import java.util.function.LongSupplier;

final class TokenBucket implements RequestLimiter {
    private final double capacity, rate;
    private final LongSupplier clock;
    private final Map<String, Bucket> buckets = new HashMap<>();
    TokenBucket(double capacity, double rate, LongSupplier clock) {
        if (!Double.isFinite(capacity) || !Double.isFinite(rate) || capacity<1 || rate <= 0) throw new IllegalArgumentException("configuration");
        this.capacity = capacity;
        this.rate = rate;
        this.clock = Objects.requireNonNull(clock);
    }
    public synchronized Decision allow(String client) {
        Objects.requireNonNull(client);
        long now = clock.getAsLong();
        var b = buckets.get(client);
        if (b == null) {
            b = new Bucket(capacity, now);
            buckets.put(client, b);
        }
        if (now<b.last)now = b.last;
        b.tokens = Math.min(capacity, b.tokens+(now-b.last)/1_000_000_000.0*rate);
        b.last = now;
        if (b.tokens >= 1) {
            b.tokens--;
            return new Decision(true, Duration.ZERO);
        }
        return new Decision(false, Duration.ofNanos((long)Math.ceil((1-b.tokens)/rate*1_000_000_000.0)));
    }
}
```

**How it works and why it belongs here:** LongSupplier supplies elapsed nanoseconds. synchronized makes refill/check/debit atomic. Main derives elapsed time from nanoTime; callers must not inject wall-clock milliseconds without unit conversion.

### 4.5. `src/study/Main.java`

```java
package study;

public class Main {

    public static void main(String[]args) {
        long start = System.nanoTime();
        RequestLimiter limiter = new TokenBucket(2, 1, () -> System.nanoTime()-start);
        for (int i = 0; i<3; i++)System.out.println("allowed: "+limiter.allow("alice").allowed());
    }
}
```

**How it works and why it belongs here:** The composition root creates the collaborators explicitly and runs the example. The public entry point lives in its own file; domain collaborators remain package-private within study.

### 4.6. `src/study/BehaviorTest.java`

```java
package study;

public class BehaviorTest {

    static void check(boolean value, String message) {
        if (!value) throw new AssertionError(message);
    }

    public static void main(String[] args) throws Exception {
        var now = new java.util.concurrent.atomic.AtomicLong();
        var l = new TokenBucket(2, 1, now::get);
        check(l.allow("a").allowed() && l.allow("a").allowed() && !l.allow("a").allowed(), "burst");
        check(l.allow("b").allowed(), "isolation");
        now.set(500_000_000L);
        var d = l.allow("a");
        check(!d.allowed() && d.retryAfter().toNanos() == 500_000_000L, "retry delay");
        now.set(1_000_000_000L);
        check(l.allow("a").allowed(), "refill");
        var concurrent = new TokenBucket(10, 1, () -> 0L);
        var accepted = new java.util.concurrent.atomic.AtomicInteger();
        var threads = new java.util.ArrayList<Thread>();
        for (int i = 0; i<100; i++) {
            var t = new Thread(() -> {
                if (concurrent.allow("x").allowed())accepted.incrementAndGet();
            });
            threads.add(t);
            t.start();
        }
        for (var t:threads)t.join();
        check(accepted.get() == 10, "concurrent spending");
        System.out.println("All behavior tests passed");
    }
}
```

**How it works and why it belongs here:** Boundary tests use virtual time. A real concurrent test verifies atomic debit under contention.

## 5. Patterns and principles: where and why

| Pattern or principle | Concrete location | Reason |
|---|---|---|
| Token bucket algorithm | Allow / allow | Permits a configured burst with a sustained refill rate. |
| Dependency injection | Clock / LongSupplier | A fake monotonic clock gives deterministic boundary tests. |
| Interface boundary | Limiter / RequestLimiter | Middleware can depend on a decision contract instead of a specific algorithm. |
| Encapsulation | bucket map protected by one lock | Concurrent requests cannot spend the same token twice. |

A mutex or synchronized method is a concurrency mechanism, not a GoF design pattern. An enum is a state representation, not automatically the State pattern. Interfaces are justified by interchangeable behavior or a useful boundary; inheritance is not required to demonstrate OOP.

### Applying SOLID without unnecessary abstractions

**Single responsibility:** the main entry point assembles dependencies; the domain service owns state invariants; policy collaborators own the behavior named in the pattern table. The tests exercise behavior through the public operations rather than depending on implementation maps.

**Open/closed:** inspect `Allow / allow` as the primary variation point. Where a policy interface exists, supply a new implementation without changing the state-transition algorithm. Where this example implements a specific data structure, do not claim its algorithm is interchangeable until you deliberately extract that boundary.

**Liskov substitution:** a replacement collaborator must preserve the documented contract, including invalid-input behavior, clock units, ownership rules and callback failure behavior. Merely matching a method signature is not enough.

**Interface segregation and dependency inversion:** interfaces expose the small question the caller needs answered. Constructors receive policies and clocks where tests or alternative behavior benefit. Simple records, enum values and internal containers remain concrete. A dedicated repository interface is useful when persistence is in scope; these samples do not pretend in-memory mutations automatically translate to database transactions.

## 6. Follow the example and the invariants

Capacity 2 and rate 1 token/sec allow two immediate requests for client A. The third is denied with retryAfter=1s. Client B still has its own full bucket. Advancing time by half a second gives A half a token: still denied, retryAfter=0.5s. At one second A is allowed. Refill caps at capacity, so being idle for a day cannot accumulate an unlimited burst. A backwards clock reading is clamped to the prior time, preventing extra refill.

### Verified execution

The complete project above was compiled and its behavior tests executed. The following is actual validation output (paths, timing and identifiers can vary):

```text
allowed: true
allowed: true
allowed: false

All behavior tests passed
```

## 7. Best practices, edge cases and production extensions

Fractional tokens use floating point because they model elapsed capacity, not money. Tests assert accepted/denied outcomes and bounded retry time rather than equality of arbitrary fractional arithmetic. Clock reads, state mutation and decision share a lock; O(1) map work is serialized across keys. Sharding locks can improve throughput while keeping each client's bucket atomic.

Buckets are never evicted in this sample, so unbounded unique keys can exhaust memory. Production eviction must only discard buckets once they are effectively full, otherwise a client could regain a burst by forcing eviction. Separate processes each grant their own capacity; for a shared quota use an atomic shared-store operation with a consistent time source. Decide how storage failure should affect admission. Retry-After is a hint based on current state, not a reservation. Authentication should define the client key; an arbitrary caller-controlled header is not an identity guarantee.

## 8. Presenting this in an interview

Start by agreeing on the scope in section 1. Draw the responsibility flow, identify the state that must remain consistent, and name the operation that owns that invariant. Implement the core model and service, then wire the collaborators in main and run a concrete example. Show at least one rejected or boundary case from the tests. Explain the pattern at the point where it solves a problem, rather than starting with a list of pattern names.

For a distributed follow-up, distinguish thread safety inside this process from coordination across replicas. In-memory objects do not survive restarts. Agree on consistency, failure recovery and storage requirements before replacing them with remote infrastructure.

## 9. Practice next

1. Add a weighted request cost and reject costs greater than capacity.
2. Add bounded idle-bucket cleanup without resetting a depleted bucket.
3. Implement a sliding-window limiter behind the same interface and compare burst behavior.
4. Wrap it in HTTP middleware returning 429 and Retry-After.
