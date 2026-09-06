# Elevator in Go — complete LLD walkthrough

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
elevator-go/
  go.mod
  internal/lld/policy.go
  internal/lld/elevator.go
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
module example.com/elevator

go 1.22
```

**How it works and why it belongs here:** The module path matches main’s import. Go 1.22 or newer is sufficient; the implementation uses only the standard library.

### 4.2. `internal/lld/policy.go`

```go
package lld

type StopPolicy interface {
	Next(int, int, []int) (int, bool)
}
type SweepPolicy struct{}

func (SweepPolicy) Next(floor, dir int, stops []int) (int, bool) {
	for _, s := range stops {
		if s == floor {
			return s, true
		}
	}
	if dir == 0 {
		dir = 1
	}
	for _, direction := range []int{dir, -dir} {
		target := 0
		found := false
		for _, s := range stops {
			if (s-floor)*direction > 0 && (!found || (s-target)*direction < 0) {
				target = s
				found = true
			}
		}
		if found {
			return target, true
		}
	}
	return 0, false
}
```

**How it works and why it belongs here:** SweepPolicy scans candidates in two passes: forward first, reverse second. Passing an empty set returns false. Direction is -1, 0 or +1 and belongs to the elevator model.

### 4.3. `internal/lld/elevator.go`

```go
package lld

import (
	"fmt"
	"sync"
)

type Snapshot struct {
	Floor, Direction int
	DoorsOpen        bool
	Pending          int
}
type Elevator struct {
	mu              sync.Mutex
	floor, dir, max int
	open            bool
	stops           map[int]bool
	policy          StopPolicy
}

func NewElevator(max int, p StopPolicy) *Elevator {
	if max < 0 || p == nil {
		panic("invalid configuration")
	}
	return &Elevator{max: max, policy: p, stops: map[int]bool{}}
}
func (e *Elevator) Request(floor int) error {
	e.mu.Lock()
	defer e.mu.Unlock()
	if floor < 0 || floor > e.max {
		return fmt.Errorf("floor out of range")
	}
	e.stops[floor] = true
	return nil
}
func (e *Elevator) Tick() Snapshot {
	e.mu.Lock()
	defer e.mu.Unlock()
	if e.open {
		e.open = false
		return e.snapshot()
	}
	stops := make([]int, 0, len(e.stops))
	for s := range e.stops {
		stops = append(stops, s)
	}
	target, ok := e.policy.Next(e.floor, e.dir, stops)
	if !ok {
		e.dir = 0
		return e.snapshot()
	}
	if !e.stops[target] {
		panic("policy returned an unrequested stop")
	}
	if target > e.floor {
		e.dir = 1
		e.floor++
	} else if target < e.floor {
		e.dir = -1
		e.floor--
	}
	if e.stops[e.floor] {
		delete(e.stops, e.floor)
		e.open = true
	}
	return e.snapshot()
}
func (e *Elevator) snapshot() Snapshot { return Snapshot{e.floor, e.dir, e.open, len(e.stops)} }
```

**How it works and why it belongs here:** Elevator owns the floor, direction and door state. Tick closes doors before any move. Snapshots expose immutable copies, and the service checks a policy result refers to actual pending work.

### 4.4. `cmd/demo/main.go`

```go
package main

import (
	"example.com/elevator/internal/lld"
	"fmt"
)

func main() {
	e := lld.NewElevator(10, lld.SweepPolicy{})
	if err := e.Request(3); err != nil {
		panic(err)
	}
	if err := e.Request(1); err != nil {
		panic(err)
	}
	for i := 0; i < 6; i++ {
		fmt.Println("tick", i, e.Tick())
	}
}
```

**How it works and why it belongs here:** The composition root constructs dependencies, runs a concrete scenario, and prints observable results. Error checks make a rejected operation visible rather than silently treating it as success.

### 4.5. `internal/lld/service_test.go`

```go
package lld

import (
	"testing"
)

func TestSweep(t *testing.T) {
	e := NewElevator(5, SweepPolicy{})
	_ = e.Request(3)
	_ = e.Request(1)
	_ = e.Request(1)
	a := e.Tick()
	if a.Floor != 1 || !a.DoorsOpen || a.Pending != 1 {
		t.Fatal(a)
	}
	b := e.Tick()
	if b.Floor != 1 || b.DoorsOpen {
		t.Fatal("moved while closing", b)
	}
	e.Tick()
	c := e.Tick()
	if c.Floor != 3 || !c.DoorsOpen {
		t.Fatal(c)
	}
	if e.Request(6) == nil {
		t.Fatal("invalid floor accepted")
	}
}
```

**How it works and why it belongs here:** Tests assert stop ordering, duplicate coalescing, door-close/movement separation and range validation.

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
?   	example.com/elevator/cmd/demo	[no test files]
ok  	example.com/elevator/internal/lld	(cached)

tick 0 {1 1 true 1}
tick 1 {1 1 false 1}
tick 2 {2 1 false 1}
tick 3 {3 1 true 0}
tick 4 {3 1 false 0}
tick 5 {3 0 false 0}
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
