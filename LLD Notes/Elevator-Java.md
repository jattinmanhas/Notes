# Elevator in Java — complete LLD walkthrough

Study order: requirements → model and flow → project tree → each complete file and explanation → patterns → walkthrough → limitations and exercises. This is a runnable single-process interview reference, with no external services or omitted source files.

## 1. Requirements and scope

Simulate one elevator over a bounded set of floors. Accept floor requests, coalesce duplicates, and move one floor per tick. Doors open for one tick at requested floors; movement occurs only with doors closed. Use a replaceable stop-selection policy. This is a deterministic simulator, not physical elevator control. Multiple-car assignment, hall-call direction, emergency operation and real hardware interlocks are outside scope.

## 2. Design and responsibilities

```text
Main -> Elevator.Request -> pending stop set
     -> Elevator.Tick -> StopPolicy.Next
                        -> close doors OR move one floor OR open at stop
State: current floor + direction + doors + pending stops
```

## 3. Project structure and running

Create each file at the shown relative path. All imports and package declarations are included.

```text
elevator-java/
  src/study/StopPolicy.java
  src/study/SweepPolicy.java
  src/study/Snapshot.java
  src/study/Elevator.java
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

### 4.1. `src/study/StopPolicy.java`

```java
package study;

import java.util.Set;

interface StopPolicy {
    Integer next(int floor, int direction, Set<Integer> stops);
}
```

**How it works and why it belongs here:** A null result means no work; the scheduler is independent of mutable elevator state.

### 4.2. `src/study/SweepPolicy.java`

```java
package study;

import java.util.Set;

final class SweepPolicy implements StopPolicy {

    public Integer next(int floor, int direction, Set<Integer> stops) {
        if (stops.contains(floor)) return floor;
        int d = direction == 0?1:direction;
        for (int dir:new int[] {
            d, -d
        }
        ) {
            Integer target = null;
            for (int s:stops)if ((s-floor)*dir>0 && (target == null || (s-target)*dir<0))target = s;
            if (target != null) return target;
        }
        return null;
    }
}
```

**How it works and why it belongs here:** Two directional scans implement a simple sweep policy. It serves the current floor immediately.

### 4.3. `src/study/Snapshot.java`

```java
package study;

record Snapshot(int floor, int direction, boolean doorsOpen, int pending) {
}
```

**How it works and why it belongs here:** Immutable output records let callers inspect state without mutating the model.

### 4.4. `src/study/Elevator.java`

```java
package study;

import java.util.Set;
import java.util.HashSet;
import java.util.Objects;

final class Elevator {
    private int floor, direction;
    private final int max;
    private boolean open;
    private final Set<Integer> stops = new HashSet<>();
    private final StopPolicy policy;
    Elevator(int max, StopPolicy policy) {
        if (max<0) throw new IllegalArgumentException("max");
        this.max = max;
        this.policy = Objects.requireNonNull(policy);
    }

    synchronized void request(int f) {
        if (f<0 || f> max) throw new IllegalArgumentException("floor");
        stops.add(f);
    }

    synchronized Snapshot tick() {
        if (open) {
            open = false;
            return snapshot();
        }
        Integer target = policy.next(floor, direction, Set.copyOf(stops));
        if (target == null) {
            direction = 0;
            return snapshot();
        }
        if (!stops.contains(target)) throw new IllegalStateException("policy returned invalid target");
        if (target> floor) {
            direction = 1;
            floor++;
        }
        else if (target<floor) {
            direction = -1;
            floor--;
        }
        if (stops.remove(floor))open = true;
        return snapshot();
    }

    private Snapshot snapshot() {
        return new Snapshot(floor, direction, open, stops.size());
    }
}
```

**How it works and why it belongs here:** synchronized request/tick protect the model. The immutable set supplied to the policy enforces separation of responsibility. One tick never closes doors and moves simultaneously.

### 4.5. `src/study/Main.java`

```java
package study;

public class Main {

    public static void main(String[]args) {
        var e = new Elevator(10, new SweepPolicy());
        e.request(3);
        e.request(1);
        for (int i = 0; i<6; i++)System.out.println("tick "+i+": "+e.tick());
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
        var e = new Elevator(5, new SweepPolicy());
        e.request(3);
        e.request(1);
        e.request(1);
        var a = e.tick();
        check(a.floor() == 1 && a.doorsOpen() && a.pending() == 1, "first stop");
        var b = e.tick();
        check(b.floor() == 1 && !b.doorsOpen(), "doors and movement");
        e.tick();
        check(e.tick().floor() == 3, "sweep");
        boolean invalid = false;
        try {
            e.request(6);
        }
        catch (IllegalArgumentException x) {
            invalid = true;
        }
        check(invalid, "range");
        System.out.println("All behavior tests passed");
    }
}
```

**How it works and why it belongs here:** Checks duplicate coalescing, sweep order, door sequencing and floor bounds.

## 5. Patterns and principles: where and why

| Pattern or principle | Concrete location | Reason |
|---|---|---|
| Strategy | StopPolicy / SweepPolicy | Change scheduling independently of the elevator state machine. |
| State-machine modeling | Tick / tick and door flag | Enforces door/movement sequence without creating a class per state. |
| Dependency injection | Elevator constructor | The scheduler is supplied by Main. |

A mutex or synchronized method is a concurrency mechanism, not a GoF design pattern. An enum is a state representation, not automatically the State pattern. Interfaces are justified by interchangeable behavior or a useful boundary; inheritance is not required to demonstrate OOP.

### Applying SOLID without unnecessary abstractions

**Single responsibility:** the main entry point assembles dependencies; the domain service owns state invariants; policy collaborators own the behavior named in the pattern table. The tests exercise behavior through the public operations rather than depending on implementation maps.

**Open/closed:** inspect `StopPolicy / SweepPolicy` as the primary variation point. Where a policy interface exists, supply a new implementation without changing the state-transition algorithm. Where this example implements a specific data structure, do not claim its algorithm is interchangeable until you deliberately extract that boundary.

**Liskov substitution:** a replacement collaborator must preserve the documented contract, including invalid-input behavior, clock units, ownership rules and callback failure behavior. Merely matching a method signature is not enough.

**Interface segregation and dependency inversion:** interfaces expose the small question the caller needs answered. Constructors receive policies and clocks where tests or alternative behavior benefit. Simple records, enum values and internal containers remain concrete. A dedicated repository interface is useful when persistence is in scope; these samples do not pretend in-memory mutations automatically translate to database transactions.

## 6. Follow the example and the invariants

The sweep policy serves requests in the current direction first, then reverses when none remain ahead. Duplicate requests occupy one set entry. Starting at floor 0, requests for 3 and 1 cause stops at 1 then 3. Arrival removes that stop and opens the doors. The following tick closes doors without moving, making the safety invariant visible. An idle elevator chooses up when work exists above, otherwise down. A request at the current floor is served before movement.

### Verified execution

The complete project above was compiled and its behavior tests executed. The following is actual validation output (paths, timing and identifiers can vary):

```text
tick 0: Snapshot[floor=1, direction=1, doorsOpen=true, pending=1]
tick 1: Snapshot[floor=1, direction=1, doorsOpen=false, pending=1]
tick 2: Snapshot[floor=2, direction=1, doorsOpen=false, pending=1]
tick 3: Snapshot[floor=3, direction=1, doorsOpen=true, pending=0]
tick 4: Snapshot[floor=3, direction=1, doorsOpen=false, pending=0]
tick 5: Snapshot[floor=3, direction=0, doorsOpen=false, pending=0]

All behavior tests passed
```

## 7. Best practices, edge cases and production extensions

The tick boundary keeps simulation deterministic and testable; wall-clock sleeps and background threads are unnecessary. Calls are synchronized so concurrent request insertion cannot corrupt pending stops. The scheduler receives a copy, cannot mutate elevator state, and must be fast and deterministic. The loop evaluates requests each tick, so repeated requests can affect fairness; this is not a starvation-proof group controller.

Selection scans N pending floors in O(N). A sorted stop set can improve lookup. A fleet design introduces an AssignmentPolicy that selects a car using distance, direction, capacity and load; do not confuse selecting a car with selecting its next stop. Real control requires independently validated safety controllers, obstruction sensors and fault states. This code is suitable only for interview simulation. Tests prove model properties, not physical safety.

## 8. Presenting this in an interview

Start by agreeing on the scope in section 1. Draw the responsibility flow, identify the state that must remain consistent, and name the operation that owns that invariant. Implement the core model and service, then wire the collaborators in main and run a concrete example. Show at least one rejected or boundary case from the tests. Explain the pattern at the point where it solves a problem, rather than starting with a list of pattern names.

For a distributed follow-up, distinguish thread safety inside this process from coordination across replicas. In-memory objects do not survive restarts. Agree on consistency, failure recovery and storage requirements before replacing them with remote infrastructure.

## 9. Practice next

1. Add hall calls with requested direction and distinguish them from cabin calls.
2. Add a fleet dispatcher without mixing car assignment into SweepPolicy.
3. Add emergency and maintenance states with explicit permitted transitions.
4. Test fairness when new requests continually appear ahead.
