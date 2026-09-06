# Splitwise in Java — complete LLD walkthrough

Study order: requirements → model and flow → project tree → each complete file and explanation → patterns → walkthrough → limitations and exercises. This is a runnable single-process interview reference, with no external services or omitted source files.

## 1. Requirements and scope

Record an expense paid by one registered user and split it among unique registered participants. Support equal and exact splits. Maintain net balances: positive means the user should receive money, negative means they owe. Record explicit settlements. Use integer minor currency units and one currency. Expense/settlement IDs are deduplicated across the ledger. Pairwise debts, optimal debt simplification, refunds and multi-currency conversion are outside scope.

## 2. Design and responsibilities

```text
Main -> Ledger.AddExpense -> SplitStrategy -> validated shares
                         -> atomic balance updates + event ID deduplication
     -> Ledger.Settle -> debit receiver / credit payer
Invariant: sum(all balances) == 0
```

## 3. Project structure and running

Create each file at the shown relative path. All imports and package declarations are included.

```text
splitwise-java/
  src/study/Splitter.java
  src/study/EqualSplit.java
  src/study/ExactSplit.java
  src/study/Ledger.java
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

### 4.1. `src/study/Splitter.java`

```java
package study;

import java.util.Map;
import java.util.List;

interface Splitter {
    Map<String, Long> split(long total, List<String> users);
}
```

**How it works and why it belongs here:** Strategy contract returns one share per participant.

### 4.2. `src/study/EqualSplit.java`

```java
package study;

import java.util.Map;
import java.util.List;
import java.util.HashMap;

final class EqualSplit implements Splitter {

    public Map<String, Long> split(long total, List<String> users) {
        if (total <= 0 || users.isEmpty()) throw new IllegalArgumentException("split");
        var r = new HashMap<String, Long>();
        for (int i = 0; i<users.size(); i++)r.put(users.get(i), total/users.size()+(i<total%users.size()?1:0));
        return r;
    }
}
```

**How it works and why it belongs here:** Integer quotient/remainder division guarantees no lost minor units for valid participants.

### 4.3. `src/study/ExactSplit.java`

```java
package study;

import java.util.Map;
import java.util.List;

final class ExactSplit implements Splitter {
    private final Map<String, Long> shares;
    ExactSplit(Map<String, Long> s) {
        shares = Map.copyOf(s);
    }

    public Map<String, Long> split(long total, List<String> users) {
        return shares;
    }
}
```

**How it works and why it belongs here:** Immutable shares prevent callers from modifying the strategy after creation; ledger validates the totals.

### 4.4. `src/study/Ledger.java`

```java
package study;

import java.util.Map;
import java.util.List;
import java.util.Set;
import java.util.HashMap;
import java.util.HashSet;

final class Ledger {
    private final Map<String, Long> balances = new HashMap<>();
    private final Set<String> seen = new HashSet<>();
    Ledger(List<String> users) {
        for (String u:users) {
            if (u == null || u.isBlank() || balances.putIfAbsent(u, 0L) != null) throw new IllegalArgumentException("user");
        }
    }

    synchronized void addExpense(String id, String payer, long total, List<String> users, Splitter splitter) {
        if (id == null || id.isBlank()) throw new IllegalArgumentException("id");
        if (seen.contains(id)) return;
        if (!balances.containsKey(payer) || total <= 0 || users.isEmpty()) throw new IllegalArgumentException("expense");
        var members = new HashSet<String>();
        for (String u:users)if (!balances.containsKey(u) || !members.add(u)) throw new IllegalArgumentException("participant");
        var shares = splitter.split(total, List.copyOf(users));
        if (shares.size() != users.size()) throw new IllegalArgumentException("shares");
        long remaining = total;
        for (var e:shares.entrySet()) {
            long v = e.getValue();
            if (!members.contains(e.getKey()) || v<0 || v> remaining) throw new IllegalArgumentException("shares");
            remaining-=v;
        }
        if (remaining != 0) throw new IllegalArgumentException("sum");
        balances.put(payer, balances.get(payer)+total);
        shares.forEach((u, v) -> balances.put(u, balances.get(u)-v));
        seen.add(id);
    }

    synchronized void settle(String id, String from, String to, long amount) {
        if (id == null || id.isBlank()) throw new IllegalArgumentException("id");
        if (seen.contains(id)) return;
        if (!balances.containsKey(from) || !balances.containsKey(to) || from.equals(to) || amount <= 0) throw new IllegalArgumentException("settlement");
        balances.put(from, balances.get(from)+amount);
        balances.put(to, balances.get(to)-amount);
        seen.add(id);
    }

    synchronized Map<String, Long> balances() {
        return Map.copyOf(balances);
    }
}
```

**How it works and why it belongs here:** The service posts a balanced operation atomically. It exposes immutable balance snapshots. The set of command IDs is shared across expenses and settlements, so IDs must be unique across both command categories.

### 4.5. `src/study/Main.java`

```java
package study;

public class Main {

    public static void main(String[]args) {
        var users = java.util.List.of("Alice", "Bob", "Cara");
        var l = new Ledger(users);
        l.addExpense("e1", "Alice", 1000, users, new EqualSplit());
        l.settle("s1", "Bob", "Alice", 333);
        System.out.println(l.balances());
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
        var users = java.util.List.of("A", "B", "C");
        var l = new Ledger(users);
        l.addExpense("e", "A", 1000, users, new EqualSplit());
        l.addExpense("e", "A", 1000, users, new EqualSplit());
        check(l.balances().get("A") == 666, "remainder/idempotency");
        boolean invalid = false;
        try {
            l.addExpense("bad", "A", 1000, users, new ExactSplit(java.util.Map.of("A", 1L)));
        }
        catch (IllegalArgumentException e) {
            invalid = true;
        }
        check(invalid, "invalid shares");
        l.settle("s", "B", "A", 333);
        check(l.balances().get("B") == 0, "settle");
        check(l.balances().values().stream().mapToLong(Long::longValue).sum() == 0, "conservation");
        System.out.println("All behavior tests passed");
    }
}
```

**How it works and why it belongs here:** Asserts remainder allocation, deduplication, split validation and zero-sum balances.

## 5. Patterns and principles: where and why

| Pattern or principle | Concrete location | Reason |
|---|---|---|
| Strategy | Splitter / EqualSplit / ExactSplit | Expense allocation varies without changing ledger posting. |
| Encapsulation | Ledger mutation methods | All postings preserve the zero-sum invariant. |
| Idempotent command | seen event ID set | A repeated command ID posts at most once. |
| Dependency injection | splitter argument | Caller chooses the allocation policy explicitly. |

A mutex or synchronized method is a concurrency mechanism, not a GoF design pattern. An enum is a state representation, not automatically the State pattern. Interfaces are justified by interchangeable behavior or a useful boundary; inheritance is not required to demonstrate OOP.

### Applying SOLID without unnecessary abstractions

**Single responsibility:** the main entry point assembles dependencies; the domain service owns state invariants; policy collaborators own the behavior named in the pattern table. The tests exercise behavior through the public operations rather than depending on implementation maps.

**Open/closed:** inspect `Splitter / EqualSplit / ExactSplit` as the primary variation point. Where a policy interface exists, supply a new implementation without changing the state-transition algorithm. Where this example implements a specific data structure, do not claim its algorithm is interchangeable until you deliberately extract that boundary.

**Liskov substitution:** a replacement collaborator must preserve the documented contract, including invalid-input behavior, clock units, ownership rules and callback failure behavior. Merely matching a method signature is not enough.

**Interface segregation and dependency inversion:** interfaces expose the small question the caller needs answered. Constructors receive policies and clocks where tests or alternative behavior benefit. Simple records, enum values and internal containers remain concrete. A dedicated repository interface is useful when persistence is in scope; these samples do not pretend in-memory mutations automatically translate to database transactions.

## 6. Follow the example and the invariants

Alice pays 1000 for Alice, Bob and Cara. EqualSplit assigns 334 to the first participant Alice and 333 to each remaining participant, deterministically allocating the remainder. The ledger credits Alice 1000 then debits each person's share: Alice +666, Bob -333, Cara -333. Bob settling 333 with Alice raises Bob to zero and lowers Alice to +333. Every posting sums to zero. Split computation and validation occur before the balance mutation, so an invalid exact split leaves all balances unchanged.

### Verified execution

The complete project above was compiled and its behavior tests executed. The following is actual validation output (paths, timing and identifiers can vary):

```text
{Bob=0, Cara=-333, Alice=333}

All behavior tests passed
```

## 7. Best practices, edge cases and production extensions

Store amounts as long/int64 minor units, never binary floating point. Participant order controls remainder allocation; document this choice. Exact shares must sum exactly to the total and cannot be negative. IDs deduplicate but different payloads with the same ID currently return the original no-op behavior; production needs fingerprint conflicts and tenant scoping.

Net balances cannot answer “who owes whom” for a specific expense. Add an append-only expense/settlement journal and group identity for auditability, then derive balances transactionally. Settle records a reported transfer; it does not move money or verify a payment. This example allows settlements that reverse a balance because it records facts rather than enforcing a recommended settlement plan. Concurrent arithmetic is protected by a single lock, but persistent multi-server accounting needs database transactions. Very large values need overflow-checked posting; the sample assumes normal-sized amounts within int64/long range.

## 8. Presenting this in an interview

Start by agreeing on the scope in section 1. Draw the responsibility flow, identify the state that must remain consistent, and name the operation that owns that invariant. Implement the core model and service, then wire the collaborators in main and run a concrete example. Show at least one rejected or boundary case from the tests. Explain the pattern at the point where it solves a problem, rather than starting with a list of pattern names.

For a distributed follow-up, distinguish thread safety inside this process from coordination across replicas. In-memory objects do not survive restarts. Agree on consistency, failure recovery and storage requirements before replacing them with remote infrastructure.

## 9. Practice next

1. Add percentage splitting with deterministic remainder handling.
2. Store an immutable expense journal and implement reversal entries.
3. Generate a settlement plan from net balances without claiming it minimizes the number of transfers.
4. Reject reused IDs with conflicting payloads.
