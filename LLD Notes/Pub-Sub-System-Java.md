# Pub-Sub System in Java — complete LLD walkthrough

Study order: requirements → model and flow → project tree → each complete file and explanation → patterns → walkthrough → limitations and exercises. This is a runnable single-process interview reference, with no external services or omitted source files.

## 1. Requirements and scope

Implement an in-process topic bus: subscribe a handler, publish an immutable string event to every subscriber of that exact topic, and unsubscribe by subscription ID. Preserve subscription order within one publish. Invoke callbacks outside the registry lock so they can subscribe/unsubscribe safely. Collect handler errors and continue fan-out. Delivery is synchronous and best-effort; there is no persistence, retry, replay or consumer group.

## 2. Design and responsibilities

```text
Publisher -> EventBus.Publish(topic,payload)
             -> snapshot matching subscription list under lock
             -> unlock
             -> handler A(event)
             -> handler B(event)
             -> return collected failures
Subscribe/Unsubscribe -> registry only
```

## 3. Project structure and running

Create each file at the shown relative path. All imports and package declarations are included.

```text
pub-sub-java/
  src/study/Event.java
  src/study/Handler.java
  src/study/Subscription.java
  src/study/EventBus.java
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

### 4.1. `src/study/Event.java`

```java
package study;

import java.util.Objects;

record Event(String topic, String payload) {
    Event {
        Objects.requireNonNull(topic);
        Objects.requireNonNull(payload);
    }
}
```

**How it works and why it belongs here:** Immutable string event avoids shared mutable payload surprises.

### 4.2. `src/study/Handler.java`

```java
package study;

interface Handler {
    void onEvent(Event event) throws Exception;
}
```

**How it works and why it belongs here:** A functional interface supports lambdas and class-based subscribers. Checked failures are reported by publish.

### 4.3. `src/study/Subscription.java`

```java
package study;

record Subscription(int id, Handler handler) {
}
```

**How it works and why it belongs here:** Registration identity belongs to the bus, not the callback object.

### 4.4. `src/study/EventBus.java`

```java
package study;

import java.util.Map;
import java.util.List;
import java.util.HashMap;
import java.util.ArrayList;
import java.util.Objects;

final class EventBus {
    private final Map<String, List<Subscription>> topics = new HashMap<>();
    private int next;

    synchronized int subscribe(String topic, Handler handler) {
        if (topic == null || topic.isBlank()) throw new IllegalArgumentException("topic");
        Objects.requireNonNull(handler);
        int id = ++next;
        topics.computeIfAbsent(topic, t -> new ArrayList<>()).add(new Subscription(id, handler));
        return id;
    }

    synchronized boolean unsubscribe(int id) {
        var it = topics.entrySet().iterator();
        while (it.hasNext()) {
            var subs = it.next().getValue();
            if (subs.removeIf(s -> s.id() == id)) {
                if (subs.isEmpty())it.remove();
                return true;
            }
        }
        return false;
    }

    List<Exception> publish(Event event) {
        List<Subscription> snapshot;
        synchronized (this) {
            snapshot = List.copyOf(topics.getOrDefault(event.topic(), List.of()));
        }
        var failures = new ArrayList<Exception>();
        for (var s:snapshot)try {
            s.handler().onEvent(event);
        }
        catch (Exception e) {
            failures.add(e);
        }
        return List.copyOf(failures);
    }
}
```

**How it works and why it belongs here:** publish deliberately is not a synchronized method: only snapshot construction holds the monitor. The returned error list is immutable. Handler calls can reenter the bus without depending on Java monitor reentrancy to mask a poor lock boundary.

### 4.5. `src/study/Main.java`

```java
package study;

public class Main {

    public static void main(String[]args) {
        var b = new EventBus();
        int id = b.subscribe("orders", e -> System.out.println("email: "+e.payload()));
        b.subscribe("orders", e -> System.out.println("audit: "+e.payload()));
        if (!b.publish(new Event("orders", "created")).isEmpty()) throw new IllegalStateException("handler failed");
        b.unsubscribe(id);
        b.publish(new Event("orders", "shipped"));
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
        var b = new EventBus();
        var count = new java.util.concurrent.atomic.AtomicInteger();
        var id = new int[1];
        id[0] = b.subscribe("a", e -> {
            b.unsubscribe(id[0]);
            throw new Exception("failed");
        });
        b.subscribe("a", e -> count.incrementAndGet());
        b.subscribe("b", e -> {
            throw new AssertionError("wrong topic");
        });
        check(b.publish(new Event("a", "one")).size() == 1 && count.get() == 1, "isolation");
        check(b.publish(new Event("a", "two")).isEmpty() && count.get() == 2, "unsubscribe");
        check(!b.unsubscribe(id[0]), "double unsubscribe");
        System.out.println("All behavior tests passed");
    }
}
```

**How it works and why it belongs here:** Verifies topic isolation, failure fan-out, self-unsubscribe and repeated removal behavior.

## 5. Patterns and principles: where and why

| Pattern or principle | Concrete location | Reason |
|---|---|---|
| Observer / publish-subscribe | EventBus + Handler | Publishers are decoupled from concrete subscribers; each matching observer receives the event. |
| Snapshot iteration | Publish copies registrations under lock | Callbacks can change the registry without deadlocking or invalidating iteration. |
| Dependency inversion | Handler contract | The bus depends on behavior rather than subscriber application classes. |
| Fan-out | per-topic subscriber list | Every subscriber receives each matching event; this differs from competing consumers on a work queue. |

A mutex or synchronized method is a concurrency mechanism, not a GoF design pattern. An enum is a state representation, not automatically the State pattern. Interfaces are justified by interchangeable behavior or a useful boundary; inheritance is not required to demonstrate OOP.

### Applying SOLID without unnecessary abstractions

**Single responsibility:** the main entry point assembles dependencies; the domain service owns state invariants; policy collaborators own the behavior named in the pattern table. The tests exercise behavior through the public operations rather than depending on implementation maps.

**Open/closed:** inspect `EventBus + Handler` as the primary variation point. Where a policy interface exists, supply a new implementation without changing the state-transition algorithm. Where this example implements a specific data structure, do not claim its algorithm is interchangeable until you deliberately extract that boundary.

**Liskov substitution:** a replacement collaborator must preserve the documented contract, including invalid-input behavior, clock units, ownership rules and callback failure behavior. Merely matching a method signature is not enough.

**Interface segregation and dependency inversion:** interfaces expose the small question the caller needs answered. Constructors receive policies and clocks where tests or alternative behavior benefit. Simple records, enum values and internal containers remain concrete. A dedicated repository interface is useful when persistence is in scope; these samples do not pretend in-memory mutations automatically translate to database transactions.

## 6. Follow the example and the invariants

Two handlers subscribe to orders. Publishing created invokes both once, in registration order. A subscriber to payments receives nothing. Unsubscribing the first orders registration leaves only the second for the next publish. If a handler unsubscribes itself while processing, the current snapshot still completes, and future snapshots omit it. An unsubscribe racing a publish may not stop an invocation already captured in that publish. A handler failure is returned to the publisher and does not prevent later handlers from running.

### Verified execution

The complete project above was compiled and its behavior tests executed. The following is actual validation output (paths, timing and identifiers can vary):

```text
email: created
audit: created
audit: shipped

All behavior tests passed
```

## 7. Best practices, edge cases and production extensions

The bus is safe for concurrent registry operations, but concurrent publish calls may execute the same handler at the same time. Subscribers must be thread-safe or add their own serialized mailbox. Per-publish registration order is guaranteed; global event ordering across publishers is not. A slow subscriber delays later subscribers and the publisher. Snapshot copying costs O(S) for S matching subscribers; unsubscribe scans registered lists in this small implementation.

Do not label this exactly-once or at-least-once messaging. It invokes each snapshot registration once per publish attempt, without crash recovery or deduplication. Go handlers return expected errors; panics propagate rather than being silently swallowed. Java catches Exception from handlers but not JVM Errors. A durable pub-sub service needs topic storage, subscription offsets, ack/redelivery, retention and dead-letter handling. Consumer groups distribute events among group members; independent groups each receive the stream. Async bounded mailboxes introduce an explicit policy for slow subscribers and shutdown draining.

## 8. Presenting this in an interview

Start by agreeing on the scope in section 1. Draw the responsibility flow, identify the state that must remain consistent, and name the operation that owns that invariant. Implement the core model and service, then wire the collaborators in main and run a concrete example. Show at least one rejected or boundary case from the tests. Explain the pattern at the point where it solves a problem, rather than starting with a list of pattern names.

For a distributed follow-up, distinguish thread safety inside this process from coordination across replicas. In-memory objects do not survive restarts. Agree on consistency, failure recovery and storage requirements before replacing them with remote infrastructure.

## 9. Practice next

1. Add per-subscriber bounded asynchronous mailboxes with an explicit overflow policy.
2. Guarantee per-topic ordering with one dispatcher and explain its throughput cost.
3. Add an envelope ID and subscriber deduplication.
4. Design durable offsets and replay separately from this in-process observer API.
