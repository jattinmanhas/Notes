# Go Concurrency — Notes & Interview Prep

A reference doc covering Go's concurrency model: goroutines, channels, sync primitives, context, common patterns, the memory model, and pitfalls — with examples, reasoning, real-world use cases, and practice questions.

## Table of Contents

1. [Goroutines](#1-goroutines)
2. [Channels](#2-channels)
3. [The select Statement](#3-the-select-statement)
4. [The sync Package](#4-the-sync-package)
5. [sync/atomic](#5-syncatomic)
6. [The Context Package](#6-the-context-package)
7. [Common Concurrency Patterns](#7-common-concurrency-patterns)
8. [Memory Model & Data Races](#8-memory-model--data-races)
9. [Common Pitfalls](#9-common-pitfalls)
10. [Graceful Shutdown — Putting It Together](#10-graceful-shutdown--putting-it-together)
11. [Practice Questions](#11-practice-questions)

---

## 1. Goroutines

### What & Why

A goroutine is a function that runs concurrently, scheduled by the Go runtime rather than the OS. You start one with `go someFunc()`.

The key difference from OS threads: an OS thread typically reserves ~1-2MB of stack and is scheduled by the kernel. A goroutine starts with a tiny stack (~2KB) that grows and shrinks dynamically, and is scheduled by Go's own runtime scheduler, not the kernel. This is *why* you can spin up hundreds of thousands of goroutines without crushing memory, where the same count of OS threads would.

Go uses the **GMP model** to schedule goroutines:
- **G** (Goroutine) — the unit of work
- **M** (Machine) — an OS thread
- **P** (Processor) — a logical context that holds a local run queue of goroutines and is required for an M to execute G's

The scheduler multiplexes many G's onto a smaller number of M's, using P's to manage local run queues and work-stealing when one P runs dry — an idle P will steal goroutines from another P's queue rather than sit empty. When a goroutine blocks on a syscall, the runtime detaches the M from its P and hands that P to another M (or spins up a new one), so other goroutines keep running instead of stalling behind a blocked thread. This is why Go achieves high concurrency without you manually managing thread pools.

The number of P's is controlled by **`GOMAXPROCS`**, which defaults to the number of logical CPUs available. It caps how many goroutines can run *truly in parallel* at once — more P's doesn't mean more goroutines exist, just more of them can execute simultaneously rather than queuing.

### Example

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	var wg sync.WaitGroup
	for i := 0; i < 5; i++ {
		wg.Add(1)
		go func(id int) {
			defer wg.Done()
			fmt.Println("worker", id, "running")
		}(i)
	}
	wg.Wait()
}
```

### Best Use Cases

- Handling each incoming HTTP request on its own goroutine (Go's `net/http` does this for you already)
- Spinning off independent units of work that don't block each other — e.g. processing each Kafka message in a worker pool goroutine, like in your job queue consumer
- Background tasks: cache warming, metrics flushing, periodic cleanup

### Interview Angle

Common probes: "why are goroutines cheaper than threads," "what happens to a goroutine when `main()` returns" (it's killed mid-flight, finished or not — Go doesn't wait for orphaned goroutines), "explain GMP," and "what does `GOMAXPROCS` control." Interviewers like to see you understand that goroutines aren't free — they still need synchronization and can still leak.

---

## 2. Channels

### What & Why

Channels are the primitive Go gives you to let goroutines communicate and synchronize, instead of sharing memory directly and protecting it with locks. The philosophy, often quoted: *"don't communicate by sharing memory; share memory by communicating."* A channel is type-safe, and using one to pass data also passes ownership — the receiving goroutine becomes the one safely allowed to read/write that data next.

**Unbuffered channels** (`make(chan int)`) block the sender until a receiver is ready, and vice versa. This makes them a synchronization point, not just a data pipe — a send/receive pair on an unbuffered channel is effectively a handshake.

**Buffered channels** (`make(chan int, n)`) let you send up to `n` items without a receiver being ready yet. They decouple sender and receiver timing, but they don't remove backpressure — once full, the sender blocks again, same as an unbuffered channel would.

**Directional channels** (`chan<- int` send-only, `<-chan int` receive-only) exist purely as a compile-time contract — they don't change runtime behavior, but they make function signatures self-documenting and prevent a function from accidentally reading from a channel it should only write to (or vice versa).

**Nil channels** block forever on both send and receive — this seems useless until you see it used deliberately in `select` to "disable" a case at runtime (assigning a case's channel to `nil` effectively removes it from consideration).

**Closed channels**: closing a channel signals "no more values are coming." Receiving from a closed channel returns the zero value immediately along with `ok == false`. Sending on a closed channel panics. This is why the convention is: the sender closes, never the receiver — the receiver has no way of knowing if more sends are still coming, but the sender always does.

### Example

```go
package main

import "fmt"

func main() {
	ch := make(chan int, 2) // buffered, capacity 2
	ch <- 1
	ch <- 2
	close(ch)

	for v := range ch { // range exits automatically when channel is closed and drained
		fmt.Println(v)
	}

	// Reading from a closed, drained channel
	v, ok := <-ch
	fmt.Println(v, ok) // 0 false
}
```

### Best Use Cases

- Unbuffered: signaling exact handshake points — e.g. "job has been picked up by a worker"
- Buffered: smoothing out bursty producers, like a buffered channel sitting between your Kafka consumer and your worker pool so a burst of messages doesn't block the consumer goroutine
- Closing channels to broadcast "done" to multiple listening goroutines (since a closed channel always returns immediately, every receiver unblocks)

### Interview Angle

Very common: "what happens if you send on a closed channel," "what happens if you close an already-closed channel" (panic, both cases), "difference between buffered/unbuffered," and "how would you broadcast a stop signal to N goroutines using a channel" (close it — don't send on it N times).

---

## 3. The select Statement

### What & Why

`select` lets a goroutine wait on multiple channel operations at once, proceeding with whichever is ready first. It exists because without it, you'd have to poll channels one at a time, which either blocks on the wrong one or wastes CPU spinning. `select` makes "wait on whichever of these is ready" a first-class, efficient operation handled by the runtime.

If multiple cases are ready simultaneously, Go picks one **pseudo-randomly** — this is intentional, to prevent one channel from being silently starved by always losing to another in a fixed priority order.

A `default` case makes a `select` non-blocking: if no channel is ready, it runs immediately instead of waiting. This is the idiomatic way to "check without waiting."

### Example — timeout pattern

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	ch := make(chan string)

	go func() {
		time.Sleep(2 * time.Second)
		ch <- "result"
	}()

	select {
	case res := <-ch:
		fmt.Println("got:", res)
	case <-time.After(1 * time.Second):
		fmt.Println("timed out")
	}
}
```

### Best Use Cases

- Implementing timeouts on a blocking operation (as above) — directly relevant if you've ever needed "wait for a worker result, but give up after N seconds"
- Listening on both a "work" channel and a "cancel"/"done" channel simultaneously, so a goroutine can be told to stop mid-loop
- Building a non-blocking check (`select` with `default`) on whether a channel has data without committing to wait

### Interview Angle

Expect: "what happens if two cases are ready at once," "how do you implement a timeout without `context`," and "how do you do a non-blocking channel read." Interviewers like to see you reach for `select` + `default` instead of awkward polling loops.

---

## 4. The sync Package

### What & Why

Channels are great for handing off data and signaling, but sometimes you just need to protect a shared piece of state (a map, a counter, a slice) from concurrent access — and a channel would be awkward or overkill for that. `sync` gives you primitives for that case.

- **`sync.Mutex`** — mutual exclusion lock. Only one goroutine can hold it at a time. Use it when you have a critical section of code (usually reading and writing shared state together) that must not be interleaved.
- **`sync.RWMutex`** — allows many concurrent readers OR one writer, never both. Use it when reads vastly outnumber writes — e.g. a config struct read on every request but updated rarely. A plain `Mutex` here would unnecessarily serialize reads that don't actually conflict with each other.
- **`sync.WaitGroup`** — lets one goroutine wait for a set of others to finish. `Add(n)` before launching, `Done()` (usually deferred) inside each goroutine, `Wait()` to block until the count hits zero. It exists because "wait for N independent things to finish" doesn't map naturally onto channels without extra bookkeeping a `WaitGroup` does for you.
- **`sync.Once`** — guarantees a function runs exactly once, no matter how many goroutines call it concurrently. Classic use: lazy-initializing a singleton (e.g. a DB connection pool) safely under concurrent access.
- **`sync.Cond`** — lets goroutines wait for a condition to become true, broadcasting when it changes. Rarely used in everyday Go code (channels usually cover the same need more idiomatically) but interviewers sometimes ask about it to test whether you know it exists.
- **`sync.Map`** — a map safe for concurrent use without an external lock, optimized specifically for two patterns: keys written once and read many times, or many goroutines operating on disjoint sets of keys. For general-purpose concurrent maps it's often *not* faster than a plain `map` + `Mutex` — worth knowing because it's a common interview trap ("just use `sync.Map`, it's faster" is not always true).

**Why channels vs. sync?** Rule of thumb: use channels when you're passing ownership of data or coordinating a sequence of events; use `sync` primitives when multiple goroutines need to read/write the *same* piece of shared state in place.

### Example — Mutex protecting shared state

```go
package main

import (
	"fmt"
	"sync"
)

type Counter struct {
	mu    sync.Mutex
	count int
}

func (c *Counter) Inc() {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.count++
}

func main() {
	c := &Counter{}
	var wg sync.WaitGroup
	for i := 0; i < 1000; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			c.Inc()
		}()
	}
	wg.Wait()
	fmt.Println(c.count) // always 1000
}
```

### Best Use Cases

- `Mutex`: protecting an in-memory rate limiter's sliding-window counters if you're not delegating that to Redis
- `RWMutex`: a cache of feature flags or config read on every request, refreshed occasionally
- `WaitGroup`: fanning out work to N goroutines (e.g. N workers each processing a batch from the queue) and waiting for all to finish before returning
- `Once`: safely initializing a Kafka producer/consumer client exactly once even if multiple goroutines might trigger setup concurrently

### Interview Angle

Very common: "Mutex vs channel — when would you use each," "what's a race condition and how does Mutex prevent it," "what happens if you forget to Unlock" (deadlock), "why RWMutex over Mutex," and "is `sync.Map` always faster than `map` + `Mutex`" (no — only for its specific access patterns).

---

## 5. sync/atomic

### What & Why

For very simple operations — incrementing a counter, swapping a pointer, flipping a flag — taking a full `Mutex` lock is often more overhead than the operation needs. `sync/atomic` provides operations (`Add`, `Load`, `Store`, `CompareAndSwap`) that the CPU executes as a single, indivisible instruction, with no goroutine able to observe a half-finished state. This makes them cheaper than a Mutex for these narrow cases, because there's no lock to acquire, hold, and release — just one atomic instruction.

The trade-off is scope: atomics only protect a *single variable's* operation. The moment your critical section involves more than one variable, or a read-then-decide-then-write sequence that needs to be treated as one unit, you're back to needing a `Mutex` — atomics can't express "do these three things as one atomic block."

### Example

```go
package main

import (
	"fmt"
	"sync"
	"sync/atomic"
)

func main() {
	var count int64
	var wg sync.WaitGroup

	for i := 0; i < 1000; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			atomic.AddInt64(&count, 1)
		}()
	}
	wg.Wait()
	fmt.Println(atomic.LoadInt64(&count)) // always 1000, no Mutex needed
}
```

### Best Use Cases

- High-frequency counters: requests-served, jobs-processed, active-worker-count metrics in your job queue, where a Mutex would add unnecessary contention under heavy concurrent increments
- A simple `running` or `shuttingDown` flag read by many goroutines and set once by a shutdown handler
- Lock-free swapping of an immutable config pointer (`atomic.Value` / `atomic.Pointer[T]`) so readers never block on a writer publishing a new version

### Interview Angle

A favorite follow-up after Mutex questions: "when would you use `atomic` instead of `Mutex`, and when does that stop being enough?" Good answers center on single-variable-only scope, and that reaching for atomics everywhere "for performance" without profiling first is premature optimization.

---

## 6. The Context Package

### What & Why

`context.Context` exists to propagate **cancellation signals, deadlines, and request-scoped values** through a call chain — especially across goroutine and API boundaries. Without it, if a client disconnects or a request times out, every downstream goroutine, DB call, and HTTP call it spawned would keep running with no way to know it should stop. `context` gives every layer a consistent way to check "should I still be doing this?"

Core idea: a `Context` is **immutable** — you don't modify one, you derive a child from it (`context.WithCancel`, `context.WithTimeout`, `context.WithDeadline`, `context.WithValue`). Cancelling a parent cancels all its children, automatically, recursively — this propagation is the entire point of the API being built around derivation rather than mutation.

- **Cancellation**: `WithCancel` gives you a `cancel()` function; calling it closes an internal channel that `ctx.Done()` returns, which every downstream function should be selecting on.
- **Timeouts/Deadlines**: `WithTimeout`/`WithDeadline` auto-cancel after a duration/time — useful so a slow downstream call (DB, Kafka, another service) doesn't hang the whole chain forever.
- **Values**: `WithValue` attaches request-scoped data (trace IDs, auth claims) — but it's explicitly *not* meant for passing optional parameters or business logic data; overusing it is considered an anti-pattern because it bypasses Go's type safety and makes data flow implicit instead of explicit in function signatures.

Why it matters in service chains specifically: imagine an HTTP handler that calls a DB query that calls an external API. If the original HTTP request is cancelled (client closed the tab), `context` is what lets that cancellation propagate all the way down so you're not wasting a DB connection and an external API call on a request nobody's waiting for anymore.

**`ctx.Err()` vs `context.Cause(ctx)`**: `ctx.Err()` tells you *that* a context was cancelled (`context.Canceled` or `context.DeadlineExceeded`), but not *why* if it was cancelled manually with a custom reason. `context.WithCancelCause` (Go 1.21+) lets you attach a specific reason, retrievable via `context.Cause(ctx)` — useful in larger systems where "the request timed out" and "an upstream explicitly cancelled this" need to be distinguishable in logs.

### Example

```go
package main

import (
	"context"
	"fmt"
	"time"
)

func doWork(ctx context.Context) error {
	select {
	case <-time.After(3 * time.Second):
		fmt.Println("work done")
		return nil
	case <-ctx.Done():
		return ctx.Err() // context.DeadlineExceeded or context.Canceled
	}
}

func main() {
	ctx, cancel := context.WithTimeout(context.Background(), 1*time.Second)
	defer cancel() // always call cancel to release resources, even on success

	if err := doWork(ctx); err != nil {
		fmt.Println("error:", err)
	}
}
```

### Best Use Cases

- Every request-handling function in an HTTP/gRPC service should accept `ctx context.Context` as its first parameter
- Bounding how long a goroutine waits on a Kafka publish or DB query before giving up, so one slow dependency can't cascade into a stuck worker pool
- Propagating a `traceID` or `requestID` through your IAM/auth middleware chain down to logging, for tracing a request across services

### Interview Angle

Frequently asked: "why is `Context` immutable / why derive instead of mutate," "what happens if you don't call `cancel()`" (resource leak — the parent's internal timer/goroutine tracking that context won't be released until it naturally expires or is GC'd, and anything still selecting on `ctx.Done()` could leak), and "is `WithValue` good practice for passing data?" (generally no — use it sparingly for cross-cutting concerns only).

---

## 7. Common Concurrency Patterns

### Worker Pool

**What & why**: a fixed number of goroutines pull work from a shared channel, instead of spawning one goroutine per task. This bounds concurrency — critical when the task involves a limited resource (DB connections, a downstream API's rate limit) where unbounded goroutines would overwhelm it rather than help.

```go
func workerPool(jobs <-chan int, results chan<- int, workerCount int) {
	var wg sync.WaitGroup
	for w := 0; w < workerCount; w++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for j := range jobs {
				results <- j * 2 // pretend this is real work
			}
		}()
	}
	wg.Wait()
	close(results)
}
```

This is directly the shape of a Kafka consumer worker pool — N goroutines pulling messages off a channel fed by your Kafka consumer, instead of one goroutine per message (which would be unbounded and could overwhelm downstream DB writes).

### Channel-as-Semaphore (Concurrency Limiter)

**What & why**: sometimes you don't want a fixed pool of long-lived workers — you want to cap how many goroutines run *at once* among a dynamically varying set of tasks. A buffered channel used purely as a token bucket does this: acquiring a slot is sending into the channel, releasing is receiving from it. If the channel is full, the next send blocks until a slot frees up — exactly the semaphore behavior, with no separate library needed.

```go
func processAll(items []string, maxConcurrent int) {
	sem := make(chan struct{}, maxConcurrent) // capacity = concurrency limit
	var wg sync.WaitGroup

	for _, item := range items {
		wg.Add(1)
		sem <- struct{}{} // acquire a slot (blocks if full)
		go func(it string) {
			defer wg.Done()
			defer func() { <-sem }() // release the slot
			process(it)
		}(item)
	}
	wg.Wait()
}
```

This differs from a worker pool in that goroutines are still spawned per item — the channel just throttles how many run concurrently, rather than reusing a fixed set of long-lived workers.

### errgroup — WaitGroup with Error Propagation

**What & why**: `sync.WaitGroup` waits for goroutines to finish but gives you no clean way to capture the first error among them or cancel the rest once one fails. `golang.org/x/sync/errgroup` solves exactly this: it's a WaitGroup that also collects the first non-nil error and, when constructed with `errgroup.WithContext`, cancels a shared context the moment any goroutine returns an error — so siblings doing unrelated work can check `ctx.Done()` and bail out early instead of finishing pointlessly.

```go
import "golang.org/x/sync/errgroup"

func fetchAll(ctx context.Context, ids []int) ([]Result, error) {
	g, ctx := errgroup.WithContext(ctx)
	results := make([]Result, len(ids))

	for i, id := range ids {
		i, id := i, id // avoid loop-variable capture (pre-1.22 habit, still common to see)
		g.Go(func() error {
			res, err := fetchOne(ctx, id)
			if err != nil {
				return err // first error cancels ctx for the rest of the group
			}
			results[i] = res
			return nil
		})
	}

	if err := g.Wait(); err != nil {
		return nil, err
	}
	return results, nil
}
```

This is the pattern you'd reach for instead of a raw `WaitGroup` whenever the parallel work can fail and you need to know about it — e.g. fetching several dependent resources for one request, where one failure should abort the rest.

### Fan-Out / Fan-In

**What & why**: fan-out means distributing work across multiple goroutines; fan-in means merging multiple result channels back into one. Useful when a single producer can't keep multiple workers fed fast enough, or when you need to combine results from several independent sources.

```go
func fanIn(channels ...<-chan int) <-chan int {
	out := make(chan int)
	var wg sync.WaitGroup
	for _, c := range channels {
		wg.Add(1)
		go func(ch <-chan int) {
			defer wg.Done()
			for v := range ch {
				out <- v
			}
		}(c)
	}
	go func() {
		wg.Wait()
		close(out)
	}()
	return out
}
```

### Pipeline

**What & why**: chaining stages together via channels, where each stage's output channel is the next stage's input. This decomposes a multi-step transformation into independently testable, independently scalable stages — e.g. a job queue's stages could be `dequeue -> validate -> process -> persist result`, each as its own goroutine(s) connected by channels.

### Pub-Sub

**What & why**: one publisher, many subscribers, each subscriber getting its own copy of every message. A single channel can't do this natively (one message goes to exactly one receiver), so pub-sub is usually built with a registry of subscriber channels that the publisher fans a message out to.

### Rate Limiting

**What & why**: bounding how often an action can happen, typically using `time.Ticker` (token bucket-ish) in pure Go, or — as in your job queue — a sliding-window counter in Redis when the limiter needs to be shared across multiple service instances rather than just within one process's memory.

```go
limiter := time.NewTicker(200 * time.Millisecond) // 5 requests/sec
defer limiter.Stop()

for req := range requests {
	<-limiter.C // blocks until next tick
	go handle(req)
}
```

**Best use cases recap**: worker pools and pipelines for Kafka message processing, the semaphore pattern for capping concurrency on bursty/dynamic workloads, `errgroup` for parallel calls that can fail, fan-in for aggregating results from parallel DB queries, and rate limiting for protecting downstream APIs or enforcing per-user quotas (in-process via ticker, distributed via Redis as you've already implemented).

### Interview Angle

"Design a worker pool from scratch" is one of the most common live-coding asks for backend roles. Also: "how would you limit concurrency to N when processing a list of items" (worker pool, or the channel-as-semaphore pattern — knowing both, and when to pick which, is a strong signal), "how would you fetch 5 things in parallel and fail fast if one errors" (`errgroup`), and "what's the difference between rate limiting in-process vs. distributed" — a good one for you to nail given your Redis rate limiter experience.

---

## 8. Memory Model & Data Races

### What & Why

A **data race** happens when two goroutines access the same memory location concurrently, at least one of them writing, with no synchronization between them. The outcome is **undefined** — not just "you might read a stale value," but the compiler/runtime is permitted to do unexpected things, because data races violate the assumptions the Go memory model is built on.

The **Go memory model** defines exactly when a write in one goroutine is *guaranteed* to be visible to a read in another — this guarantee is called a **happens-before relationship**. Without an established happens-before relationship, there's no guarantee the reading goroutine ever sees the write at all (it might see a stale cached value, due to CPU caching and compiler reordering).

Synchronization primitives are what *establish* happens-before relationships:
- A send on a channel happens-before the corresponding receive completes
- A `Mutex.Unlock()` happens-before a subsequent `Mutex.Lock()` (by another goroutine) completes
- A `sync.WaitGroup`'s `Wait()` happens-before it returns, relative to the matching `Done()` calls
- A `go` statement happens-before the goroutine it starts begins executing
- A goroutine's exit does **not** happen-before anything — you can't assume a goroutine has effects visible just because you assume it "must have finished by now"

This is *why* you can't just use a plain `bool` flag with no lock or channel to signal "done" between goroutines — without a happens-before edge, the compiler is allowed to assume no other goroutine touches that variable and can cache it in a register, meaning the other goroutine might never observe the change.

### Detecting races

Go ships a built-in race detector:

```bash
go run -race main.go
go test -race ./...
```

It instruments memory accesses at runtime and flags genuine races. It has overhead (slower, more memory), so it's used in testing/CI, not production.

### Example — a real race

```go
package main

import "fmt"

func main() {
	count := 0
	done := make(chan bool)

	go func() {
		count++ // unsynchronized write
		done <- true
	}()

	count++ // unsynchronized write — race with the goroutine above
	<-done
	fmt.Println(count) // result is undefined: could be 1 or 2, and -race will flag this
}
```

### Best Use Cases

This section is less about "use cases" and more about discipline: always run `-race` in CI for any service with concurrent code, especially around shared caches, counters, or in-memory state in your job queue's worker pool.

### Interview Angle

A favorite: "what is a data race, and is it the same as a deadlock?" (no — a race is about unsynchronized concurrent access with undefined outcome; a deadlock is about goroutines permanently blocked waiting on each other). Also: "how does the Go race detector work" and "show me a happens-before example."

---

## 9. Common Pitfalls

### Goroutine Leaks

**What & why**: a goroutine that blocks forever (e.g. sending on a channel nobody will ever receive from) never gets garbage collected — Go can't know it'll never be needed again, so it just sits there consuming a stack and whatever resources it's holding. Over time, this is a slow memory/resource leak that's easy to miss in development and painful in production.

```go
// LEAK: if nobody ever reads from ch, this goroutine blocks forever
func leaky() {
	ch := make(chan int)
	go func() {
		val := compute()
		ch <- val // blocks forever if main goroutine returns without reading
	}()
}
```

**Fix**: always give a goroutine a way out — a `context` cancellation it selects on, a buffered channel sized so it can't block, or a guarantee the receiver will always show up. In production, tools like `pprof`'s goroutine profile (`/debug/pprof/goroutine`) are the standard way to confirm a suspected leak by watching the goroutine count climb over time instead of staying flat under steady load.

### Deadlocks

**What & why**: two or more goroutines each waiting on something the other holds, so neither can proceed. The simplest case is one goroutine sending on an unbuffered channel with no other goroutine ever receiving — Go's runtime will actually detect this specific "all goroutines asleep" case and panic with `fatal error: all goroutines are asleep - deadlock!`.

```go
func main() {
	ch := make(chan int) // unbuffered
	ch <- 1               // deadlock: nobody is receiving, and main IS the only goroutine
}
```

### Closure-Over-Loop-Variable Bug

**What & why**: this was a famous Go gotcha. In Go versions before 1.22, the loop variable was *reused* across iterations rather than being a fresh variable each time — so a closure capturing it by reference would see whatever value it held when the goroutine actually *ran*, not when it was launched.

```go
// Pre-Go 1.22 bug:
for i := 0; i < 3; i++ {
	go func() {
		fmt.Println(i) // likely prints 3, 3, 3 — not 0, 1, 2
	}()
}

// Fix (pre-1.22): shadow the variable
for i := 0; i < 3; i++ {
	i := i // create a new variable per iteration
	go func() {
		fmt.Println(i)
	}()
}
```

**Note**: Go 1.22 (released Feb 2024) changed loop semantics so each iteration gets its own variable, which fixes this by default. Still worth knowing — it's a classic interview "what does this print" trap, and you may still encounter it in older codebases or see interviewers explicitly ask about pre-1.22 behavior to test if you understand *why* it happened, not just that it did.

### Unbuffered Channel Blocking Gotchas

**What & why**: forgetting that an unbuffered channel send blocks until *someone* receives is a frequent source of accidental deadlocks — e.g. sending results back on an unbuffered channel from inside a `select` with a timeout, where if the timeout fires first, the sending goroutine is stuck forever because nobody will ever receive.

```go
func doWork(ch chan<- int) {
	result := slowCompute()
	ch <- result // if the caller already gave up (timeout), this blocks forever -> leak
}
```

**Fix**: size the channel with a buffer of 1 so the send can't block even if the receiver has moved on, or have the worker also select on a cancellation context.

### Interview Angle

These pitfalls are exactly what interviewers love to put in front of you as "what's wrong with this code" snippets — especially the loop-variable closure bug and the unbuffered-channel-with-timeout leak. Being able to *name* the bug class (goroutine leak vs. deadlock vs. race) quickly is often what separates a strong answer from a so-so one.

---

## 10. Graceful Shutdown — Putting It Together

### What & Why

This pattern ties together `context`, `WaitGroup`/`errgroup`, channels, and OS signal handling into the thing you actually need in production: when a service receives `SIGTERM` (e.g. during a deploy or pod eviction in Kubernetes), it should stop accepting new work, let in-flight work finish (within a bound), and exit cleanly — not just die mid-request. This is exactly what shutting down your job queue's worker pool cleanly looks like: stop pulling new messages from Kafka, let currently-processing jobs finish, then exit.

```go
package main

import (
	"context"
	"fmt"
	"os"
	"os/signal"
	"sync"
	"syscall"
	"time"
)

func main() {
	ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
	defer stop()

	jobs := make(chan int, 10)
	var wg sync.WaitGroup

	// Start workers
	for w := 0; w < 3; w++ {
		wg.Add(1)
		go func(id int) {
			defer wg.Done()
			for {
				select {
				case job, ok := <-jobs:
					if !ok {
						return // channel closed, no more work
					}
					fmt.Printf("worker %d processing job %d\n", id, job)
					time.Sleep(500 * time.Millisecond)
				case <-ctx.Done():
					fmt.Printf("worker %d shutting down\n", id)
					return
				}
			}
		}(w)
	}

	// Simulate producing jobs until shutdown signal
	go func() {
		for i := 0; ; i++ {
			select {
			case jobs <- i:
			case <-ctx.Done():
				close(jobs)
				return
			}
		}
	}()

	<-ctx.Done()
	fmt.Println("shutdown signal received, draining workers...")
	wg.Wait()
	fmt.Println("clean exit")
}
```

### Best Use Cases

- Any long-running service (HTTP server, Kafka consumer, queue worker) that needs to drain in-flight work instead of dropping it on deploy
- Combining with `http.Server.Shutdown(ctx)` so an HTTP server stops accepting new connections but finishes in-flight requests within a deadline

### Interview Angle

"How would you gracefully shut down a service processing messages from a queue" is a realistic systems-style question that expects you to combine several primitives correctly, not just recite definitions — a strong way to demonstrate you understand how the pieces compose, not just what each one does in isolation.

---

## 11. Practice Questions

1. What's the difference between a goroutine and an OS thread, and why does that difference matter for scalability?
2. Explain the GMP model in your own words. What happens when a goroutine makes a blocking syscall? What does `GOMAXPROCS` actually control?
3. Why does Go favor channels over shared memory + locks as its primary concurrency idiom? When would you reach for `sync.Mutex` instead?
4. What happens when you send on a closed channel? What happens when you close an already-closed channel?
5. What's the difference between a buffered and unbuffered channel in terms of synchronization guarantees, not just "one has a capacity"?
6. What does this print, and why?
   ```go
   ch := make(chan int)
   go func() { ch <- 1 }()
   go func() { ch <- 2 }()
   fmt.Println(<-ch)
   fmt.Println(<-ch)
   ```
7. What's wrong with this code, and how would you fix it?
   ```go
   func process(items []int) {
       var wg sync.WaitGroup
       for _, item := range items {
           wg.Add(1)
           go func() {
               defer wg.Done()
               fmt.Println(item)
           }()
       }
       wg.Wait()
   }
   ```
8. Why is `sync.RWMutex` sometimes preferred over `sync.Mutex`? Give a realistic scenario where it would NOT help.
9. When would you use `sync/atomic` instead of `sync.Mutex`? Where does atomic stop being sufficient?
10. What does `context.WithCancel` actually do internally, in terms of channels? Why must you always call the returned `cancel` function, even on the success path?
11. Explain the difference between a goroutine leak and a deadlock. Can the Go runtime always detect deadlocks?
12. Design a worker pool that processes N jobs with a max concurrency of 5, and returns all results. What are the failure modes you need to handle (a job panics, a job hangs)?
13. How would you limit concurrency to 5 over a dynamically-sized, varying list of tasks without a fixed worker pool? (channel-as-semaphore)
14. What does `errgroup` give you over a plain `sync.WaitGroup`? When would you reach for it specifically?
15. What is a "happens-before" relationship in the Go memory model? Name three things that establish one.
16. Why doesn't a plain `bool` variable work as a "stop signal" between goroutines without a lock or channel?
17. You have a function that calls a downstream API and a DB write inside a `select` with a context timeout — if the timeout fires, what could go wrong with goroutines spawned to do that work, and how do you avoid it?
18. In a system processing messages from Kafka with a worker pool, where would you put backpressure (buffered channel size, number of workers, rate limiter), and what failure mode does each one protect against?
19. Walk through how you'd implement graceful shutdown for a service consuming from a queue — what needs to happen, in what order, when `SIGTERM` is received?
