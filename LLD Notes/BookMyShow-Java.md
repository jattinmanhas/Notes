# BookMyShow in Java — complete LLD walkthrough

Study order: requirements → model and flow → project tree → each complete file and explanation → patterns → walkthrough → limitations and exercises. This is a runnable single-process interview reference, with no external services or omitted source files.

## 1. Requirements and scope

Implement seat inventory for one show: list available seats, atomically hold a group of seats for a user, confirm before expiry, or cancel a hold. Holds have a TTL and owner. Reject duplicate seats, unknown seats, already held/booked seats and confirmation by another user. A confirmed hold stays booked; repeat confirmation by its owner is idempotent. Search, theaters, show scheduling and actual payment are outside scope.

## 2. Design and responsibilities

```text
Main -> BookingService -> show seat inventory
                      -> Hold(id,owner,seats,expiry,state)
Clock -> lazy hold expiry
AVAILABLE -> HELD -> BOOKED
              \-> EXPIRED/CANCELLED -> AVAILABLE
```

## 3. Project structure and running

Create each file at the shown relative path. All imports and package declarations are included.

```text
bookmyshow-java/
  src/study/HoldStatus.java
  src/study/Hold.java
  src/study/BookingService.java
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

### 4.1. `src/study/HoldStatus.java`

```java
package study;

enum HoldStatus {
    HELD, BOOKED, EXPIRED, CANCELLED
}
```

**How it works and why it belongs here:** Explicit states prevent an expired reservation from being confirmed.

### 4.2. `src/study/Hold.java`

```java
package study;

import java.time.Instant;
import java.util.List;

record Hold(int id, String owner, List<String> seats, Instant expires, HoldStatus status) {
    Hold {
        seats = List.copyOf(seats);
    }

    Hold withStatus(HoldStatus s) {
        return new Hold(id, owner, seats, expires, s);
    }
}
```

**How it works and why it belongs here:** The compact constructor copies seat lists; withStatus creates a replacement immutable record.

### 4.3. `src/study/BookingService.java`

```java
package study;

import java.util.Map;
import java.util.List;
import java.util.HashMap;
import java.util.HashSet;
import java.util.Objects;
import java.time.*;
import java.util.function.Supplier;

final class BookingService {
    private final Map<String, Integer> seats = new HashMap<>();
    private final Map<Integer, Hold> holds = new HashMap<>();
    private int next;
    private final Supplier<Instant> clock;
    BookingService(List<String> ids, Supplier<Instant> clock) {
        this.clock = Objects.requireNonNull(clock);
        for (String id:ids)if (id == null || id.isBlank() || seats.putIfAbsent(id, 0) != null) throw new IllegalArgumentException("seat");
    }

    private void expire() {
        Instant now = clock.get();
        for (var entry:holds.entrySet()) {
            var h = entry.getValue();
            if (h.status() == HoldStatus.HELD && !now.isBefore(h.expires())) {
                entry.setValue(h.withStatus(HoldStatus.EXPIRED));
                h.seats().forEach(s -> seats.put(s, 0));
            }
        }
    }

    synchronized Hold hold(String owner, List<String> requested, Duration ttl) {
        expire();
        if (owner == null || owner.isBlank() || requested.isEmpty() || ttl.isNegative() || ttl.isZero()) throw new IllegalArgumentException("request");
        var seen = new HashSet<String>();
        for (String seat:requested)if (!seats.containsKey(seat) || seats.get(seat) != 0 || !seen.add(seat)) throw new IllegalStateException("seat unavailable");
        var h = new Hold(++next, owner, requested, clock.get().plus(ttl), HoldStatus.HELD);
        holds.put(h.id(), h);
        h.seats().forEach(s -> seats.put(s, h.id()));
        return h;
    }

    synchronized void confirm(int id, String owner) {
        expire();
        var h = holds.get(id);
        if (h == null || !h.owner().equals(owner)) throw new IllegalArgumentException("owner/hold");
        if (h.status() == HoldStatus.BOOKED) return;
        if (h.status() != HoldStatus.HELD) throw new IllegalStateException("inactive");
        holds.put(id, h.withStatus(HoldStatus.BOOKED));
    }

    synchronized void cancel(int id, String owner) {
        expire();
        var h = holds.get(id);
        if (h == null || !h.owner().equals(owner) || h.status() != HoldStatus.HELD) throw new IllegalStateException("inactive/owner");
        holds.put(id, h.withStatus(HoldStatus.CANCELLED));
        h.seats().forEach(s -> seats.put(s, 0));
    }

    synchronized List<String> available() {
        expire();
        return seats.entrySet().stream().filter(e -> e.getValue() == 0).map(Map.Entry::getKey).sorted().toList();
    }
}
```

**How it works and why it belongs here:** All seat-group transitions synchronize on one show inventory. Terminal Hold records are replaced instead of mutated. Available returns a detached sorted list.

### 4.4. `src/study/Main.java`

```java
package study;

public class Main {

    public static void main(String[]args) {
        var s = new BookingService(java.util.List.of("A1", "A2", "A3"), java.time.Instant::now);
        var h = s.hold("alice", java.util.List.of("A1", "A2"), java.time.Duration.ofMinutes(1));
        s.confirm(h.id(), "alice");
        System.out.println("Booked hold "+h.id()+" available: "+s.available());
    }
}
```

**How it works and why it belongs here:** The composition root creates the collaborators explicitly and runs the example. The public entry point lives in its own file; domain collaborators remain package-private within study.

### 4.5. `src/study/BehaviorTest.java`

```java
package study;

public class BehaviorTest {

    static void check(boolean value, String message) {
        if (!value) throw new AssertionError(message);
    }

    public static void main(String[] args) throws Exception {
        var now = new java.util.concurrent.atomic.AtomicReference<>(java.time.Instant.EPOCH);
        var s = new BookingService(java.util.List.of("A", "B"), now::get);
        var h = s.hold("u", java.util.List.of("A"), java.time.Duration.ofSeconds(1));
        boolean failed = false;
        try {
            s.hold("v", java.util.List.of("B", "A"), java.time.Duration.ofSeconds(1));
        }
        catch (IllegalStateException e) {
            failed = true;
        }
        check(failed && s.available().equals(java.util.List.of("B")), "partial allocation");
        now.set(now.get().plusSeconds(1));
        failed = false;
        try {
            s.confirm(h.id(), "u");
        }
        catch (IllegalStateException e) {
            failed = true;
        }
        check(failed, "expiry");
        var next = s.hold("v", java.util.List.of("A"), java.time.Duration.ofSeconds(1));
        s.confirm(next.id(), "v");
        s.confirm(next.id(), "v");
        check(s.available().equals(java.util.List.of("B")), "booked availability");
        {
            var inventory = new BookingService(java.util.List.of("X"), java.time.Instant::now);
            var wins = new java.util.concurrent.atomic.AtomicInteger();
            var racers = new java.util.ArrayList<Thread>();
            for (String user:java.util.List.of("x", "y")) {
                var thread = new Thread(() -> {
                    try {
                        inventory.hold(user, java.util.List.of("X"), java.time.Duration.ofHours(1));
                        wins.incrementAndGet();
                    }
                    catch (IllegalStateException expected) {
                    }
                });
                racers.add(thread);
                thread.start();
            }
            for (var thread:racers)thread.join();
            check(wins.get() == 1, "seat acquired twice");
        }
        System.out.println("All behavior tests passed");
    }
}
```

**How it works and why it belongs here:** Validates all-or-nothing allocation, expiry, reuse and repeat confirmation. A contention test proves only one caller can acquire the same seat.

## 5. Patterns and principles: where and why

| Pattern or principle | Concrete location | Reason |
|---|---|---|
| Aggregate boundary | BookingService / ShowInventory | One lock protects a multi-seat hold as an all-or-nothing operation. |
| State-machine modeling | HoldStatus and confirm/cancel/expire | Only allowed transitions can alter availability. |
| Dependency injection | clock in service constructor | Tests can expire reservations without sleeping. |
| Idempotency | confirm on an already booked hold | Repeated confirmation does not allocate twice. |

A mutex or synchronized method is a concurrency mechanism, not a GoF design pattern. An enum is a state representation, not automatically the State pattern. Interfaces are justified by interchangeable behavior or a useful boundary; inheritance is not required to demonstrate OOP.

### Applying SOLID without unnecessary abstractions

**Single responsibility:** the main entry point assembles dependencies; the domain service owns state invariants; policy collaborators own the behavior named in the pattern table. The tests exercise behavior through the public operations rather than depending on implementation maps.

**Open/closed:** inspect `BookingService / ShowInventory` as the primary variation point. Where a policy interface exists, supply a new implementation without changing the state-transition algorithm. Where this example implements a specific data structure, do not claim its algorithm is interchangeable until you deliberately extract that boundary.

**Liskov substitution:** a replacement collaborator must preserve the documented contract, including invalid-input behavior, clock units, ownership rules and callback failure behavior. Merely matching a method signature is not enough.

**Interface segregation and dependency inversion:** interfaces expose the small question the caller needs answered. Constructors receive policies and clocks where tests or alternative behavior benefit. Simple records, enum values and internal containers remain concrete. A dedicated repository interface is useful when persistence is in scope; these samples do not pretend in-memory mutations automatically translate to database transactions.

## 6. Follow the example and the invariants

User A holds seats A1 and A2. Both seats become unavailable in one atomic operation. User B cannot hold A2. At expiry exactly, A's unconfirmed hold releases both seats, permitting B to hold A2. A cannot then confirm the expired hold. A confirmed booking is not expired by the hold cleanup, and only its owner may repeat confirmation. The demo holds A1/A2 and confirms successfully before the clock advances.

### Verified execution

The complete project above was compiled and its behavior tests executed. The following is actual validation output (paths, timing and identifiers can vary):

```text
Booked hold 1 available: [A3]

All behavior tests passed
```

## 7. Best practices, edge cases and production extensions

This intentionally centers the hard seat-ownership invariant. The service owns one show, so seat IDs need no show prefix internally; a larger system partitions inventory by show ID. Expiration scans all holds lazily on each operation, O(H). Production needs cleanup/indexing and archival of terminal holds. Every public operation returns copies or immutable values so a caller cannot change seat ownership.

Do not call a payment provider under the inventory lock. A payment integration requires an idempotent payment attempt, a durable reservation, and a transaction that verifies the reservation is still valid. If payment succeeds after expiry, compensate with a refund rather than stealing a seat from another user. Multi-server seat ownership needs database conditional writes/transactions and a consistent expiry policy. Hold creation is not idempotent here; add a client request key for network retries. Cancellation is for active holds only, not a refund API for booked tickets.

## 8. Presenting this in an interview

Start by agreeing on the scope in section 1. Draw the responsibility flow, identify the state that must remain consistent, and name the operation that owns that invariant. Implement the core model and service, then wire the collaborators in main and run a concrete example. Show at least one rejected or boundary case from the tests. Explain the pattern at the point where it solves a problem, rather than starting with a list of pattern names.

For a distributed follow-up, distinguish thread safety inside this process from coordination across replicas. In-memory objects do not survive restarts. Agree on consistency, failure recovery and storage requirements before replacing them with remote infrastructure.

## 9. Practice next

1. Add show IDs and keep inventory locks independent per show.
2. Add a request key to hold creation and detect conflicting reuse.
3. Model payment success after reservation expiry.
4. Race two callers for overlapping seat groups and assert no partial allocation.
