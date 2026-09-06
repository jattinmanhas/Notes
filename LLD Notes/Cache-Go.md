# Cache in Go — complete LLD walkthrough

Study order: requirements → model and flow → project tree → each complete file and explanation → patterns → walkthrough → limitations and exercises. This is a runnable single-process interview reference, with no external services or omitted source files.

## 1. Requirements and scope

Build a capacity-bounded, thread-safe LRU cache with per-entry TTL. Support put, get and delete. Get promotes a live entry to most recently used. Updating a key replaces its value and expiry. Reject nonpositive capacity and TTL. Use an injected clock so expiration tests need no sleeps. This is an in-process generic cache, with no remote storage, loading or persistence.

## 2. Design and responsibilities

```text
Main -> Cache<K,V>
        -> map key -> linked-list node (Go) / access-ordered map (Java)
        -> recency order: least used -> most used
        -> Clock for TTL
put -> remove expired -> update/insert -> evict least recently used
get -> expire if needed -> promote -> return
```

## 3. Project structure and running

Create each file at the shown relative path. All imports and package declarations are included.

```text
cache-go/
  go.mod
  internal/lld/clock.go
  internal/lld/cache.go
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
module example.com/cache

go 1.22
```

**How it works and why it belongs here:** The module path matches main’s import. Go 1.22 or newer is sufficient; the implementation uses only the standard library.

### 4.2. `internal/lld/clock.go`

```go
package lld

import (
	"time"
)

type Clock interface{ Now() time.Time }
type SystemClock struct{}

func (SystemClock) Now() time.Time { return time.Now() }
```

**How it works and why it belongs here:** The consumer needs only Now. Production and test clocks satisfy the same interface implicitly.

### 4.3. `internal/lld/cache.go`

```go
package lld

import (
	"container/list"
	"sync"
	"time"
)

type entry[K comparable, V any] struct {
	key     K
	value   V
	expires time.Time
}
type Cache[K comparable, V any] struct {
	mu       sync.Mutex
	capacity int
	clock    Clock
	order    *list.List
	items    map[K]*list.Element
}

func NewCache[K comparable, V any](capacity int, clock Clock) *Cache[K, V] {
	if capacity < 1 || clock == nil {
		panic("invalid cache configuration")
	}
	return &Cache[K, V]{capacity: capacity, clock: clock, order: list.New(), items: map[K]*list.Element{}}
}
func (c *Cache[K, V]) remove(e *list.Element) {
	delete(c.items, e.Value.(entry[K, V]).key)
	c.order.Remove(e)
}
func (c *Cache[K, V]) Put(key K, value V, ttl time.Duration) {
	if ttl <= 0 {
		panic("TTL must be positive")
	}
	c.mu.Lock()
	defer c.mu.Unlock()
	now := c.clock.Now()
	for _, e := range c.items {
		if !now.Before(e.Value.(entry[K, V]).expires) {
			c.remove(e)
		}
	}
	if e, ok := c.items[key]; ok {
		e.Value = entry[K, V]{key, value, now.Add(ttl)}
		c.order.MoveToFront(e)
		return
	}
	c.items[key] = c.order.PushFront(entry[K, V]{key, value, now.Add(ttl)})
	if len(c.items) > c.capacity {
		c.remove(c.order.Back())
	}
}
func (c *Cache[K, V]) Get(key K) (V, bool) {
	c.mu.Lock()
	defer c.mu.Unlock()
	var zero V
	e, ok := c.items[key]
	if !ok {
		return zero, false
	}
	v := e.Value.(entry[K, V])
	if !c.clock.Now().Before(v.expires) {
		c.remove(e)
		return zero, false
	}
	c.order.MoveToFront(e)
	return v.value, true
}
func (c *Cache[K, V]) Delete(key K) {
	c.mu.Lock()
	defer c.mu.Unlock()
	if e, ok := c.items[key]; ok {
		c.remove(e)
	}
}
```

**How it works and why it belongs here:** The map stores direct list-node references so promotion/removal need no search. Front is MRU and back is LRU. The mutex protects both structures together; remove assumes its caller holds the lock. Generics require comparable keys and allow arbitrary values.

### 4.4. `cmd/demo/main.go`

```go
package main

import (
	"example.com/cache/internal/lld"
	"fmt"
	"time"
)

func main() {
	c := lld.NewCache[string, string](2, lld.SystemClock{})
	c.Put("A", "alpha", time.Minute)
	c.Put("B", "beta", time.Minute)
	c.Get("A")
	c.Put("C", "gamma", time.Minute)
	_, ok := c.Get("B")
	fmt.Println("B present:", ok)
	v, _ := c.Get("A")
	fmt.Println("A:", v)
}
```

**How it works and why it belongs here:** The composition root constructs dependencies, runs a concrete scenario, and prints observable results. Error checks make a rejected operation visible rather than silently treating it as success.

### 4.5. `internal/lld/service_test.go`

```go
package lld

import (
	"sync"
	"testing"
	"time"
)

type fakeClock struct{ now time.Time }

func (f *fakeClock) Now() time.Time { return f.now }
func TestLRUAndTTL(t *testing.T) {
	f := &fakeClock{time.Unix(0, 0)}
	c := NewCache[string, int](2, f)
	c.Put("a", 1, time.Second)
	c.Put("b", 2, time.Hour)
	c.Get("a")
	c.Put("c", 3, time.Hour)
	if _, ok := c.Get("b"); ok {
		t.Fatal("not LRU")
	}
	f.now = f.now.Add(time.Second)
	if _, ok := c.Get("a"); ok {
		t.Fatal("expiry boundary")
	}
	c.Put("c", 4, time.Hour)
	v, ok := c.Get("c")
	if !ok || v != 4 {
		t.Fatal("update")
	}
	c.Delete("c")
	if _, ok := c.Get("c"); ok {
		t.Fatal("delete")
	}
}
func TestExpiredBeforeLiveEviction(t *testing.T) {
	f := &fakeClock{time.Unix(0, 0)}
	c := NewCache[string, int](2, f)
	c.Put("live", 1, time.Hour)
	c.Put("expired", 2, time.Second)
	f.now = f.now.Add(time.Second)
	c.Put("new", 3, time.Hour)
	if _, ok := c.Get("live"); !ok {
		t.Fatal("evicted live before expired")
	}
}

func TestConcurrentCacheAccess(t *testing.T) {
	c := NewCache[int, int](8, SystemClock{})
	var wg sync.WaitGroup
	for worker := 0; worker < 8; worker++ {
		wg.Add(1)
		go func(k int) {
			defer wg.Done()
			for n := 0; n < 100; n++ {
				c.Put(k, n, time.Hour)
				c.Get(k)
				c.Delete(k)
			}
		}(worker)
	}
	wg.Wait()
	c.Put(1, 42, time.Hour)
	if v, ok := c.Get(1); !ok || v != 42 {
		t.Fatal("cache corrupted")
	}
}
```

**How it works and why it belongs here:** A fake clock tests exact expiration without sleep. Checks include LRU promotion, updates, deletion and expired-MRU cleanup before live eviction. A concurrent put/get/delete smoke test exercises the shared recency structures; the Go run also uses race detection.

## 5. Patterns and principles: where and why

| Pattern or principle | Concrete location | Reason |
|---|---|---|
| LRU eviction algorithm | Cache get/put | Recency determines which live entry leaves at capacity. |
| Dependency injection | Clock / Supplier<Instant> | Makes TTL deterministic in tests. |
| Encapsulation | Cache mutex / synchronized operations | Lookup, expiry, promotion and eviction are one atomic action. |
| Composition | Map plus recency structure | Combines fast lookup with ordered eviction; this sample does not invent an unnecessary eviction Strategy interface. |

A mutex or synchronized method is a concurrency mechanism, not a GoF design pattern. An enum is a state representation, not automatically the State pattern. Interfaces are justified by interchangeable behavior or a useful boundary; inheritance is not required to demonstrate OOP.

### Applying SOLID without unnecessary abstractions

**Single responsibility:** the main entry point assembles dependencies; the domain service owns state invariants; policy collaborators own the behavior named in the pattern table. The tests exercise behavior through the public operations rather than depending on implementation maps.

**Open/closed:** inspect `Cache get/put` as the primary variation point. Where a policy interface exists, supply a new implementation without changing the state-transition algorithm. Where this example implements a specific data structure, do not claim its algorithm is interchangeable until you deliberately extract that boundary.

**Liskov substitution:** a replacement collaborator must preserve the documented contract, including invalid-input behavior, clock units, ownership rules and callback failure behavior. Merely matching a method signature is not enough.

**Interface segregation and dependency inversion:** interfaces expose the small question the caller needs answered. Constructors receive policies and clocks where tests or alternative behavior benefit. Simple records, enum values and internal containers remain concrete. A dedicated repository interface is useful when persistence is in scope; these samples do not pretend in-memory mutations automatically translate to database transactions.

## 6. Follow the example and the invariants

With capacity 2, put A and B, then get A. B becomes the least recently used live entry. Put C and B is evicted. Advancing the injected clock to A's expiry causes get A to miss. Expiry uses now >= expiresAt, so the exact boundary is expired. Put removes all expired entries first so an expired MRU entry does not cause an unrelated live LRU entry to be evicted. Get miss does not create a cache entry.

### Verified execution

The complete project above was compiled and its behavior tests executed. The following is actual validation output (paths, timing and identifiers can vary):

```text
?   	example.com/cache/cmd/demo	[no test files]
ok  	example.com/cache/internal/lld	1.606s

B present: false
A: alpha
```

## 7. Best practices, edge cases and production extensions

Get and delete are O(1) average; put is O(N) because it scans for expired entries before insertion. Do not advertise every operation as O(1). The explicit scan makes expiration behavior easy to reason about. A min-heap or timing wheel can support more efficient expiration at greater implementation cost. Expired entries may retain memory until an operation removes them, but physical entries never exceed capacity.

Returning a generic value does not deep-copy mutable objects. Cache callers should use immutable values or own their synchronization. The clock must be nondecreasing for intuitive TTL semantics; production time sources require care around wall-clock adjustments. LRU is a policy choice, not automatically best for scan-heavy workloads. Loading on misses requires a separate single-flight mechanism to prevent stampedes. Distributed caches need serialization, consistent key definitions and failure semantics; none are implicit in this mutex-protected implementation.

## 8. Presenting this in an interview

Start by agreeing on the scope in section 1. Draw the responsibility flow, identify the state that must remain consistent, and name the operation that owns that invariant. Implement the core model and service, then wire the collaborators in main and run a concrete example. Show at least one rejected or boundary case from the tests. Explain the pattern at the point where it solves a problem, rather than starting with a list of pattern names.

For a distributed follow-up, distinguish thread safety inside this process from coordination across replicas. In-memory objects do not survive restarts. Agree on consistency, failure recovery and storage requirements before replacing them with remote infrastructure.

## 9. Practice next

1. Add a size/weight limit instead of only entry count.
2. Implement a separate loading-cache wrapper with one in-flight load per key.
3. Replace the expiration scan with a heap and compare complexity.
4. Add hit/miss/eviction metrics without exposing internal mutable state.
