# Pub-Sub System in Go — complete LLD walkthrough

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
pub-sub-go/
  go.mod
  internal/lld/event.go
  internal/lld/bus.go
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
module example.com/pub-sub

go 1.22
```

**How it works and why it belongs here:** The module path matches main’s import. Go 1.22 or newer is sufficient; the implementation uses only the standard library.

### 4.2. `internal/lld/event.go`

```go
package lld

type Event struct{ Topic, Payload string }
type Handler func(Event) error
type subscription struct {
	id      int
	handler Handler
}
```

**How it works and why it belongs here:** Strings are immutable values; a handler cannot rewrite the payload for another subscriber. A function type lets ordinary functions or methods act as subscribers.

### 4.3. `internal/lld/bus.go`

```go
package lld

import (
	"fmt"
	"sync"
)

type EventBus struct {
	mu     sync.Mutex
	next   int
	topics map[string][]subscription
}

func NewEventBus() *EventBus { return &EventBus{topics: map[string][]subscription{}} }
func (b *EventBus) Subscribe(topic string, h Handler) int {
	if topic == "" || h == nil {
		panic("invalid subscription")
	}
	b.mu.Lock()
	defer b.mu.Unlock()
	b.next++
	b.topics[topic] = append(b.topics[topic], subscription{b.next, h})
	return b.next
}
func (b *EventBus) Unsubscribe(id int) bool {
	b.mu.Lock()
	defer b.mu.Unlock()
	for topic, subs := range b.topics {
		for i, s := range subs {
			if s.id == id {
				remaining := append(subs[:i:i], subs[i+1:]...)
				if len(remaining) == 0 {
					delete(b.topics, topic)
				} else {
					b.topics[topic] = remaining
				}
				return true
			}
		}
	}
	return false
}
func (b *EventBus) Publish(event Event) []error {
	b.mu.Lock()
	snapshot := append([]subscription(nil), b.topics[event.Topic]...)
	b.mu.Unlock()
	var failures []error
	for _, s := range snapshot {
		if err := s.handler(event); err != nil {
			failures = append(failures, fmt.Errorf("subscriber %d: %w", s.id, err))
		}
	}
	return failures
}
```

**How it works and why it belongs here:** The registry lock protects registration and snapshot creation only. Callback execution outside the lock supports reentrant subscribe/unsubscribe. A newly registered handler cannot receive a publish whose snapshot was already taken. IDs distinguish two registrations of the same handler.

### 4.4. `cmd/demo/main.go`

```go
package main

import (
	"example.com/pub-sub/internal/lld"
	"fmt"
)

func main() {
	b := lld.NewEventBus()
	id := b.Subscribe("orders", func(e lld.Event) error { fmt.Println("email:", e.Payload); return nil })
	b.Subscribe("orders", func(e lld.Event) error { fmt.Println("audit:", e.Payload); return nil })
	if errs := b.Publish(lld.Event{Topic: "orders", Payload: "created"}); len(errs) > 0 {
		panic(errs[0])
	}
	b.Unsubscribe(id)
	b.Publish(lld.Event{Topic: "orders", Payload: "shipped"})
}
```

**How it works and why it belongs here:** The composition root constructs dependencies, runs a concrete scenario, and prints observable results. Error checks make a rejected operation visible rather than silently treating it as success.

### 4.5. `internal/lld/service_test.go`

```go
package lld

import (
	"fmt"
	"testing"
)

func TestFanoutAndReentrancy(t *testing.T) {
	b := NewEventBus()
	count := 0
	id := 0
	id = b.Subscribe("a", func(Event) error { b.Unsubscribe(id); return fmt.Errorf("failed") })
	b.Subscribe("a", func(Event) error { count++; return nil })
	b.Subscribe("b", func(Event) error { t.Fatal("wrong topic"); return nil })
	if len(b.Publish(Event{"a", "one"})) != 1 || count != 1 {
		t.Fatal("failure isolation")
	}
	if len(b.Publish(Event{"a", "two"})) != 0 || count != 2 {
		t.Fatal("unsubscribe snapshot")
	}
	if b.Unsubscribe(id) {
		t.Fatal("unsubscribed twice")
	}
}
```

**How it works and why it belongs here:** Tests exact topic routing, error isolation and a self-unsubscribing handler, which would deadlock if callbacks ran under a non-reentrant registry mutex.

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
?   	example.com/pub-sub/cmd/demo	[no test files]
ok  	example.com/pub-sub/internal/lld	(cached)

email: created
audit: created
audit: shipped
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
