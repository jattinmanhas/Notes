# Parking Lot in Go — complete LLD walkthrough

Study order: requirements → model and flow → project tree → each complete file and explanation → patterns → walkthrough → limitations and exercises. This is a runnable single-process interview reference, with no external services or omitted source files.

## 1. Requirements and scope

Park cars or motorcycles, allocate a compatible free spot, issue a ticket, and free the spot on exit. Calculate fees using a replaceable policy. Reject duplicate entry, unknown tickets, time-travelling exits and a full lot. Money uses integer minor units. A motorcycle may use a car spot only when no motorcycle spot is free. Payment processing and physical gates are outside scope.

## 2. Design and responsibilities

```text
Main -> ParkingLot -> compatible spot allocation
                   -> active ticket map + occupied spot map (one lock)
                   -> FeePolicy on exit
Vehicle -> Ticket -> Spot
```

## 3. Project structure and running

Create each file at the shown relative path. All imports and package declarations are included.

```text
parking-lot-go/
  go.mod
  internal/lld/model.go
  internal/lld/pricing.go
  internal/lld/lot.go
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
module example.com/parking-lot

go 1.22
```

**How it works and why it belongs here:** The module path matches main’s import. Go 1.22 or newer is sufficient; the implementation uses only the standard library.

### 4.2. `internal/lld/model.go`

```go
package lld

import (
	"time"
)

type Kind int

const (
	Motorcycle Kind = iota
	Car
)

type Vehicle struct {
	Plate string
	Kind  Kind
}
type Spot struct {
	ID   int
	Kind Kind
}
type Ticket struct {
	ID      int
	Plate   string
	SpotID  int
	Entered time.Time
}
```

**How it works and why it belongs here:** Value types describe vehicles, spots and immutable ticket snapshots. Kind is a named type; the service validates values because Go permits arbitrary integer conversions.

### 4.3. `internal/lld/pricing.go`

```go
package lld

import (
	"fmt"
	"time"
)

type FeePolicy interface {
	Fee(time.Duration) (int64, error)
}
type HourlyFee struct{ Rate int64 }

func (p HourlyFee) Fee(d time.Duration) (int64, error) {
	if d < 0 || p.Rate < 0 {
		return 0, fmt.Errorf("invalid duration or rate")
	}
	hours := int64(d / time.Hour)
	if d%time.Hour != 0 {
		hours++
	}
	return hours * p.Rate, nil
}
```

**How it works and why it belongs here:** FeePolicy is the Strategy boundary. HourlyFee rounds up without floating-point arithmetic. It treats a zero stay as free and rejects negative durations/rates.

### 4.4. `internal/lld/lot.go`

```go
package lld

import (
	"fmt"
	"sort"
	"sync"
	"time"
)

type ParkingLot struct {
	mu       sync.Mutex
	spots    []Spot
	occupied map[int]bool
	tickets  map[int]Ticket
	plates   map[string]bool
	next     int
	fee      FeePolicy
}

func NewParkingLot(spots []Spot, fee FeePolicy) *ParkingLot {
	if fee == nil {
		panic("fee required")
	}
	copySpots := append([]Spot(nil), spots...)
	seen := map[int]bool{}
	for _, s := range copySpots {
		if seen[s.ID] || (s.Kind != Car && s.Kind != Motorcycle) {
			panic("invalid spot")
		}
		seen[s.ID] = true
	}
	sort.Slice(copySpots, func(i, j int) bool { return copySpots[i].ID < copySpots[j].ID })
	return &ParkingLot{spots: copySpots, occupied: map[int]bool{}, tickets: map[int]Ticket{}, plates: map[string]bool{}, fee: fee}
}
func (l *ParkingLot) Park(v Vehicle, at time.Time) (Ticket, error) {
	l.mu.Lock()
	defer l.mu.Unlock()
	if v.Plate == "" || (v.Kind != Car && v.Kind != Motorcycle) || l.plates[v.Plate] {
		return Ticket{}, fmt.Errorf("invalid or already parked vehicle")
	}
	for _, kind := range []Kind{v.Kind, Car} {
		for _, s := range l.spots {
			if s.Kind != kind || l.occupied[s.ID] {
				continue
			}
			l.next++
			t := Ticket{l.next, v.Plate, s.ID, at}
			l.occupied[s.ID] = true
			l.plates[v.Plate] = true
			l.tickets[t.ID] = t
			return t, nil
		}
	}
	return Ticket{}, fmt.Errorf("no compatible spot")
}
func (l *ParkingLot) Exit(id int, at time.Time) (int64, error) {
	l.mu.Lock()
	defer l.mu.Unlock()
	t, ok := l.tickets[id]
	if !ok {
		return 0, fmt.Errorf("unknown ticket")
	}
	fee, err := l.fee.Fee(at.Sub(t.Entered))
	if err != nil {
		return 0, err
	}
	delete(l.tickets, id)
	delete(l.plates, t.Plate)
	delete(l.occupied, t.SpotID)
	return fee, nil
}
```

**How it works and why it belongs here:** ParkingLot owns all mutations. The constructor copies caller-owned spots and rejects duplicate IDs. Park and Exit serialize transitions across tickets, plates and occupancy. Pricing errors leave occupancy intact.

### 4.5. `cmd/demo/main.go`

```go
package main

import (
	"example.com/parking-lot/internal/lld"
	"fmt"
	"time"
)

func main() {
	lot := lld.NewParkingLot([]lld.Spot{{ID: 1, Kind: lld.Car}}, lld.HourlyFee{Rate: 100})
	now := time.Unix(0, 0)
	t, err := lot.Park(lld.Vehicle{Plate: "AB123", Kind: lld.Car}, now)
	if err != nil {
		panic(err)
	}
	fee, err := lot.Exit(t.ID, now.Add(61*time.Minute))
	if err != nil {
		panic(err)
	}
	fmt.Println("Fee:", fee)
}
```

**How it works and why it belongs here:** The composition root constructs dependencies, runs a concrete scenario, and prints observable results. Error checks make a rejected operation visible rather than silently treating it as success.

### 4.6. `internal/lld/service_test.go`

```go
package lld

import (
	"sync"
	"sync/atomic"
	"testing"
	"time"
)

func TestParking(t *testing.T) {
	l := NewParkingLot([]Spot{{1, Car}}, HourlyFee{100})
	now := time.Unix(0, 0)
	a, e := l.Park(Vehicle{"A", Car}, now)
	if e != nil {
		t.Fatal(e)
	}
	if _, e = l.Park(Vehicle{"B", Car}, now); e == nil {
		t.Fatal("overbooked")
	}
	if _, e = l.Exit(a.ID, now.Add(-time.Second)); e == nil {
		t.Fatal("negative stay")
	}
	fee, e := l.Exit(a.ID, now.Add(61*time.Minute))
	if e != nil || fee != 200 {
		t.Fatal(fee, e)
	}
	if _, e = l.Park(Vehicle{"B", Car}, now); e != nil {
		t.Fatal("spot not freed")
	}
}

func TestConcurrentAllocation(t *testing.T) {
	lot := NewParkingLot([]Spot{{1, Car}}, HourlyFee{100})
	var wg sync.WaitGroup
	var wins atomic.Int64
	for _, plate := range []string{"A", "B"} {
		wg.Add(1)
		go func(p string) {
			defer wg.Done()
			if _, err := lot.Park(Vehicle{p, Car}, time.Unix(0, 0)); err == nil {
				wins.Add(1)
			}
		}(plate)
	}
	wg.Wait()
	if wins.Load() != 1 {
		t.Fatal("one physical spot assigned twice")
	}
}
func TestSmallestCompatibleSpot(t *testing.T) {
	lot := NewParkingLot([]Spot{{1, Car}, {2, Motorcycle}}, HourlyFee{100})
	bike, e := lot.Park(Vehicle{"M", Motorcycle}, time.Unix(0, 0))
	if e != nil || bike.SpotID != 2 {
		t.Fatal("motorcycle consumed car space")
	}
	if _, e = lot.Park(Vehicle{"C", Car}, time.Unix(0, 0)); e != nil {
		t.Fatal(e)
	}
}
```

**How it works and why it belongs here:** Tests cover capacity, invalid exit preserving state, fee rounding and reuse of a released spot. Additional tests race for one physical spot and verify motorcycles prefer motorcycle spots.

## 5. Patterns and principles: where and why

| Pattern or principle | Concrete location | Reason |
|---|---|---|
| Strategy | FeePolicy / HourlyFee | Change pricing without rewriting allocation or exit. |
| Dependency injection | ParkingLot constructor in Main | Pricing is supplied explicitly. |
| Encapsulation | ParkingLot.Park/Exit or park/exit | Allocation and ticket state change together. |
| Composition | ParkingLot contains spots and policy | The lot has resources and behavior collaborators; subclassing a lot is unnecessary. |

A mutex or synchronized method is a concurrency mechanism, not a GoF design pattern. An enum is a state representation, not automatically the State pattern. Interfaces are justified by interchangeable behavior or a useful boundary; inheritance is not required to demonstrate OOP.

### Applying SOLID without unnecessary abstractions

**Single responsibility:** the main entry point assembles dependencies; the domain service owns state invariants; policy collaborators own the behavior named in the pattern table. The tests exercise behavior through the public operations rather than depending on implementation maps.

**Open/closed:** inspect `FeePolicy / HourlyFee` as the primary variation point. Where a policy interface exists, supply a new implementation without changing the state-transition algorithm. Where this example implements a specific data structure, do not claim its algorithm is interchangeable until you deliberately extract that boundary.

**Liskov substitution:** a replacement collaborator must preserve the documented contract, including invalid-input behavior, clock units, ownership rules and callback failure behavior. Merely matching a method signature is not enough.

**Interface segregation and dependency inversion:** interfaces expose the small question the caller needs answered. Constructors receive policies and clocks where tests or alternative behavior benefit. Simple records, enum values and internal containers remain concrete. A dedicated repository interface is useful when persistence is in scope; these samples do not pretend in-memory mutations automatically translate to database transactions.

## 6. Follow the example and the invariants

Entry scans spot IDs deterministically, preferring the smallest compatible type. Allocation, vehicle registration and ticket creation happen under one lock. No two active tickets can own one spot, and one plate cannot be parked twice. Exit computes the fee before releasing resources, so an invalid timestamp does not lose the ticket. A 61-minute stay at 100 minor units per started hour costs 200. A zero-duration stay is free in this agreed policy. The demo parks a car, exits after 61 minutes and prints the fee.

### Verified execution

The complete project above was compiled and its behavior tests executed. The following is actual validation output (paths, timing and identifiers can vary):

```text
?   	example.com/parking-lot/cmd/demo	[no test files]
ok  	example.com/parking-lot/internal/lld	1.706s

Fee: 200
```

## 7. Best practices, edge cases and production extensions

A single lock intentionally protects the invariant spanning three maps. This is simple and sufficient for an interview, but scans are O(S) for S spots. Separate ordered free-spot indexes can improve allocation while retaining atomicity. Tickets are returned as copies/immutable records; callers cannot rewrite occupancy. The fee strategy executes under the lock and must be fast and side-effect-free; do not call a payment network there.

For production, persist ticket ownership and use database constraints/transactions. Add a payment state machine: ACTIVE -> PAYMENT_PENDING -> PAID -> CLOSED, with payment idempotency and explicit gate-failure recovery. Do not release the spot merely because a network request was attempted. A real vehicle identity may require jurisdiction plus plate. The demo has no reservations, levels, disabled/EV spots, or concurrent payment integration. Very large money values require overflow limits.

## 8. Presenting this in an interview

Start by agreeing on the scope in section 1. Draw the responsibility flow, identify the state that must remain consistent, and name the operation that owns that invariant. Implement the core model and service, then wire the collaborators in main and run a concrete example. Show at least one rejected or boundary case from the tests. Explain the pattern at the point where it solves a problem, rather than starting with a list of pattern names.

For a distributed follow-up, distinguish thread safety inside this process from coordination across replicas. In-memory objects do not survive restarts. Agree on consistency, failure recovery and storage requirements before replacing them with remote infrastructure.

## 9. Practice next

1. Add EV-compatible spots without breaking normal cars.
2. Inject weekend pricing and test it independently.
3. Add a free-spot index and preserve deterministic allocation.
4. Model payment failure and duplicate exit requests.
