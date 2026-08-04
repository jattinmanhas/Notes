# LLD Q3 — Concurrent Seat Booking / Ticket Reservation (Java)

> **Interview framing:** This question is *not* really about tickets. It's a concurrency question wearing a domain costume. The interviewer is checking one thing above all: **do you understand that `check-then-act` is not atomic, and do you know the full menu of ways to make it atomic — and their trade-offs?** Everything else (seat maps, pricing, payments) is scenery.

---

## Table of Contents

1. [The Problem in One Paragraph](#1-the-problem-in-one-paragraph)
2. [Clarifying Questions to Ask First](#2-clarifying-questions-to-ask-first)
3. [Requirements](#3-requirements)
4. [**Concurrency Primer — the theory you need**](#4-concurrency-primer--the-theory-you-need)
5. [Domain Model & Seat Lifecycle](#5-domain-model--seat-lifecycle)
6. [Choosing the Concurrency Strategy](#6-choosing-the-concurrency-strategy)
7. [The Code](#7-the-code)
8. [Interleaving Walkthroughs](#8-interleaving-walkthroughs)
9. [Extensibility](#9-extensibility)
10. [Failure Scenarios](#10-failure-scenarios)
11. [Testing Concurrent Code](#11-testing-concurrent-code)
12. [Interview Cheat Sheet](#12-interview-cheat-sheet)

---

## 1. The Problem in One Paragraph

A show has a fixed set of seats. Thousands of users hit "book" for the same seats within the same second (concert on-sale, IPL final). The system must guarantee **one seat is sold at most once**, must **hold** seats while the user completes payment (so they don't lose them mid-checkout), must **release** those holds automatically when the user abandons, and must survive retries, crashes and duplicate submissions without double-booking or double-charging.

The three hard parts:

| Hard part | Why it's hard |
|---|---|
| **No double-booking under contention** | The naive `if (available) { book(); }` is a check-then-act race. Two threads both pass the check. |
| **Temporary holds that expire** | A hold is a lock with a TTL held by a *user*, not a thread. Thread-based locks can't model it — the user goes to make coffee. |
| **Exactly-once effects across retries** | Confirm and payment can be retried. Idempotency (see Q1) is mandatory. |

---

## 2. Clarifying Questions to Ask First

1. **"Single JVM or multiple app servers?"** — This is *the* question. Single JVM → `ReentrantLock` is a valid answer. Multiple servers → in-process locks are useless and the database (or another shared arbiter) must decide. *(Assume multiple servers; I'll show both.)*
2. **"Do users select specific seats, or is it general admission?"** — Specific seats = set-based all-or-nothing reservation. GA = just a counter, which is a much easier `AtomicInteger`/`UPDATE … SET left = left - n WHERE left >= n` problem. *(Assume assigned seating; mention GA as the easy case.)*
3. **"Is a partial booking acceptable?"** — If I ask for seats A5, A6, A7 and only A5 is free, do I get A5? *(Assume **all-or-nothing** — that's what users expect for a group, and it's the more interesting concurrency problem.)*
4. **"How long is the payment window?"** — *(Assume a 10-minute hold TTL.)*
5. **"What's the read/write ratio?"** — Seat-map views massively outnumber bookings. Matters for choosing read locks / caching. *(Assume ~1000:1 reads.)*
6. **"Is a stale seat map acceptable to readers?"** — *(Assume yes — the seat map is a hint; the booking attempt is the source of truth. This is a huge simplification and worth stating.)*

---

## 3. Requirements

### 3.1 Functional

| # | Requirement |
|---|---|
| F1 | View the seat map for a show. |
| F2 | Hold a set of seats atomically (all-or-nothing) for a TTL. |
| F3 | Confirm a hold into a booking after successful payment. |
| F4 | Release a hold explicitly (user cancels) or implicitly (TTL expiry). |
| F5 | Never sell the same seat twice — the hard invariant. |
| F6 | Confirm and payment must be idempotent under retry. |
| F7 | Expired holds return seats to the pool automatically. |

### 3.2 Non-Functional

| # | Requirement | How the design meets it |
|---|---|---|
| N1 | **Correctness under contention is non-negotiable** | The invariant is enforced by a single atomic operation + a DB unique constraint as a backstop |
| N2 | **No lost updates, no phantom bookings** | Conditional `UPDATE … WHERE status = 'AVAILABLE'` with a row-count check |
| N3 | **No deadlocks** | Multi-seat locking always acquires in **sorted seat order**; single-statement DB updates avoid the problem entirely |
| N4 | **Bounded wait** | `tryLock(timeout)` — a booking request fails fast rather than hanging |
| N5 | **Liveness under crash** | Holds expire by wall-clock TTL, not by lock ownership, so a dead client can't wedge a seat |
| N6 | **Read scalability** | Seat-map reads never take a write lock; they're allowed to be stale |

---

## 4. Concurrency Primer — the theory you need

This section is the actual content of the interview. Everything here shows up in the design that follows.

### 4.1 The one bug everything else defends against: check-then-act

```java
// ❌ BROKEN. This is the entire question in four lines.
public boolean book(Seat seat, User user) {
    if (seat.getStatus() == AVAILABLE) {   // ← CHECK
        seat.setStatus(BOOKED);            // ← ACT
        seat.setOwner(user);
        return true;
    }
    return false;
}
```

```
Thread A                        Thread B
--------                        --------
read status → AVAILABLE
                                read status → AVAILABLE     ← both passed the check
write status = BOOKED
write owner = Alice
                                write status = BOOKED
                                write owner = Bob           ← Alice's booking silently vanishes
```

The gap between CHECK and ACT is where the money is lost. Every technique below is a different way to close that gap. The general name is a **TOCTOU** (time-of-check to time-of-use) race; the specific database name is a **lost update**.

> **The framing to say out loud:** *"I need the check and the act to be a single atomic operation. There are exactly three places I can make that happen: in the JVM with a lock, in the JVM lock-free with CAS, or in the database with a conditional write. Which one I pick depends on whether the state is shared across processes."*

### 4.2 Three separate guarantees: atomicity, visibility, ordering

Most people say "thread safety" and mean only the first. Naming all three is a strong signal.

| Guarantee | Question it answers | What breaks without it |
|---|---|---|
| **Atomicity** | Can another thread see a half-finished operation? | `count++` is read-modify-write; two increments become one |
| **Visibility** | Will another thread ever see my write? | Thread B loops forever on a `boolean stop` flag cached in a register/core cache |
| **Ordering** | Can the compiler/CPU reorder my statements? | Classic broken double-checked locking: another thread sees a non-null reference to a partially constructed object |

The Java Memory Model formalises this with **happens-before**. Key edges you should be able to recite:

- Program order within a single thread.
- An `unlock` on a monitor *happens-before* every subsequent `lock` on that same monitor.
- A write to a `volatile` field *happens-before* every subsequent read of that field.
- `Thread.start()` *happens-before* anything in the started thread; anything in a thread *happens-before* another thread's successful `join()`.
- Everything before putting an object into a `BlockingQueue`/`ConcurrentHashMap` *happens-before* another thread taking/reading it.

**The trap to avoid:** `volatile` gives you visibility and ordering but **not atomicity**. `volatile int count; count++;` is still broken. Conversely `synchronized` gives you all three.

### 4.3 The mutual-exclusion toolkit

| Tool | Use when | Notes |
|---|---|---|
| `synchronized` | Simple, short critical section | Reentrant, auto-released on exception, JIT can bias/elide. **Cannot time out or be interrupted** — that's its fatal flaw here. |
| `ReentrantLock` | You need `tryLock(timeout)`, interruptibility, fairness, or multiple `Condition`s | Must `unlock()` in `finally`. **This is what I use** — a booking request should fail in 200 ms, not block. |
| `ReadWriteLock` | Read-heavy, writes rare | Seat maps are read ~1000× more than written. Beware writer starvation with unfair policy. |
| `StampedLock` | Read-heavy and you can retry a read | Optimistic reads with no lock at all (`tryOptimisticRead` + `validate`). **Not reentrant** — a real footgun. |
| `Semaphore` | Limit N concurrent users of a resource | e.g. throttle concurrent payment-gateway calls. |
| `CountDownLatch` / `CyclicBarrier` | One-shot / repeated rendezvous | Mostly a *testing* tool here — see §11. |
| `ConcurrentHashMap` | Shared map | Its compound ops (`putIfAbsent`, `computeIfAbsent`, `merge`) are atomic. `containsKey` + `put` is **not**. |
| Atomics + CAS | Single-variable updates | Lock-free, no blocking, no deadlock. |

```java
// The ReentrantLock idiom. The finally block is not optional.
if (lock.tryLock(200, TimeUnit.MILLISECONDS)) {
    try   { /* critical section */ }
    finally { lock.unlock(); }
} else {
    throw new SeatServiceBusyException();   // fail fast, don't queue up a thundering herd
}
```

### 4.4 CAS — the idea behind every optimistic scheme

**Compare-And-Swap** is a single CPU instruction (`LOCK CMPXCHG` on x86): *"set this location to N, but only if it currently equals E; tell me whether you succeeded."* Check and act, fused into one uninterruptible step.

```java
// The CAS retry loop — the shape of every lock-free algorithm.
AtomicReference<SeatStatus> status = ...;
SeatStatus witnessed;
do {
    witnessed = status.get();
    if (witnessed != AVAILABLE) return false;      // someone else took it
} while (!status.compareAndSet(witnessed, HELD));  // retry if the world moved
return true;
```

Properties worth naming:

- **Lock-free**, not wait-free: no thread blocks, but an individual thread can be starved by unlucky retries.
- **No deadlock possible** — there's no lock to hold.
- Degrades under high contention (cache-line ping-pong, wasted retries). At extreme contention a lock can actually beat CAS.
- **ABA problem:** the value went A → B → A between your read and your CAS; the CAS succeeds but the world changed underneath. Fix with a version/stamp (`AtomicStampedReference`). **This is exactly why the database gets a `version` column** — a version number is a monotonically increasing stamp that makes ABA impossible.

> **The connecting insight to state in the interview:** *"Optimistic locking with a `version` column is just CAS, executed by the database instead of the CPU. `UPDATE … WHERE version = ?` **is** a compare-and-swap, and the affected-row count is the boolean it returns."* That one sentence ties the whole answer together.

### 4.5 Lock granularity: correctness is free, throughput is not

| Granularity | Correct? | Throughput | Deadlock risk |
|---|---|---|---|
| One global lock | ✅ | ❌ Terrible — the whole site serialises on one mutex | None |
| One lock **per show** | ✅ | ✅ Good — shows are independent, so N shows run in parallel | None (one lock held at a time) |
| One lock **per seat** | ✅ | ✅✅ Best in theory | ⚠️ **Real** — a multi-seat booking must hold several locks |

Per-show locking is the sweet spot for this problem: contention *within* a hot show is inherent (everyone wants the same 4 seats), so finer granularity buys much less than it looks like it will, and it costs you a deadlock hazard.

**Lock striping** is the general technique for getting per-key locks without allocating a lock per key forever:

```java
// N stripes, key hashed to a stripe. ConcurrentHashMap.computeIfAbsent is atomic,
// so two threads racing for the same show get the SAME lock object.
private final ConcurrentHashMap<ShowId, ReentrantLock> locks = new ConcurrentHashMap<>();
ReentrantLock lock = locks.computeIfAbsent(showId, k -> new ReentrantLock());
```

> ⚠️ `computeIfAbsent`'s mapping function runs **while holding the bin lock**. Never do I/O, blocking work, or a recursive update to the same map inside it — you can deadlock the map itself. Allocating a `new ReentrantLock()` is fine.

### 4.6 Deadlock — and the one-line prevention

Deadlock needs all four **Coffman conditions** simultaneously:

1. **Mutual exclusion** — the resource can't be shared.
2. **Hold and wait** — a thread holds one lock while requesting another.
3. **No preemption** — locks can't be forcibly taken away.
4. **Circular wait** — a cycle exists in the wait-for graph.

Break any one and deadlock becomes impossible.

```
❌ Deadlock: Alice books seats {A5, A6}; Bob books {A6, A5}

Alice: lock(A5) ✔ … lock(A6) ⏳ waiting for Bob
Bob:   lock(A6) ✔ … lock(A5) ⏳ waiting for Alice
                        → circular wait → both hang forever

✅ Fix — GLOBAL LOCK ORDERING. Always acquire in sorted seat order:
Alice: lock(A5) ✔ lock(A6) ✔        Bob: lock(A5) ⏳ (just waits, then proceeds)
                        → no cycle is constructible → deadlock impossible
```

Sorting the seat IDs before locking removes condition 4 for one line of code. The other practical defences: `tryLock` with a timeout (breaks *no preemption* — back off and retry), and holding only one lock at a time (breaks *hold and wait*, which is what per-show locking achieves).

**Databases deadlock for exactly the same reason** — two transactions locking rows in opposite orders. The same fix applies: `SELECT … FOR UPDATE … ORDER BY seat_id`. The difference is the DB *detects* the cycle and kills a victim with a deadlock error, so you must be prepared to catch and retry.

### 4.7 Pessimistic vs optimistic locking

| | **Pessimistic** | **Optimistic** |
|---|---|---|
| Assumption | Conflict is likely | Conflict is rare |
| Mechanism | Take the lock *before* reading (`SELECT … FOR UPDATE`, `synchronized`) | Read freely, detect conflict *at write time* (`WHERE version = ?`) |
| Cost when no conflict | Lock overhead on every request | ~Zero |
| Cost when conflict | Waiting | Wasted work + retry |
| Deadlock possible | ✅ yes | ❌ no |
| Scales across processes | Only via a shared DB/lock service | ✅ naturally |
| Best for | Long critical sections, very high contention on one row | Short critical sections, low-to-moderate contention |

Seat booking looks like the pessimistic case (thousands fighting for one seat), but there's a better third option that gets the best of both:

**Make the check part of the write.** One statement, no read-then-write gap at all:

```sql
UPDATE show_seat
   SET status = 'HELD', hold_id = :holdId, hold_expires_at = :expiry
 WHERE show_id = :showId
   AND seat_id IN (:seats)
   AND (status = 'AVAILABLE' OR (status = 'HELD' AND hold_expires_at < :now));
-- affected rows == number of requested seats ? COMMIT : ROLLBACK
```

Why this is the answer:

- **There is no gap.** The database evaluates the predicate and writes under the same row locks. It *is* a compare-and-swap over a set of rows.
- **All-or-nothing falls out for free** from the row-count check plus the transaction.
- **Expiry is handled lazily in the predicate** — no need for the sweeper to have run first. The sweeper becomes a cleanup optimisation, not a correctness requirement. That's a big deal.
- **No explicit locking** in application code, so no application-level deadlock.

### 4.8 Database isolation levels (know the anomalies by name)

| Level | Dirty read | Non-repeatable read | Phantom read | Lost update |
|---|---|---|---|---|
| READ UNCOMMITTED | ✅ possible | ✅ | ✅ | ✅ |
| READ COMMITTED | ❌ | ✅ | ✅ | ✅ **← the default in Postgres/Oracle, and it does NOT save you** |
| REPEATABLE READ | ❌ | ❌ | ✅ (❌ in InnoDB, which uses gap locks) | ❌ (detected) |
| SERIALIZABLE | ❌ | ❌ | ❌ | ❌ |

The point to make: **READ COMMITTED does not prevent the lost update in §4.1.** Both transactions read `AVAILABLE`, both write `BOOKED`, both commit. Isolation level alone is not a solution; you need `FOR UPDATE`, a version predicate, or a constraint.

Engine specifics worth a sentence:

- **Postgres** REPEATABLE READ is snapshot isolation: the second writer gets `could not serialize access` and you **must retry**. `SERIALIZABLE` adds predicate locking and more retries. Retry logic is mandatory, not optional.
- **MySQL/InnoDB** REPEATABLE READ uses next-key (gap) locks, so it blocks rather than aborting — which prevents phantoms but creates more deadlock opportunities.

### 4.9 The last line of defence: make the invariant a constraint

Application logic can have bugs. Database constraints cannot be bypassed.

```sql
CREATE TABLE booking_seat (
    booking_id  BIGINT NOT NULL,
    show_id     BIGINT NOT NULL,
    seat_id     BIGINT NOT NULL,
    CONSTRAINT uq_seat_per_show UNIQUE (show_id, seat_id)   -- ← the real guarantee
);
```

Even if every lock, every version check and every code path fails, the second insert throws a unique-constraint violation and the transaction rolls back. **Say this out loud:** *"Locks are how I get good behaviour; the unique constraint is how I get correctness. I want both, because they fail differently."*

### 4.10 Why I would *not* reach for a distributed lock

The tempting answer for multi-server is "Redis lock". Push back on it:

- A lock with a TTL isn't mutual exclusion. If holder A stops the world for a GC pause longer than the TTL, the lock expires, B acquires it, and now **two threads believe they hold it**. Redlock doesn't fix this; it's a well-known debate.
- The correct patch is **fencing tokens** — a monotonically increasing number issued with the lock, which the storage layer checks and rejects if stale. But if your storage can already reject stale writes… you didn't need the lock, you needed the version check.
- So: **use the database's atomicity as the arbiter.** A distributed lock can still be useful as a *contention reducer* (fail fast without hitting the DB), but never as the correctness mechanism.

### 4.11 A hold is not a lock — and that distinction matters

A 10-minute seat hold is **not** a thread lock. It is owned by a *human*, survives across HTTP requests, and must expire on wall-clock time regardless of what any thread is doing.

| | Thread lock | Seat hold |
|---|---|---|
| Owner | A thread | A user/session |
| Lifetime | Microseconds | Minutes |
| Released by | `unlock()` in `finally` | Confirm, explicit release, or **TTL expiry** |
| Survives a crash? | No (released) | Yes — and that's why expiry is by timestamp, not ownership |
| Stored in | JVM memory | **Durable storage** |

This is why the hold's `expires_at` is a column and why the expiry predicate is inlined into the hold query. **Lazy expiry (check at read/write time) is correct on its own; the background sweeper only exists to keep the seat map pretty.** If you rely on the sweeper for correctness, a sweeper outage becomes a customer-visible bug.

### 4.12 Backpressure

Ten thousand threads blocking on one show's lock is its own outage. Bound the work:

- Fixed thread pool with a **bounded** queue and `CallerRunsPolicy` or an explicit 503.
- `tryLock(200ms)` → fail fast with "seats are busy, try again", not an indefinite block.
- Optionally a `Semaphore` per hot show to cap in-flight attempts.

---

## 5. Domain Model & Seat Lifecycle

### 5.1 Seat state machine

```
                    ┌──────────────┐
       ┌───────────►│  AVAILABLE   │◄────────────┐
       │            └──────┬───────┘             │
       │                   │ hold(ttl)           │
       │                   ▼                     │
       │            ┌──────────────┐             │
       │  release() │     HELD     │             │ TTL expiry
       └────────────┤ (expires_at) ├─────────────┘ (lazy predicate
                    └──────┬───────┘                + sweeper)
                           │ confirm()  [payment captured]
                           ▼
                    ┌──────────────┐
                    │    BOOKED    │   ← TERMINAL. Only refund/cancel leaves it,
                    └──────────────┘     and that's a different use case.
```

Guards worth stating explicitly:

- `AVAILABLE → HELD` requires the seat to be available **or** to be held by an *already-expired* hold.
- `HELD → BOOKED` requires the hold to be **the same hold** and **not yet expired**. Confirming an expired hold must fail — otherwise a slow payment steals a seat someone else legitimately re-held.
- `BOOKED` is terminal. No path back except an explicit refund flow.

### 5.2 Hold lifecycle vs booking lifecycle

Two objects, deliberately separate:

```
SeatHold                              Booking
  holdId (idempotency anchor)           bookingId
  showId, seats[], userId               holdId
  createdAt, expiresAt                  paymentId
  status: ACTIVE|CONFIRMED|             seats[]
          RELEASED|EXPIRED              confirmedAt
```

Why separate: the hold is *pessimistic and temporary*, the booking is *permanent and financial*. Merging them forces you to model a "booking that isn't a booking yet", which is where bugs breed.

### 5.3 Where the concurrency actually lives

```
  Read path  (99.9% of traffic)          Write path (the contested one)
  ─────────────────────────────          ──────────────────────────────
  GET /shows/{id}/seats                  POST /holds        ← ATOMIC, contested
  → cached, possibly stale               POST /holds/{id}/confirm ← idempotent
  → NO locks taken                       DELETE /holds/{id}
                                         (background) hold sweeper
```

Only **one** operation needs to be atomic: `hold`. Confirm is idempotent rather than contested (the hold already fenced the seats off). Reads take no locks at all. Narrowing the problem to a single critical operation like this is itself a design win worth pointing out.

---

## 6. Choosing the Concurrency Strategy

Walk the interviewer down this ladder — showing the progression is worth more than jumping to the answer.

| # | Approach | Verdict |
|---|---|---|
| 1 | `synchronized` on the booking method | ❌ Global lock; whole site serialises; also useless across servers |
| 2 | `synchronized` per show object | ⚠️ Correct in one JVM, still useless across servers |
| 3 | `ReentrantLock` per show via `ConcurrentHashMap` + `tryLock(timeout)` | ✅ **Best single-JVM answer.** Fail-fast, no deadlock (one lock at a time) |
| 4 | Per-seat locks acquired in sorted order | ⚠️ More parallelism, deadlock-safe only *because* of the ordering; complexity rarely pays off |
| 5 | `SELECT … FOR UPDATE ORDER BY seat_id` (pessimistic DB) | ✅ Correct across servers. Holds row locks for the whole transaction; deadlock-safe via ordering |
| 6 | **Conditional `UPDATE … WHERE status = 'AVAILABLE'` + row-count check** | ✅✅ **The answer.** One statement, no gap, all-or-nothing, expiry inline, no app-level locks |
| 7 | Redis distributed lock | ❌ Not a correctness mechanism (§4.10). Fine as a contention reducer |
| 8 | Unique constraint on `(show_id, seat_id)` | ✅ Always add it — the backstop, not the strategy |

**My answer: #6, backed by #8, with #3 as the in-process fast path when everything runs in one JVM.**

The single sentence for the whiteboard:

> *"I make the seat allocation a single conditional UPDATE whose WHERE clause contains the availability check, then compare the affected row count to the number of seats requested. That's a compare-and-swap across a set of rows, executed by the database, so there is no window between check and act — and a unique constraint on (show_id, seat_id) catches anything that somehow gets past it."*

---

## 7. The Code

> Java 17. Records, sealed interfaces, `instanceof` patterns (Java 16+) — deliberately **no** pattern `switch`, so this compiles on 17 without preview flags.

### 7.1 Value objects and results

```java
// SeatStatus.java / HoldStatus.java
package com.ticketing.booking;

public enum SeatStatus { AVAILABLE, HELD, BOOKED }

public enum HoldStatus { ACTIVE, CONFIRMED, RELEASED, EXPIRED }
```

```java
// ShowId.java / SeatId.java / BookingId.java / HoldId.java
// (one public type per file — grouped here only for readability)
package com.ticketing.booking;

import java.util.UUID;

public record ShowId(String value)    { }
public record SeatId(String value)    implements Comparable<SeatId> {
    /** Comparable is NOT decoration: sorted acquisition is how we avoid deadlock. */
    @Override public int compareTo(SeatId o) { return value.compareTo(o.value); }
}
public record BookingId(String value) { }
public record HoldId(String value) {
    public static HoldId newId() { return new HoldId(UUID.randomUUID().toString()); }
}
```

```java
// SeatHold.java
package com.ticketing.booking;

import java.time.Instant;
import java.util.Set;

/**
 * A hold is owned by a USER, not a thread. It survives across HTTP requests and
 * expires on wall-clock time, which is why expiresAt is data rather than the
 * lifetime of a lock object.
 */
public record SeatHold(
        HoldId id,
        ShowId showId,
        Set<SeatId> seats,
        String userId,
        Instant createdAt,
        Instant expiresAt,
        HoldStatus status) {

    public SeatHold {
        seats = Set.copyOf(seats);           // defensive: records are only shallowly final
    }

    public boolean isExpiredAt(Instant now) { return !now.isBefore(expiresAt); }

    public boolean isActiveAt(Instant now)  {
        return status == HoldStatus.ACTIVE && !isExpiredAt(now);
    }

    public SeatHold withStatus(HoldStatus newStatus) {
        return new SeatHold(id, showId, seats, userId, createdAt, expiresAt, newStatus);
    }
}
```

```java
// HoldResult.java
package com.ticketing.booking;

import java.util.Set;

/**
 * Sealed so callers must handle every outcome. Note that "busy" is a first-class
 * result, not an exception: under contention, failing fast is normal operation.
 */
public sealed interface HoldResult {

    record Granted(SeatHold hold) implements HoldResult {}

    /** All-or-nothing: we report exactly which seats blocked the request. */
    record Rejected(Set<SeatId> unavailableSeats) implements HoldResult {}

    /** Could not acquire the show lock within the timeout — client should retry. */
    record Busy() implements HoldResult {}
}
```

```java
// ConfirmResult.java
package com.ticketing.booking;

public sealed interface ConfirmResult {

    record Confirmed(BookingId bookingId) implements ConfirmResult {}

    record Failed(Reason reason) implements ConfirmResult {}

    enum Reason { HOLD_NOT_FOUND, HOLD_EXPIRED, ALREADY_CONFIRMED, PAYMENT_FAILED }
}
```

```java
// SeatInventory.java
package com.ticketing.booking;

import java.time.Duration;
import java.time.Instant;
import java.util.Set;

/**
 * The ONLY component that has to be atomic. Everything else in the system is either
 * a read (allowed to be stale) or an idempotent follow-up.
 */
public interface SeatInventory {

    /** Atomic, all-or-nothing. This is the contested operation. */
    HoldResult hold(ShowId showId, Set<SeatId> seats, String userId, Duration ttl);

    /** HELD -> BOOKED. Must fail if the hold expired, even by a millisecond. */
    boolean confirm(HoldId holdId);

    /** Explicit user cancellation. Idempotent. */
    void release(HoldId holdId);

    /** Cleanup only — correctness does NOT depend on this having run. */
    int expireStaleHolds(Instant now);

    Set<SeatId> availableSeats(ShowId showId);
}
```

### 7.2 In-memory implementation — locks done properly

The right answer *if and only if* everything runs in one JVM. Present it, then say why it doesn't survive a second app server.

```java
// InMemorySeatInventory.java
package com.ticketing.booking;

import java.time.Clock;
import java.time.Duration;
import java.time.Instant;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.LinkedHashSet;
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.ReentrantLock;

public final class InMemorySeatInventory implements SeatInventory {

    /** Fail fast. A booking request must never block indefinitely on a hot show. */
    private static final long LOCK_TIMEOUT_MS = 200;

    /** Mutable seat state, only ever touched while holding the owning show's lock. */
    private static final class SeatRecord {
        SeatStatus status = SeatStatus.AVAILABLE;
        HoldId holdId;
        Instant expiresAt;

        boolean isFreeAt(Instant now) {
            return status == SeatStatus.AVAILABLE
                || (status == SeatStatus.HELD && expiresAt != null && expiresAt.isBefore(now));
        }

        void release() { status = SeatStatus.AVAILABLE; holdId = null; expiresAt = null; }
    }

    /**
     * LOCK GRANULARITY: one lock per show. Shows are independent, so different shows
     * proceed in parallel; contention within a show is inherent to the problem.
     * Crucially, a request only ever holds ONE lock, which breaks the "hold and wait"
     * Coffman condition — deadlock is structurally impossible here.
     */
    private static final class ShowInventory {
        final ReentrantLock lock = new ReentrantLock();
        final Map<SeatId, SeatRecord> seats = new HashMap<>();
        final Map<HoldId, SeatHold> holds = new HashMap<>();
    }

    private final ConcurrentHashMap<ShowId, ShowInventory> shows = new ConcurrentHashMap<>();
    /** holdId -> showId, so confirm/release can find the right lock. */
    private final ConcurrentHashMap<HoldId, ShowId> holdIndex = new ConcurrentHashMap<>();
    private final Clock clock;

    public InMemorySeatInventory(Clock clock) { this.clock = clock; }

    public void registerShow(ShowId showId, Set<SeatId> seatIds) {
        ShowInventory inv = new ShowInventory();
        for (SeatId s : seatIds) inv.seats.put(s, new SeatRecord());
        // putIfAbsent is atomic; containsKey-then-put would be a check-then-act race
        // in this very class. The bug we're designing against is easy to reintroduce.
        shows.putIfAbsent(showId, inv);
    }

    // =====================================================================
    //  THE CONTESTED OPERATION
    // =====================================================================

    @Override
    public HoldResult hold(ShowId showId, Set<SeatId> requested, String userId, Duration ttl) {
        ShowInventory show = shows.get(showId);
        if (show == null) throw new IllegalArgumentException("unknown show " + showId);

        boolean acquired;
        try {
            acquired = show.lock.tryLock(LOCK_TIMEOUT_MS, TimeUnit.MILLISECONDS);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();   // ALWAYS restore the flag
            return new HoldResult.Busy();
        }
        if (!acquired) return new HoldResult.Busy();

        try {
            Instant now = clock.instant();

            // ---- CHECK and ACT are both inside the critical section. That is the
            // ---- whole point; splitting them is the bug from §4.1.
            Set<SeatId> unavailable = new LinkedHashSet<>();
            for (SeatId id : requested) {
                SeatRecord rec = show.seats.get(id);
                if (rec == null || !rec.isFreeAt(now)) unavailable.add(id);
            }
            if (!unavailable.isEmpty()) {
                return new HoldResult.Rejected(Set.copyOf(unavailable));   // all-or-nothing
            }

            HoldId holdId = HoldId.newId();
            Instant expiresAt = now.plus(ttl);
            for (SeatId id : requested) {
                SeatRecord rec = show.seats.get(id);
                // If this seat was held by an EXPIRED hold, we are stealing it —
                // mark that stale hold dead so its owner cannot confirm later.
                if (rec.holdId != null) invalidate(show, rec.holdId);
                rec.status = SeatStatus.HELD;
                rec.holdId = holdId;
                rec.expiresAt = expiresAt;
            }

            SeatHold hold = new SeatHold(holdId, showId, requested, userId,
                                         now, expiresAt, HoldStatus.ACTIVE);
            show.holds.put(holdId, hold);
            holdIndex.put(holdId, showId);
            return new HoldResult.Granted(hold);

        } finally {
            show.lock.unlock();          // NEVER outside a finally
        }
    }

    private void invalidate(ShowInventory show, HoldId staleId) {
        SeatHold stale = show.holds.get(staleId);
        if (stale != null && stale.status() == HoldStatus.ACTIVE) {
            show.holds.put(staleId, stale.withStatus(HoldStatus.EXPIRED));
        }
    }

    // =====================================================================
    //  CONFIRM / RELEASE
    // =====================================================================

    @Override
    public boolean confirm(HoldId holdId) {
        ShowId showId = holdIndex.get(holdId);
        if (showId == null) return false;
        ShowInventory show = shows.get(showId);

        show.lock.lock();
        try {
            SeatHold hold = show.holds.get(holdId);
            if (hold == null) return false;
            if (hold.status() == HoldStatus.CONFIRMED) return true;   // idempotent
            Instant now = clock.instant();
            if (!hold.isActiveAt(now)) return false;                  // expired: MUST fail

            for (SeatId id : hold.seats()) {
                SeatRecord rec = show.seats.get(id);
                // Re-verify ownership: another request may have stolen an expired seat.
                if (rec == null || !holdId.equals(rec.holdId)) return false;
            }
            for (SeatId id : hold.seats()) {
                SeatRecord rec = show.seats.get(id);
                rec.status = SeatStatus.BOOKED;
                rec.expiresAt = null;
            }
            show.holds.put(holdId, hold.withStatus(HoldStatus.CONFIRMED));
            return true;
        } finally {
            show.lock.unlock();
        }
    }

    @Override
    public void release(HoldId holdId) {
        ShowId showId = holdIndex.get(holdId);
        if (showId == null) return;
        ShowInventory show = shows.get(showId);

        show.lock.lock();
        try {
            SeatHold hold = show.holds.get(holdId);
            if (hold == null || hold.status() != HoldStatus.ACTIVE) return;  // idempotent
            for (SeatId id : hold.seats()) {
                SeatRecord rec = show.seats.get(id);
                if (rec != null && holdId.equals(rec.holdId)) rec.release();
            }
            show.holds.put(holdId, hold.withStatus(HoldStatus.RELEASED));
        } finally {
            show.lock.unlock();
        }
    }

    // =====================================================================
    //  CLEANUP (optimisation, not correctness — see §4.11)
    // =====================================================================

    @Override
    public int expireStaleHolds(Instant now) {
        int expired = 0;
        for (Map.Entry<ShowId, ShowInventory> e : shows.entrySet()) {
            ShowInventory show = e.getValue();
            // tryLock, not lock: the sweeper must never block live booking traffic.
            if (!show.lock.tryLock()) continue;
            try {
                List<HoldId> dead = new ArrayList<>();
                for (SeatHold h : show.holds.values()) {
                    if (h.status() == HoldStatus.ACTIVE && h.isExpiredAt(now)) dead.add(h.id());
                }
                for (HoldId id : dead) {
                    SeatHold h = show.holds.get(id);
                    for (SeatId s : h.seats()) {
                        SeatRecord rec = show.seats.get(s);
                        if (rec != null && id.equals(rec.holdId)
                                && rec.status == SeatStatus.HELD) {
                            rec.release();
                        }
                    }
                    show.holds.put(id, h.withStatus(HoldStatus.EXPIRED));
                    expired++;
                }
            } finally {
                show.lock.unlock();
            }
        }
        return expired;
    }

    @Override
    public Set<SeatId> availableSeats(ShowId showId) {
        ShowInventory show = shows.get(showId);
        if (show == null) return Set.of();
        Instant now = clock.instant();
        // Deliberately LOCK-FREE and therefore possibly stale. Seat maps are a hint;
        // the hold() call is the source of truth. This is what keeps reads scalable.
        Set<SeatId> free = new LinkedHashSet<>();
        for (Map.Entry<SeatId, SeatRecord> e : show.seats.entrySet()) {
            if (e.getValue().isFreeAt(now)) free.add(e.getKey());
        }
        return free;
    }
}
```

> **The honest caveat to volunteer:** *"This is correct on one JVM and worthless on two — `ReentrantLock` is process-local. The moment we scale horizontally, the arbiter has to be the shared store."* Saying this before the interviewer asks is worth a lot.

### 7.3 The production answer — the database as the arbiter

#### Schema

```sql
-- ids are VARCHAR here so the SQL lines up with the Java value objects
-- (ShowId/SeatId wrap String); BIGINT surrogate keys work identically.
CREATE TABLE show_seat (
    show_id          VARCHAR(36) NOT NULL,
    seat_id          VARCHAR(36) NOT NULL,
    status           VARCHAR(16) NOT NULL,   -- AVAILABLE | HELD | BOOKED
    hold_id          VARCHAR(36) NULL,
    hold_expires_at  TIMESTAMP   NULL,
    version          BIGINT      NOT NULL DEFAULT 0,
    PRIMARY KEY (show_id, seat_id)           -- one row per seat per show
);

-- Sweeper / lazy-expiry support.
CREATE INDEX idx_show_seat_expiry ON show_seat (status, hold_expires_at);

CREATE TABLE booking (
    booking_id       VARCHAR(36) PRIMARY KEY,
    hold_id          VARCHAR(36) NOT NULL,
    user_id          VARCHAR(64) NOT NULL,
    payment_id       VARCHAR(64) NULL,
    confirmed_at     TIMESTAMP   NOT NULL,
    CONSTRAINT uq_booking_hold UNIQUE (hold_id)      -- idempotency: 1 hold -> 1 booking
);

CREATE TABLE booking_seat (
    booking_id       VARCHAR(36) NOT NULL REFERENCES booking(booking_id),
    show_id          VARCHAR(36) NOT NULL,
    seat_id          VARCHAR(36) NOT NULL,
    -- ⭐ THE INVARIANT AS A CONSTRAINT. Application bugs cannot get past this.
    CONSTRAINT uq_seat_per_show UNIQUE (show_id, seat_id)
);
```

Three separate defences, and they fail differently — that's the point:

1. The conditional `UPDATE` predicate — normal operation.
2. `uq_booking_hold` — makes double-confirm of one hold impossible (idempotency).
3. `uq_seat_per_show` — makes a double-sold seat impossible, full stop.

#### The atomic hold

```java
// JdbcSeatInventory.java
package com.ticketing.booking;

import java.sql.Connection;
import java.sql.PreparedStatement;
import java.sql.SQLException;
import java.sql.Timestamp;
import java.time.Clock;
import java.time.Duration;
import java.time.Instant;
import java.util.ArrayList;
import java.util.List;
import java.util.Set;
import java.util.TreeSet;
import javax.sql.DataSource;

public final class JdbcSeatInventory implements SeatInventory {

    private final DataSource dataSource;
    private final Clock clock;

    public JdbcSeatInventory(DataSource dataSource, Clock clock) {
        this.dataSource = dataSource;
        this.clock = clock;
    }

    /**
     * ⭐ THE CORE OF THE WHOLE DESIGN.
     *
     * A single UPDATE whose WHERE clause contains the availability check. The database
     * evaluates the predicate and writes under the same row locks, so there is NO gap
     * between check and act — this is a compare-and-swap over a set of rows, and the
     * affected-row count is the boolean it returns.
     *
     * Three things fall out for free:
     *   1. All-or-nothing  — rowsAffected != seats.size() ⇒ ROLLBACK.
     *   2. Lazy expiry     — the predicate treats an expired hold as available, so
     *                        correctness does not depend on the sweeper having run.
     *   3. No app locks    — hence no application-level deadlock.
     */
    @Override
    public HoldResult hold(ShowId showId, Set<SeatId> requested, String userId, Duration ttl) {
        // Sorted: keeps the DB's internal row-lock acquisition order consistent across
        // transactions, which minimises engine-level deadlocks (§4.6).
        List<SeatId> seats = new ArrayList<>(new TreeSet<>(requested));

        Instant now = clock.instant();
        Instant expiresAt = now.plus(ttl);
        HoldId holdId = HoldId.newId();

        String placeholders = "?,".repeat(seats.size());
        placeholders = placeholders.substring(0, placeholders.length() - 1);

        String sql = """
                UPDATE show_seat
                   SET status = 'HELD',
                       hold_id = ?,
                       hold_expires_at = ?,
                       version = version + 1
                 WHERE show_id = ?
                   AND seat_id IN (%s)
                   AND (status = 'AVAILABLE'
                        OR (status = 'HELD' AND hold_expires_at < ?))
                """.formatted(placeholders);

        try (Connection conn = dataSource.getConnection()) {
            conn.setAutoCommit(false);
            try (PreparedStatement ps = conn.prepareStatement(sql)) {
                int i = 1;
                ps.setString(i++, holdId.value());
                ps.setTimestamp(i++, Timestamp.from(expiresAt));
                ps.setString(i++, showId.value());
                for (SeatId s : seats) ps.setString(i++, s.value());
                ps.setTimestamp(i, Timestamp.from(now));

                int rows = ps.executeUpdate();

                if (rows != seats.size()) {
                    conn.rollback();                     // ← all-or-nothing
                    return new HoldResult.Rejected(Set.copyOf(requested));
                }
                conn.commit();
                return new HoldResult.Granted(new SeatHold(
                        holdId, showId, requested, userId, now, expiresAt, HoldStatus.ACTIVE));
            } catch (SQLException e) {
                conn.rollback();
                throw e;
            }
        } catch (SQLException e) {
            if (isSerializationFailureOrDeadlock(e)) {
                // Postgres 40001 / 40P01, MySQL 1213. The engine chose us as the victim.
                // These are EXPECTED under contention and must be retried, not logged
                // as errors. Retry with jittered backoff (see Q1).
                return new HoldResult.Busy();
            }
            throw new IllegalStateException("hold failed", e);
        }
    }

    private boolean isSerializationFailureOrDeadlock(SQLException e) {
        String state = e.getSQLState();
        return "40001".equals(state)      // serialization_failure (PG RR/SERIALIZABLE)
            || "40P01".equals(state)      // deadlock_detected     (PG)
            || e.getErrorCode() == 1213;  // ER_LOCK_DEADLOCK      (MySQL)
    }

    /**
     * HELD -> BOOKED, gated on (a) the hold still owning the rows and (b) not expired.
     * Same trick: the guard is in the WHERE clause, so there's no read-then-write gap.
     */
    @Override
    public boolean confirm(HoldId holdId) {
        String sql = """
                UPDATE show_seat
                   SET status = 'BOOKED',
                       hold_expires_at = NULL,
                       version = version + 1
                 WHERE hold_id = ?
                   AND status = 'HELD'
                   AND hold_expires_at >= ?
                """;
        String countSql = "SELECT COUNT(*) FROM show_seat WHERE hold_id = ?";

        try (Connection conn = dataSource.getConnection()) {
            conn.setAutoCommit(false);
            int expected;
            try (PreparedStatement ps = conn.prepareStatement(countSql)) {
                ps.setString(1, holdId.value());
                var rs = ps.executeQuery();
                rs.next();
                expected = rs.getInt(1);
            }
            try (PreparedStatement ps = conn.prepareStatement(sql)) {
                ps.setString(1, holdId.value());
                ps.setTimestamp(2, Timestamp.from(clock.instant()));
                int rows = ps.executeUpdate();
                if (expected == 0 || rows != expected) {
                    conn.rollback();     // expired, stolen, or partially confirmed
                    return false;
                }
                conn.commit();
                return true;
            }
        } catch (SQLException e) {
            throw new IllegalStateException("confirm failed", e);
        }
    }

    @Override
    public void release(HoldId holdId) {
        String sql = """
                UPDATE show_seat
                   SET status = 'AVAILABLE', hold_id = NULL,
                       hold_expires_at = NULL, version = version + 1
                 WHERE hold_id = ? AND status = 'HELD'
                """;
        execute(sql, holdId.value());
    }

    /** Cleanup pass. Batched and bounded so it never long-locks a hot show. */
    @Override
    public int expireStaleHolds(Instant now) {
        String sql = """
                UPDATE show_seat
                   SET status = 'AVAILABLE', hold_id = NULL,
                       hold_expires_at = NULL, version = version + 1
                 WHERE status = 'HELD' AND hold_expires_at < ?
                """;
        try (Connection conn = dataSource.getConnection();
             PreparedStatement ps = conn.prepareStatement(sql)) {
            ps.setTimestamp(1, Timestamp.from(now));
            return ps.executeUpdate();
        } catch (SQLException e) {
            throw new IllegalStateException("expiry sweep failed", e);
        }
    }

    @Override
    public Set<SeatId> availableSeats(ShowId showId) {
        // Read-only, no locks, may be stale by design. Serve from a cache in production.
        String sql = """
                SELECT seat_id FROM show_seat
                 WHERE show_id = ?
                   AND (status = 'AVAILABLE'
                        OR (status = 'HELD' AND hold_expires_at < ?))
                """;
        Set<SeatId> out = new TreeSet<>();
        try (Connection conn = dataSource.getConnection();
             PreparedStatement ps = conn.prepareStatement(sql)) {
            ps.setString(1, showId.value());
            ps.setTimestamp(2, Timestamp.from(clock.instant()));
            var rs = ps.executeQuery();
            while (rs.next()) out.add(new SeatId(rs.getString(1)));
            return out;
        } catch (SQLException e) {
            throw new IllegalStateException("seat map query failed", e);
        }
    }

    private void execute(String sql, String param) {
        try (Connection conn = dataSource.getConnection();
             PreparedStatement ps = conn.prepareStatement(sql)) {
            ps.setString(1, param);
            ps.executeUpdate();
        } catch (SQLException e) {
            throw new IllegalStateException("statement failed", e);
        }
    }
}
```

#### The pessimistic alternative, for comparison

Worth writing on the board so you can contrast them:

```sql
BEGIN;
  -- ORDER BY is not cosmetic: it imposes a global lock order across transactions,
  -- which is what makes engine-level deadlock impossible (§4.6).
  SELECT seat_id, status
    FROM show_seat
   WHERE show_id = :showId AND seat_id IN (:seats)
   ORDER BY seat_id
     FOR UPDATE;                       -- row locks held until COMMIT

  -- application checks every row is AVAILABLE (or expired) ...
  UPDATE show_seat SET status = 'HELD', ... WHERE show_id = :showId AND seat_id IN (:seats);
COMMIT;
```

| | Conditional UPDATE | `SELECT … FOR UPDATE` |
|---|---|---|
| Round trips | 1 | 2+ |
| Lock hold time | Duration of one statement | Whole transaction (includes app think-time) |
| Deadlock risk | Very low | Real — mitigated only by `ORDER BY` |
| Can inspect rows before deciding | ❌ (only a row count) | ✅ (can report *which* seats failed) |
| Verdict | ✅ Default choice | Use when you need per-seat diagnostics |

### 7.4 Booking service — where idempotency lives

```java
// BookingService.java
package com.ticketing.booking;

import java.time.Clock;
import java.time.Duration;
import java.time.Instant;
import java.util.Optional;
import java.util.Set;
import java.util.concurrent.ConcurrentHashMap;

public final class BookingService {

    /**
     * Don't start a payment we can't finish. If the hold has less than this left,
     * reject up front rather than risk charging the card and then failing to confirm.
     */
    private static final Duration MIN_REMAINING_FOR_PAYMENT = Duration.ofSeconds(30);

    private final SeatInventory inventory;
    private final PaymentPort payments;
    private final ConcurrentHashMap<HoldId, BookingId> bookingsByHold = new ConcurrentHashMap<>();
    private final Clock clock;

    public BookingService(SeatInventory inventory, PaymentPort payments, Clock clock) {
        this.inventory = inventory;
        this.payments = payments;
        this.clock = clock;
    }

    public HoldResult hold(ShowId showId, Set<SeatId> seats, String userId, Duration ttl) {
        return inventory.hold(showId, seats, userId, ttl);
    }

    /**
     * Idempotent by holdId. A user mashing "Pay" or a client retrying a timed-out
     * request must produce ONE booking and ONE charge — the same discipline as Q1.
     */
    public ConfirmResult confirm(SeatHold hold, String paymentToken) {
        // Fast path: this hold was already turned into a booking.
        BookingId existing = bookingsByHold.get(hold.id());
        if (existing != null) return new ConfirmResult.Confirmed(existing);

        Instant now = clock.instant();
        if (hold.isExpiredAt(now)) {
            return new ConfirmResult.Failed(ConfirmResult.Reason.HOLD_EXPIRED);
        }
        if (Duration.between(now, hold.expiresAt()).compareTo(MIN_REMAINING_FOR_PAYMENT) < 0) {
            return new ConfirmResult.Failed(ConfirmResult.Reason.HOLD_EXPIRED);
        }

        // The hold is our fence: the seats are already reserved, so it is safe to take
        // the money first. The idempotency key is the holdId, so a retry replays the
        // original charge instead of creating a second one.
        Optional<String> paymentId = payments.charge(hold.id().value(), paymentToken);
        if (paymentId.isEmpty()) {
            inventory.release(hold.id());
            return new ConfirmResult.Failed(ConfirmResult.Reason.PAYMENT_FAILED);
        }

        if (!inventory.confirm(hold.id())) {
            // Rare: the hold expired between the check above and the confirm. We took
            // the money, so we MUST give it back. This window is why the 30s guard
            // exists — it makes this branch rare, but it can never be zero, so the
            // compensating refund is mandatory.
            payments.refund(paymentId.get());
            return new ConfirmResult.Failed(ConfirmResult.Reason.HOLD_EXPIRED);
        }

        BookingId bookingId = new BookingId(java.util.UUID.randomUUID().toString());
        // putIfAbsent, not put: two concurrent confirms of one hold must agree on the
        // SAME booking id. Whoever loses the race returns the winner's id.
        BookingId winner = bookingsByHold.putIfAbsent(hold.id(), bookingId);
        return new ConfirmResult.Confirmed(winner != null ? winner : bookingId);
    }

    public void cancelHold(HoldId holdId) { inventory.release(holdId); }
}
```

```java
// PaymentPort.java
package com.ticketing.booking;

import java.util.Optional;

public interface PaymentPort {
    /** {@code idempotencyKey} is the holdId — one hold can only ever be charged once. */
    Optional<String> charge(String idempotencyKey, String paymentToken);
    void refund(String paymentId);
}
```

### 7.5 The expiry sweeper

```java
// HoldExpirySweeper.java
package com.ticketing.booking;

import java.time.Clock;
import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.TimeUnit;

/**
 * Cleanup only. Correctness lives in the expiry predicate inside hold()/confirm(),
 * so a sweeper outage degrades the seat map's freshness — it does NOT let a seat be
 * double-sold. Never make a background job load-bearing for an invariant.
 */
public final class HoldExpirySweeper implements AutoCloseable {

    private final SeatInventory inventory;
    private final Clock clock;
    private final ScheduledExecutorService executor =
            Executors.newSingleThreadScheduledExecutor(r -> {
                Thread t = new Thread(r, "hold-expiry-sweeper");
                t.setDaemon(true);
                return t;
            });

    public HoldExpirySweeper(SeatInventory inventory, Clock clock) {
        this.inventory = inventory;
        this.clock = clock;
    }

    public void start(long periodSeconds) {
        executor.scheduleAtFixedRate(this::sweep, periodSeconds, periodSeconds, TimeUnit.SECONDS);
    }

    private void sweep() {
        try {
            inventory.expireStaleHolds(clock.instant());
        } catch (RuntimeException e) {
            // MUST swallow: an exception escaping a scheduleAtFixedRate task silently
            // cancels all future runs. This is a classic production incident.
        }
    }

    @Override public void close() { executor.shutdownNow(); }
}
```

> **That `catch` is a real interview point.** If a task submitted to `scheduleAtFixedRate` throws, the executor cancels the schedule permanently and *nothing is logged*. Your sweeper dies quietly at 3 a.m. and nobody notices until the seat map is full of ghosts.

---

## 8. Interleaving Walkthroughs

Draw these. Interleaving diagrams are the most convincing artefact in a concurrency interview.

### 8.1 The bug (no protection)

```
       Alice                          Bob                        Seat A5
       -----                          ---                        -------
  read status  ──────────────────────────────────────────────►   AVAILABLE
                                 read status ───────────────►    AVAILABLE
  status = BOOKED, owner = Alice ─────────────────────────────►  BOOKED(Alice)
                                 status = BOOKED, owner = Bob ►  BOOKED(Bob)  ❌

  Result: two confirmation emails, one seat. Alice's row is silently overwritten.
```

### 8.2 Fixed with a per-show lock

```
       Alice                          Bob
       -----                          ---
  tryLock(show) ✔
  read A5 → AVAILABLE
  A5 = HELD(Alice)          tryLock(show) ⏳ blocked (up to 200 ms)
  unlock ──────────────────►
                            tryLock(show) ✔
                            read A5 → HELD, not expired  → NOT free
                            → Rejected({A5})   ✅ correct, and it's a normal result
                            unlock
```

Note Bob gets a *result*, not an error. Contention is expected traffic, not an exception.

### 8.3 Fixed with the conditional UPDATE (multi-server)

```
   Tx1 (server A)                          Tx2 (server B)
   --------------                          --------------
   UPDATE ... WHERE seat IN (A5,A6)
              AND status='AVAILABLE'
   → row locks on A5, A6
   → rows = 2  == requested 2  ✅
                                           UPDATE ... WHERE seat IN (A6,A7)
                                                      AND status='AVAILABLE'
                                           ⏳ blocks on A6's row lock
   COMMIT ────────────────────────────────►
                                           (re-evaluates predicate under READ COMMITTED)
                                           A6 is now HELD → predicate fails for A6
                                           → rows = 1  != requested 2  ❌
                                           ROLLBACK → Rejected   ✅ all-or-nothing
```

The row-count comparison is doing all the work. No application lock exists anywhere in this diagram.

### 8.4 Expiry vs confirm — the nastiest race

```
   t=0:00   Alice holds A5, expiresAt = 10:00
   t=9:59   Alice's client sends confirm
   t=10:00  hold expires
   t=10:00  Bob's hold() runs: predicate sees status='HELD' AND hold_expires_at < now
            → treats A5 as available → Bob now holds A5 with a NEW hold_id
   t=10:01  Alice's confirm arrives:
            UPDATE ... WHERE hold_id = alice AND status='HELD' AND hold_expires_at >= now
            → 0 rows (hold_id no longer matches) → confirm returns false  ✅
            → BookingService refunds Alice's charge
```

Two defences did the work here: the `hold_id` predicate (ownership) and the `hold_expires_at` predicate (freshness). Checking only expiry would let Alice confirm a seat that Bob now owns. **Always check ownership *and* freshness in the same predicate.**

The `MIN_REMAINING_FOR_PAYMENT` guard shrinks this window but can't eliminate it — which is why the compensating refund exists. Say that explicitly: *"you can shrink a race window with a guard, but if you can't close it, you need a compensating action."*

### 8.5 Deadlock with per-seat locks, and the fix

```
❌ Unordered acquisition
   Alice wants {A5, A6}          Bob wants {A6, A5}
   lock(A5) ✔                    lock(A6) ✔
   lock(A6) ⏳                    lock(A5) ⏳          → circular wait, both hang

✅ Sorted acquisition — one line of code
   Alice: seats.sort() → [A5, A6]   Bob: seats.sort() → [A5, A6]
   Alice: lock(A5) ✔ lock(A6) ✔     Bob: lock(A5) ⏳ … proceeds after Alice
   → the wait-for graph can never contain a cycle
```

Same reasoning applies to `SELECT … FOR UPDATE … ORDER BY seat_id` at the database level.

### 8.6 Duplicate confirm (idempotency)

```
   Request 1: confirm(hold=H1) ──► charge(key=H1) → pay_abc → inventory.confirm → BOOKED
                                   ──► putIfAbsent(H1, B1) → null → returns B1
   (client times out and retries)
   Request 2: confirm(hold=H1) ──► bookingsByHold.get(H1) = B1 → return B1 immediately
                                   ✅ no second charge, no second booking, same id returned
```

Even if both requests raced past the fast path, `charge` is keyed on `H1` (the gateway replays the original charge) and `putIfAbsent` makes both callers agree on one `BookingId`. Layered defences again.

---

## 9. Extensibility

| Want | How | Concurrency impact |
|---|---|---|
| **General admission** (no seat selection) | `UPDATE show SET remaining = remaining - :n WHERE id = :id AND remaining >= :n` | Single-row CAS. Much easier — but the row becomes a hot spot; shard the counter into N buckets if needed |
| **Waiting list** | `BlockingQueue` per show; a released hold hands seats to the head | Producer/consumer; watch for unbounded queues |
| **Virtual queue / lobby** | `Semaphore` admits K users into the booking flow at a time | Turns a stampede into a stream — the real-world answer for on-sales |
| **Seat locking preview** ("someone is looking at this seat") | Short soft holds, best-effort, no correctness role | Deliberately unreliable — must never gate the real hold |
| **Multi-region** | Partition by show; each show has a single home region | Avoids cross-region consensus entirely — partitioning beats coordination |
| **Dynamic pricing** | Price snapshot pinned into the hold at creation | Prevents a price change mid-checkout |
| **Extend a hold** | `UPDATE … SET hold_expires_at = ? WHERE hold_id = ? AND status='HELD' AND hold_expires_at >= ?` | Same conditional-update pattern; cap total extensions to prevent squatting |
| **Redis fast path** | Check availability in Redis before hitting the DB | Optimisation only — the DB stays the arbiter (§4.10) |

---

## 10. Failure Scenarios

| # | Scenario | Handling |
|---|---|---|
| 1 | Two users book the same seat simultaneously | Conditional UPDATE row-count check; loser gets `Rejected` |
| 2 | User abandons checkout | Hold expires by TTL; lazy predicate frees the seat even before the sweeper runs |
| 3 | App server crashes holding an in-JVM lock | JVM death releases nothing meaningful — which is exactly why holds live in the DB, not in a lock object |
| 4 | Sweeper is down for an hour | Seat map looks stale; **bookings remain correct** because expiry is in the query predicate |
| 5 | Payment succeeds, confirm fails (hold expired) | Compensating refund; `MIN_REMAINING_FOR_PAYMENT` makes it rare |
| 6 | Payment times out — did it charge? | Idempotency key = `holdId`; reconcile with the gateway (see Q1 §9.5) before retrying |
| 7 | Client retries confirm | `bookingsByHold` fast path + `uq_booking_hold` constraint; one booking, one charge |
| 8 | DB deadlock under load | Caught via SQLState `40P01`/`40001`/MySQL 1213 → return `Busy` → client retries with jittered backoff |
| 9 | Postgres serialization failure at RR/SERIALIZABLE | Same path — **expected**, must be retried, must not be alerted on |
| 10 | Clock skew across servers | Expiry compares against **database** `now()`, not app-server clocks. Single time source |
| 11 | Thundering herd at on-sale | Bounded thread pool + `tryLock(200ms)` + virtual queue/`Semaphore`; shed load rather than collapse |
| 12 | Hot-row contention on one show | Contention is inherent; mitigate with a virtual queue. Do **not** shard a seat row — it's the correctness anchor |
| 13 | Someone bypasses the service layer | `uq_seat_per_show` unique constraint rejects it at the storage layer |
| 14 | Sweeper task throws | Caught inside the task — otherwise `scheduleAtFixedRate` silently cancels all future runs |
| 15 | Interrupted while waiting on `tryLock` | Catch `InterruptedException`, **restore the interrupt flag**, return `Busy` |

---

## 11. Testing Concurrent Code

Concurrency bugs don't show up in ordinary unit tests — they need contention manufactured on purpose.

### 11.1 The stress test that actually catches double-booking

```java
@Test
void onlyOneOfManyConcurrentBookersWinsTheSameSeat() throws Exception {
    int threads = 200;
    var inventory = new InMemorySeatInventory(Clock.systemUTC());
    var showId = new ShowId("show-1");
    inventory.registerShow(showId, Set.of(new SeatId("A5")));

    // A latch makes all threads start at the SAME instant. Without it they run
    // sequentially and the test passes even against completely broken code.
    var startGate = new CountDownLatch(1);
    var doneGate  = new CountDownLatch(threads);
    var granted   = new AtomicInteger();
    var pool      = Executors.newFixedThreadPool(threads);

    for (int i = 0; i < threads; i++) {
        int user = i;
        pool.submit(() -> {
            try {
                startGate.await();                       // everyone blocks here
                HoldResult r = inventory.hold(showId, Set.of(new SeatId("A5")),
                                              "user-" + user, Duration.ofMinutes(10));
                if (r instanceof HoldResult.Granted) granted.incrementAndGet();
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            } finally {
                doneGate.countDown();
            }
        });
    }

    startGate.countDown();                               // fire
    assertTrue(doneGate.await(10, TimeUnit.SECONDS));
    pool.shutdownNow();

    assertEquals(1, granted.get());                      // ⭐ exactly one winner
}
```

> **Say this:** *"A passing concurrency test proves nothing on its own — it only proves the bug didn't happen this time. So I run it with high thread counts, repeat it (`@RepeatedTest(100)`), and vary timing. For real confidence I'd use jcstress, which enumerates interleavings, or a deterministic scheduler."*

### 11.2 The full matrix

| Test | Technique |
|---|---|
| No double-booking | `CountDownLatch` start gate, N threads, assert exactly 1 winner |
| All-or-nothing | Two threads request overlapping seat sets; assert one gets *all*, the other gets *none* — never a partial |
| Deadlock freedom | Two threads book `{A,B}` and `{B,A}` in a loop 10 000×; assert the test finishes within a timeout |
| Expiry correctness | `Clock.fixed`, advance past TTL, assert the seat is re-holdable and the old hold's confirm fails |
| Confirm-after-expiry | Advance the clock between hold and confirm; assert `false` and assert a refund was issued |
| Idempotent confirm | Call confirm twice concurrently; assert one charge and identical `BookingId` |
| Sweeper resilience | Make `expireStaleHolds` throw; assert subsequent sweeps still run |
| DB-level correctness | **Testcontainers** with a real Postgres/MySQL — H2 does not reproduce row-locking, gap locks or deadlock behaviour |
| Deadlock retry path | Force a deadlock in the real DB; assert `Busy` is returned and the retry succeeds |
| Memory-model bugs | **jcstress** for anything relying on `volatile`/publication |
| Load behaviour | 10 000 concurrent holds against 100 seats; assert exactly 100 granted, p99 latency bounded, zero errors |

The row that earns the most credit: **"H2 won't catch this; I'd use Testcontainers against the real engine, because row-locking and deadlock semantics are engine-specific."**

---

## 12. Interview Cheat Sheet

### 12.1 The 60-second opener

> "The core of this is a check-then-act race: `if (available) book()` is not atomic, so two threads both pass the check and the seat is sold twice. I need to fuse the check and the act into one atomic operation. In a single JVM I'd do that with a `ReentrantLock` per show and `tryLock` with a timeout so requests fail fast. But in-process locks are worthless across app servers, so the real answer is to let the database do it: a single conditional `UPDATE … SET status='HELD' … WHERE status='AVAILABLE' OR hold_expires_at < now()`, then compare the affected row count to the number of seats requested — that's a compare-and-swap over a set of rows, with all-or-nothing and lazy expiry falling out for free. On top of that I put a unique constraint on `(show_id, seat_id)` as the last line of defence, because locks give me good behaviour and constraints give me correctness. Holds expire on wall-clock time rather than lock ownership, since the lock holder is a human who might walk away, and confirm is idempotent keyed on the hold id."

### 12.2 Lines that earn points

- "Check-then-act isn't atomic. Everything in this design exists to close that gap."
- "Optimistic locking with a version column *is* compare-and-swap, run by the database."
- "`volatile` gives visibility and ordering, not atomicity. `count++` is still broken."
- "Locks give me good behaviour; the unique constraint gives me correctness. I want both."
- "READ COMMITTED does not prevent a lost update. Isolation level alone is not a solution."
- "Sorted lock acquisition breaks circular wait — that's one line of code for deadlock freedom."
- "A hold is owned by a user, not a thread, so it expires by timestamp, not by `unlock()`."
- "Expiry is in the query predicate, so the sweeper is an optimisation, not a correctness dependency."
- "A distributed lock with a TTL isn't mutual exclusion — a GC pause gives you two holders."
- "Contention is expected traffic, so `Busy` is a result type, not an exception."
- "Serialization failures and deadlock victims are normal under load. Retry them; don't alert on them."

### 12.3 Common traps

| Trap | Wrong answer | Right answer |
|---|---|---|
| "How do you prevent double booking?" | "`synchronized` on the book method" | Global lock, and useless across servers. Conditional UPDATE + row-count check. |
| "Multiple servers?" | "Redis lock" | Not a correctness mechanism — TTL + GC pause = two holders. DB is the arbiter. |
| "How do holds expire?" | "A background job frees them" | Lazy expiry in the predicate; the job is cleanup. Never make a cron job load-bearing. |
| "Is `volatile` enough?" | "Yes, it's thread-safe" | Visibility ≠ atomicity. Read-modify-write still races. |
| "Isolation level?" | "READ COMMITTED is fine" | It permits lost updates. You need `FOR UPDATE`, a version predicate, or a constraint. |
| "What if two seats are requested?" | Lock them one at a time | Deadlock unless sorted. Or use one statement and dodge the problem. |
| "Confirm after payment?" | "Just mark it booked" | Must re-check ownership **and** freshness; refund if the hold died. |
| "How do you test this?" | "Unit test with two threads" | Start gate, high thread count, repeats, Testcontainers, jcstress. |

### 12.4 If they push further

- **Hot show / on-sale stampede:** virtual queue admitting K users at a time (`Semaphore`); it converts a stampede into a stream. This is what Ticketmaster actually does.
- **Sharding:** partition by `show_id` so each show has one home shard — a seat row must never be split, it's the correctness anchor.
- **Event sourcing:** append `SeatHeld` / `SeatBooked` events; the invariant becomes a uniqueness check on the stream. More auditable, more machinery.
- **CRDTs?** No. Seat allocation is a uniqueness constraint, and uniqueness fundamentally requires coordination — that's a consequence of CAP, and it's worth saying so plainly.

### 12.5 Complexity

| Operation | Cost |
|---|---|
| `hold` (in-memory) | O(k) under one lock, k = seats requested |
| `hold` (JDBC) | 1 statement, O(k) index lookups on the PK |
| `confirm` | O(k), single conditional UPDATE |
| `availableSeats` | O(seats in show), lock-free, cacheable |
| `expireStaleHolds` | O(expired rows) via the `(status, hold_expires_at)` index |
| Lock memory | O(active shows) with `ConcurrentHashMap` striping |

---

*End of Q3.*
