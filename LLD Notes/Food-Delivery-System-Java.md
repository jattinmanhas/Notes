# Food Delivery System in Java — complete LLD walkthrough

Study order: requirements → model and flow → project tree → each complete file and explanation → patterns → walkthrough → limitations and exercises. This is a runnable single-process interview reference, with no external services or omitted source files.

## 1. Requirements and scope

Place orders at one restaurant using menu items and quantities, snapshot prices, reserve stock, and advance through PLACED -> ACCEPTED -> READY -> OUT_FOR_DELIVERY -> DELIVERED. Cancel only before READY and restore reserved stock. Assign a free driver when ready and release them after delivery. Choose a driver through a replaceable policy. Payment, multiple restaurants, live GPS and external notifications are outside scope.

## 2. Design and responsibilities

```text
Main -> FoodService.Place -> validate menu/quantity -> reserve stock -> Order
     -> Accept -> Ready -> Dispatch -> DriverPolicy selects free driver
     -> Deliver -> release driver
     -> Cancel (PLACED/ACCEPTED only) -> restore stock
One service lock protects order, inventory and driver ownership
```

## 3. Project structure and running

Create each file at the shown relative path. All imports and package declarations are included.

```text
food-delivery-java/
  src/study/Status.java
  src/study/MenuItem.java
  src/study/Order.java
  src/study/DriverPolicy.java
  src/study/FirstAvailable.java
  src/study/FoodService.java
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

### 4.1. `src/study/Status.java`

```java
package study;

enum Status {
    PLACED, ACCEPTED, READY, OUT_FOR_DELIVERY, DELIVERED, CANCELLED
}
```

**How it works and why it belongs here:** Enum makes permitted transitions explicit.

### 4.2. `src/study/MenuItem.java`

```java
package study;

record MenuItem(long price, int stock) {
}
```

**How it works and why it belongs here:** Immutable menu values are replaced when stock changes.

### 4.3. `src/study/Order.java`

```java
package study;

import java.util.Map;

record Order(int id, Map<String, Integer> items, long total, Status status, String driver) {
    Order {
        items = Map.copyOf(items);
    }

    Order withStatus(Status s, String d) {
        return new Order(id, items, total, s, d);
    }
}
```

**How it works and why it belongs here:** Order is an immutable snapshot. State changes create replacement records; callers cannot change quantities.

### 4.4. `src/study/DriverPolicy.java`

```java
package study;

import java.util.List;

interface DriverPolicy {
    String choose(List<String> free);
}
```

**How it works and why it belongs here:** Strategy chooses an ID or null when no driver is available.

### 4.5. `src/study/FirstAvailable.java`

```java
package study;

import java.util.List;

final class FirstAvailable implements DriverPolicy {

    public String choose(List<String> free) {
        return free.stream().sorted().findFirst().orElse(null);
    }
}
```

**How it works and why it belongs here:** Deterministic selection makes tests stable; it does not claim geographic optimality.

### 4.6. `src/study/FoodService.java`

```java
package study;

import java.util.Map;
import java.util.List;
import java.util.HashMap;
import java.util.Objects;

final class FoodService {
    private final Map<String, MenuItem> menu = new HashMap<>();
    private final Map<String, Boolean> drivers = new HashMap<>();
    private final Map<Integer, Order> orders = new HashMap<>();
    private final DriverPolicy policy;
    private int next;
    FoodService(Map<String, MenuItem> menu, List<String> drivers, DriverPolicy policy) {
        this.policy = Objects.requireNonNull(policy);
        menu.forEach((id, m) -> {
            if (id.isBlank() || m.price()<0 || m.price()>1_000_000_000L || m.stock()<0) throw new IllegalArgumentException("menu");
            this.menu.put(id, m);
        });
        for (String id:drivers)if (id.isBlank() || this.drivers.putIfAbsent(id, false) != null) throw new IllegalArgumentException("driver");
    }

    synchronized Order place(Map<String, Integer> items) {
        if (items.isEmpty()) throw new IllegalArgumentException("empty order");
        long total = 0;
        for (var e:items.entrySet()) {
            var m = menu.get(e.getKey());
            int q = e.getValue();
            if (m == null || q <= 0 || q>10000 || q> m.stock()) throw new IllegalArgumentException("item");
            total = Math.addExact(total, Math.multiplyExact(m.price(), q));
        }
        var o = new Order(++next, items, total, Status.PLACED, null);
        items.forEach((id, q) -> {
            var m = menu.get(id);
            menu.put(id, new MenuItem(m.price(), m.stock()-q));
        });
        orders.put(o.id(), o);
        return o;
    }

    private void transition(int id, Status from, Status to) {
        var o = orders.get(id);
        if (o == null || o.status() != from) throw new IllegalStateException("transition");
        orders.put(id, o.withStatus(to, o.driver()));
    }

    synchronized void accept(int id) {
        transition(id, Status.PLACED, Status.ACCEPTED);
    }

    synchronized void ready(int id) {
        transition(id, Status.ACCEPTED, Status.READY);
    }

    synchronized void dispatch(int id) {
        var o = orders.get(id);
        if (o == null || o.status() != Status.READY) throw new IllegalStateException("not ready");
        String d = policy.choose(drivers.entrySet().stream().filter(e -> !e.getValue()).map(Map.Entry::getKey).toList());
        if (d == null || !drivers.containsKey(d) || drivers.get(d)) throw new IllegalStateException("driver");
        drivers.put(d, true);
        orders.put(id, o.withStatus(Status.OUT_FOR_DELIVERY, d));
    }

    synchronized void deliver(int id) {
        var o = orders.get(id);
        if (o == null || o.status() != Status.OUT_FOR_DELIVERY) throw new IllegalStateException("not out");
        orders.put(id, o.withStatus(Status.DELIVERED, o.driver()));
        drivers.put(o.driver(), false);
    }

    synchronized void cancel(int id) {
        var o = orders.get(id);
        if (o == null || (o.status() != Status.PLACED && o.status() != Status.ACCEPTED)) throw new IllegalStateException("cannot cancel");
        o.items().forEach((key, q) -> {
            var m = menu.get(key);
            menu.put(key, new MenuItem(m.price(), m.stock()+q));
        });
        orders.put(id, o.withStatus(Status.CANCELLED, null));
    }

    synchronized Order get(int id) {
        var o = orders.get(id);
        if (o == null) throw new IllegalArgumentException("order");
        return o;
    }
}
```

**How it works and why it belongs here:** Immutable records plus synchronized mutation methods protect stock/order/driver consistency. Price multiplication and accumulation use checked arithmetic. The service coordinates a single restaurant in one process.

### 4.7. `src/study/Main.java`

```java
package study;

public class Main {

    public static void main(String[]args) {
        var s = new FoodService(java.util.Map.of("burger", new MenuItem(500, 2)), java.util.List.of("driver1"), new FirstAvailable());
        var o = s.place(java.util.Map.of("burger", 1));
        s.accept(o.id());
        s.ready(o.id());
        s.dispatch(o.id());
        s.deliver(o.id());
        System.out.println("Order "+s.get(o.id()));
    }
}
```

**How it works and why it belongs here:** The composition root creates the collaborators explicitly and runs the example. The public entry point lives in its own file; domain collaborators remain package-private within study.

### 4.8. `src/study/BehaviorTest.java`

```java
package study;

public class BehaviorTest {

    static void check(boolean value, String message) {
        if (!value) throw new AssertionError(message);
    }

    public static void main(String[] args) throws Exception {
        var s = new FoodService(java.util.Map.of("x", new MenuItem(100, 2)), java.util.List.of("d"), new FirstAvailable());
        var a = s.place(java.util.Map.of("x", 1));
        boolean rejected = false;
        try {
            s.deliver(a.id());
        }
        catch (IllegalStateException e) {
            rejected = true;
        }
        check(rejected, "illegal jump");
        s.cancel(a.id());
        rejected = false;
        try {
            s.cancel(a.id());
        }
        catch (IllegalStateException e) {
            rejected = true;
        }
        check(rejected, "double cancel");
        var b = s.place(java.util.Map.of("x", 2));
        s.accept(b.id());
        s.ready(b.id());
        s.dispatch(b.id());
        s.deliver(b.id());
        check(s.get(b.id()).total() == 200 && s.get(b.id()).status() == Status.DELIVERED, "lifecycle");
        {
            var fleet = new FoodService(java.util.Map.of("x", new MenuItem(100, 2)), java.util.List.of("d"), new FirstAvailable());
            var first = fleet.place(java.util.Map.of("x", 1));
            var second = fleet.place(java.util.Map.of("x", 1));
            for (int id:java.util.List.of(first.id(), second.id())) {
                fleet.accept(id);
                fleet.ready(id);
            }
            fleet.dispatch(first.id());
            boolean busy = false;
            try {
                fleet.dispatch(second.id());
            }
            catch (IllegalStateException expected) {
                busy = true;
            }
            check(busy, "busy driver assigned twice");
            fleet.deliver(first.id());
            fleet.dispatch(second.id());
        }
        System.out.println("All behavior tests passed");
    }
}
```

**How it works and why it belongs here:** Tests transition guards, cancellation consistency, stock restoration and completion. An additional test prevents assigning a busy driver twice and verifies delivery releases the driver.

## 5. Patterns and principles: where and why

| Pattern or principle | Concrete location | Reason |
|---|---|---|
| Strategy | DriverPolicy / FirstAvailable | Driver selection changes without changing order transitions. |
| State-machine modeling | Accept/Ready/Dispatch/Deliver/Cancel | Rejects impossible lifecycle jumps. |
| Aggregate coordination | FoodService | Inventory and driver ownership updates happen with order state updates. |
| Snapshot/value object | Order price total and copied item quantities | Later menu changes cannot change a placed order total. |

A mutex or synchronized method is a concurrency mechanism, not a GoF design pattern. An enum is a state representation, not automatically the State pattern. Interfaces are justified by interchangeable behavior or a useful boundary; inheritance is not required to demonstrate OOP.

### Applying SOLID without unnecessary abstractions

**Single responsibility:** the main entry point assembles dependencies; the domain service owns state invariants; policy collaborators own the behavior named in the pattern table. The tests exercise behavior through the public operations rather than depending on implementation maps.

**Open/closed:** inspect `DriverPolicy / FirstAvailable` as the primary variation point. Where a policy interface exists, supply a new implementation without changing the state-transition algorithm. Where this example implements a specific data structure, do not claim its algorithm is interchangeable until you deliberately extract that boundary.

**Liskov substitution:** a replacement collaborator must preserve the documented contract, including invalid-input behavior, clock units, ownership rules and callback failure behavior. Merely matching a method signature is not enough.

**Interface segregation and dependency inversion:** interfaces expose the small question the caller needs answered. Constructors receive policies and clocks where tests or alternative behavior benefit. Simple records, enum values and internal containers remain concrete. A dedicated repository interface is useful when persistence is in scope; these samples do not pretend in-memory mutations automatically translate to database transactions.

## 6. Follow the example and the invariants

The menu contains two burgers in stock at 500 minor units each. Placing one burger reserves one and snapshots total=500. The restaurant accepts and marks it ready. Dispatch selects the lexicographically first free driver, marks the driver busy and the order out for delivery atomically. Deliver marks the order delivered and frees that driver. If an order is cancelled while placed/accepted, quantities return to stock exactly once; repeating cancellation returns an error rather than restocking twice.

### Verified execution

The complete project above was compiled and its behavior tests executed. The following is actual validation output (paths, timing and identifiers can vary):

```text
Order Order[id=1, items={burger=1}, total=500, status=DELIVERED, driver=driver1]

All behavior tests passed
```

## 7. Best practices, edge cases and production extensions

No external calls occur under the service lock. FirstAvailable is intentionally deterministic, not a geographic optimization algorithm. A richer assignment strategy needs location, capacity and restaurant pickup readiness, and must still atomically claim the chosen driver. Mutable maps are copied on input/output; menu stock stays service-owned. Quantity validation precedes reservation, so a bad item cannot partially consume another item's stock.

The example bounds item price and quantity to keep illustrative totals within signed 64-bit range for ordinary menu sizes; production needs overflow checking on accumulated totals. Lifecycle endpoints are not generally idempotent and have no authenticated actor checks. Production adds idempotency keys, restaurant/customer/driver authorization and immutable event history. Across restaurant, payment and courier services there is no single in-memory transaction: persist workflow states and use compensating actions such as refund/release. A distributed saga is an extension, not a pattern already implemented here.

## 8. Presenting this in an interview

Start by agreeing on the scope in section 1. Draw the responsibility flow, identify the state that must remain consistent, and name the operation that owns that invariant. Implement the core model and service, then wire the collaborators in main and run a concrete example. Show at least one rejected or boundary case from the tests. Explain the pattern at the point where it solves a problem, rather than starting with a list of pattern names.

For a distributed follow-up, distinguish thread safety inside this process from coordination across replicas. In-memory objects do not survive restarts. Agree on consistency, failure recovery and storage requirements before replacing them with remote infrastructure.

## 9. Practice next

1. Add actor authorization for customer, restaurant and driver operations.
2. Add an idempotency key to placeOrder and snapshot line-item prices individually.
3. Replace FirstAvailable with nearest-driver selection.
4. Model payment rejection and restaurant rejection with compensation.
