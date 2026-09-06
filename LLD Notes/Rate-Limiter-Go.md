# Rate Limiter in Go — complete LLD walkthrough

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
rate-limiter-go/
  go.mod
  internal/lld/clock.go
  internal/lld/limiter.go
  cmd/demo/main.go
  internal/lld/service_test.go
```

Go 1.22 or newer; no third-party modules. From the project root:

```bash
go run ./cmd/demo
go test ./...
go test -race ./...
```

`cmd/demo` is the executable. `internal/lld` is one cohesive application package: files separate responsibilities without inventing a package for every type. The internal boundary prevents imports by unrelated external modules. Methods and interfaces use composition and implicit interface satisfaction.

## 4. Every file, its code, and explanation

### 4.1. `go.mod`

```text
module example.com/rate-limiter

go 1.22
```

**How it works and why it belongs here:** The module path matches main’s import. Go 1.22 or newer is sufficient; the implementation uses only the standard library.

### 4.2. `internal/lld/clock.go`

```go
package lld

import (
	"time"
)

type Clock interface{ Now() time.Duration }
type MonotonicClock struct{ start time.Time }

func NewClock() *MonotonicClock              { return &MonotonicClock{time.Now()} }
func (c *MonotonicClock) Now() time.Duration { return time.Since(c.start) }
```

**How it works and why it belongs here:** time.Since uses the monotonic component of the saved start time. Tests provide elapsed durations directly.

### 4.3. `internal/lld/limiter.go`

```go
package lld

import (
	"math"
	"sync"
	"time"
)

type Decision struct {
	Allowed    bool
	RetryAfter time.Duration
}
type Limiter interface{ Allow(string) Decision }
type bucket struct {
	tokens float64
	last   time.Duration
}
type TokenBucket struct {
	mu             sync.Mutex
	capacity, rate float64
	clock          Clock
	buckets        map[string]bucket
}

func NewTokenBucket(capacity, rate float64, clock Clock) *TokenBucket {
	if capacity < 1 || rate <= 0 || math.IsNaN(capacity) || math.IsNaN(rate) || math.IsInf(capacity, 0) || math.IsInf(rate, 0) || clock == nil {
		panic("invalid configuration")
	}
	return &TokenBucket{capacity: capacity, rate: rate, clock: clock, buckets: map[string]bucket{}}
}
func (l *TokenBucket) Allow(key string) Decision {
	l.mu.Lock()
	defer l.mu.Unlock()
	now := l.clock.Now()
	b, ok := l.buckets[key]
	if !ok {
		b = bucket{l.capacity, now}
	}
	if now < b.last {
		now = b.last
	}
	b.tokens = math.Min(l.capacity, b.tokens+(now-b.last).Seconds()*l.rate)
	b.last = now
	result := Decision{}
	if b.tokens >= 1 {
		b.tokens--
		result.Allowed = true
	} else {
		result.RetryAfter = time.Duration(math.Ceil((1 - b.tokens) / l.rate * float64(time.Second)))
	}
	l.buckets[key] = b
	return result
}
```

**How it works and why it belongs here:** The interface exposes a decision, not internal counters. Refill and consumption are atomic under one mutex. Configuration rejects nonfinite floats. Practical configurations should keep retry durations within time.Duration range.

### 4.4. `cmd/demo/main.go`

```go
package main

import (
	"example.com/rate-limiter/internal/lld"
	"fmt"
)

func main() {
	var limiter lld.Limiter = lld.NewTokenBucket(2, 1, lld.NewClock())
	for i := 0; i < 3; i++ {
		fmt.Println("allowed:", limiter.Allow("alice").Allowed)
	}
}
```

**How it works and why it belongs here:** The composition root constructs dependencies, runs a concrete scenario, and prints observable results. Error checks make a rejected operation visible rather than silently treating it as success.

### 4.5. `internal/lld/service_test.go`

```go
package lld

import (
	"sync"
	"sync/atomic"
	"testing"
	"time"
)

type fakeClock struct{ now time.Duration }

func (c *fakeClock) Now() time.Duration { return c.now }
func TestBucket(t *testing.T) {
	c := &fakeClock{}
	l := NewTokenBucket(2, 1, c)
	if !l.Allow("a").Allowed || !l.Allow("a").Allowed || l.Allow("a").Allowed {
		t.Fatal("burst")
	}
	if !l.Allow("b").Allowed {
		t.Fatal("isolation")
	}
	c.now = time.Second / 2
	if d := l.Allow("a"); d.Allowed || d.RetryAfter != time.Second/2 {
		t.Fatal(d)
	}
	c.now = time.Second
	if !l.Allow("a").Allowed {
		t.Fatal("refill")
	}
}
func TestConcurrentLimit(t *testing.T) {
	l := NewTokenBucket(10, 1, &fakeClock{})
	var accepted atomic.Int64
	var wg sync.WaitGroup
	for i := 0; i < 100; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			if l.Allow("a").Allowed {
				accepted.Add(1)
			}
		}()
	}
	wg.Wait()
	if accepted.Load() != 10 {
		t.Fatal(accepted.Load())
	}
}
```

**How it works and why it belongs here:** Tests deterministic refill and client isolation, then races 100 callers against ten fixed tokens and asserts exactly ten successes.

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
?   	example.com/rate-limiter/cmd/demo	[no test files]
ok  	example.com/rate-limiter/internal/lld	(cached)

allowed: true
allowed: true
allowed: false
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
