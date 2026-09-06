# Parking Lot in Java — complete LLD walkthrough

Study order: requirements → model and flow → project tree → each complete file and explanation → patterns → walkthrough → limitations and exercises. This is a runnable single-process interview reference, with no external services or omitted source files.

## 1. Requirements and scope

Park cars or motorcycles, allocate a compatible free spot, issue a ticket, and free the spot on exit. Calculate fees using a replaceable policy. Reject duplicate entry, unknown tickets, time-travelling exits and a full lot. Money uses integer minor units. A motorcycle may use a car spot only when no motorcycle spot is free. Payment processing and physical gates are outside scope.

## 2. Design and responsibilities

```text
Main -> ParkingLot -> compatible spot allocation
                   -> active ticket map + occupied spot map (one lock)
                   -> FeePolicy on exit
Vehicle -> Ticket -> Spot
```

## 3. Project structure and running

Create each file at the shown relative path. All imports and package declarations are included.

```text
parking-lot-java/
  src/study/Kind.java
  src/study/Vehicle.java
  src/study/Spot.java
  src/study/Ticket.java
  src/study/FeePolicy.java
  src/study/HourlyFee.java
  src/study/ParkingLot.java
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

### 4.1. `src/study/Kind.java`

```java
package study;

enum Kind {
    MOTORCYCLE, CAR
}
```

**How it works and why it belongs here:** Enum values constrain vehicle/spot categories.

### 4.2. `src/study/Vehicle.java`

```java
package study;

record Vehicle(String plate, Kind kind) {
}
```

**How it works and why it belongs here:** Vehicle is an immutable request value.

### 4.3. `src/study/Spot.java`

```java
package study;

record Spot(int id, Kind kind) {
}
```

**How it works and why it belongs here:** Spot defines a physical resource; occupancy is managed by ParkingLot.

### 4.4. `src/study/Ticket.java`

```java
package study;

import java.time.Instant;

record Ticket(int id, String plate, int spotId, Instant entered) {
}
```

**How it works and why it belongs here:** An immutable ticket prevents callers from changing resource ownership.

### 4.5. `src/study/FeePolicy.java`

```java
package study;

import java.time.Duration;

interface FeePolicy {
    long fee(Duration duration);
}
```

**How it works and why it belongs here:** Strategy exposes the variable pricing algorithm.

### 4.6. `src/study/HourlyFee.java`

```java
package study;

import java.time.Duration;

final class HourlyFee implements FeePolicy {
    private final long rate;
    HourlyFee(long rate) {
        if (rate<0) throw new IllegalArgumentException("rate");
        this.rate = rate;
    }

    public long fee(Duration d) {
        if (d.isNegative()) throw new IllegalArgumentException("negative duration");
        long hours = d.toHours();
        if (!d.minusHours(hours).isZero())hours++;
        return Math.multiplyExact(hours, rate);
    }
}
```

**How it works and why it belongs here:** Rounding is based on Duration, including sub-hour fractions. multiplyExact rejects overflow.

### 4.7. `src/study/ParkingLot.java`

```java
package study;

import java.util.Map;
import java.util.List;
import java.util.Set;
import java.util.HashMap;
import java.util.HashSet;
import java.util.Comparator;
import java.util.Objects;
import java.time.Instant;

final class ParkingLot {
    private final List<Spot> spots;
    private final Set<Integer> occupied = new HashSet<>();
    private final Map<Integer, Ticket> tickets = new HashMap<>();
    private final Set<String> plates = new HashSet<>();
    private final FeePolicy fee;
    private int next;
    ParkingLot(List<Spot> spots, FeePolicy fee) {
        this.fee = Objects.requireNonNull(fee);
        this.spots = spots.stream().sorted(Comparator.comparingInt(Spot::id)).toList();
        var ids = new HashSet<Integer>();
        for (var s:this.spots)if (s.kind() == null || !ids.add(s.id())) throw new IllegalArgumentException("spot");
    }

    synchronized Ticket park(Vehicle v, Instant at) {
        Objects.requireNonNull(at);
        if (v.plate() == null || v.plate().isBlank() || v.kind() == null || plates.contains(v.plate())) throw new IllegalArgumentException("vehicle");
        for (var kind:List.of(v.kind(), Kind.CAR))for (var s:spots) {
            if (s.kind() != kind || occupied.contains(s.id()))continue;
            var t = new Ticket(++next, v.plate(), s.id(), at);
            occupied.add(s.id());
            plates.add(v.plate());
            tickets.put(t.id(), t);
            return t;
        }
        throw new IllegalStateException("full");
    }

    synchronized long exit(int id, Instant at) {
        var t = tickets.get(id);
        if (t == null) throw new IllegalArgumentException("ticket");
        long amount = fee.fee(java.time.Duration.between(t.entered(), at));
        tickets.remove(id);
        plates.remove(t.plate());
        occupied.remove(t.spotId());
        return amount;
    }
}
```

**How it works and why it belongs here:** synchronized protects the whole allocation transaction. Immutable records can safely be returned. The policy is injected and no I/O occurs while computing the fee.

### 4.8. `src/study/Main.java`

```java
package study;

public class Main {

    public static void main(String[]args) {
        var lot = new ParkingLot(java.util.List.of(new Spot(1, Kind.CAR)), new HourlyFee(100));
        var now = java.time.Instant.EPOCH;
        var t = lot.park(new Vehicle("AB123", Kind.CAR), now);
        System.out.println("Fee: "+lot.exit(t.id(), now.plusSeconds(3660)));
    }
}
```

**How it works and why it belongs here:** The composition root creates the collaborators explicitly and runs the example. The public entry point lives in its own file; domain collaborators remain package-private within study.

### 4.9. `src/study/BehaviorTest.java`

```java
package study;

public class BehaviorTest {

    static void check(boolean value, String message) {
        if (!value) throw new AssertionError(message);
    }

    public static void main(String[] args) throws Exception {
        var lot = new ParkingLot(java.util.List.of(new Spot(1, Kind.CAR)), new HourlyFee(100));
        var now = java.time.Instant.EPOCH;
        var a = lot.park(new Vehicle("A", Kind.CAR), now);
        boolean full = false;
        try {
            lot.park(new Vehicle("B", Kind.CAR), now);
        }
        catch (IllegalStateException e) {
            full = true;
        }
        check(full, "overbooked");
        boolean invalid = false;
        try {
            lot.exit(a.id(), now.minusSeconds(1));
        }
        catch (IllegalArgumentException e) {
            invalid = true;
        }
        check(invalid, "negative stay");
        check(lot.exit(a.id(), now.plusSeconds(3660)) == 200, "rounding");
        lot.park(new Vehicle("B", Kind.CAR), now);
        {
            var single = new ParkingLot(java.util.List.of(new Spot(1, Kind.CAR)), new HourlyFee(100));
            var wins = new java.util.concurrent.atomic.AtomicInteger();
            var racers = new java.util.ArrayList<Thread>();
            for (String plate:java.util.List.of("A", "B")) {
                var thread = new Thread(() -> {
                    try {
                        single.park(new Vehicle(plate, Kind.CAR), java.time.Instant.EPOCH);
                        wins.incrementAndGet();
                    }
                    catch (IllegalStateException expected) {
                    }
                });
                racers.add(thread);
                thread.start();
            }
            for (var thread:racers)thread.join();
            check(wins.get() == 1, "one spot allocated twice");
            var mixed = new ParkingLot(java.util.List.of(new Spot(1, Kind.CAR), new Spot(2, Kind.MOTORCYCLE)), new HourlyFee(100));
            check(mixed.park(new Vehicle("M", Kind.MOTORCYCLE), java.time.Instant.EPOCH).spotId() == 2, "best fit");
            mixed.park(new Vehicle("C", Kind.CAR), java.time.Instant.EPOCH);
        }
        System.out.println("All behavior tests passed");
    }
}
```

**How it works and why it belongs here:** Behavior checks validate full-lot rejection, failed-exit consistency, pricing and spot reuse. Additional tests race for one physical spot and verify motorcycles prefer motorcycle spots.

## 5. Patterns and principles: where and why

| Pattern or principle | Concrete location | Reason |
|---|---|---|
| Strategy | FeePolicy / HourlyFee | Change pricing without rewriting allocation or exit. |
| Dependency injection | ParkingLot constructor in Main | Pricing is supplied explicitly. |
| Encapsulation | ParkingLot.Park/Exit or park/exit | Allocation and ticket state change together. |
| Composition | ParkingLot contains spots and policy | The lot has resources and behavior collaborators; subclassing a lot is unnecessary. |

A mutex or synchronized method is a concurrency mechanism, not a GoF design pattern. An enum is a state representation, not automatically the State pattern. Interfaces are justified by interchangeable behavior or a useful boundary; inheritance is not required to demonstrate OOP.

### Applying SOLID without unnecessary abstractions

**Single responsibility:** the main entry point assembles dependencies; the domain service owns state invariants; policy collaborators own the behavior named in the pattern table. The tests exercise behavior through the public operations rather than depending on implementation maps.

**Open/closed:** inspect `FeePolicy / HourlyFee` as the primary variation point. Where a policy interface exists, supply a new implementation without changing the state-transition algorithm. Where this example implements a specific data structure, do not claim its algorithm is interchangeable until you deliberately extract that boundary.

**Liskov substitution:** a replacement collaborator must preserve the documented contract, including invalid-input behavior, clock units, ownership rules and callback failure behavior. Merely matching a method signature is not enough.

**Interface segregation and dependency inversion:** interfaces expose the small question the caller needs answered. Constructors receive policies and clocks where tests or alternative behavior benefit. Simple records, enum values and internal containers remain concrete. A dedicated repository interface is useful when persistence is in scope; these samples do not pretend in-memory mutations automatically translate to database transactions.

## 6. Follow the example and the invariants

Entry scans spot IDs deterministically, preferring the smallest compatible type. Allocation, vehicle registration and ticket creation happen under one lock. No two active tickets can own one spot, and one plate cannot be parked twice. Exit computes the fee before releasing resources, so an invalid timestamp does not lose the ticket. A 61-minute stay at 100 minor units per started hour costs 200. A zero-duration stay is free in this agreed policy. The demo parks a car, exits after 61 minutes and prints the fee.

### Verified execution

The complete project above was compiled and its behavior tests executed. The following is actual validation output (paths, timing and identifiers can vary):

```text
Fee: 200

All behavior tests passed
```

## 7. Best practices, edge cases and production extensions

A single lock intentionally protects the invariant spanning three maps. This is simple and sufficient for an interview, but scans are O(S) for S spots. Separate ordered free-spot indexes can improve allocation while retaining atomicity. Tickets are returned as copies/immutable records; callers cannot rewrite occupancy. The fee strategy executes under the lock and must be fast and side-effect-free; do not call a payment network there.

For production, persist ticket ownership and use database constraints/transactions. Add a payment state machine: ACTIVE -> PAYMENT_PENDING -> PAID -> CLOSED, with payment idempotency and explicit gate-failure recovery. Do not release the spot merely because a network request was attempted. A real vehicle identity may require jurisdiction plus plate. The demo has no reservations, levels, disabled/EV spots, or concurrent payment integration. Very large money values require overflow limits.

## 8. Presenting this in an interview

Start by agreeing on the scope in section 1. Draw the responsibility flow, identify the state that must remain consistent, and name the operation that owns that invariant. Implement the core model and service, then wire the collaborators in main and run a concrete example. Show at least one rejected or boundary case from the tests. Explain the pattern at the point where it solves a problem, rather than starting with a list of pattern names.

For a distributed follow-up, distinguish thread safety inside this process from coordination across replicas. In-memory objects do not survive restarts. Agree on consistency, failure recovery and storage requirements before replacing them with remote infrastructure.

## 9. Practice next

1. Add EV-compatible spots without breaking normal cars.
2. Inject weekend pricing and test it independently.
3. Add a free-spot index and preserve deterministic allocation.
4. Model payment failure and duplicate exit requests.
