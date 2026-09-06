# BookMyShow in Go — complete LLD walkthrough

Study order: requirements → model and flow → project tree → each complete file and explanation → patterns → walkthrough → limitations and exercises. This is a runnable single-process interview reference, with no external services or omitted source files.

## 1. Requirements and scope

Implement seat inventory for one show: list available seats, atomically hold a group of seats for a user, confirm before expiry, or cancel a hold. Holds have a TTL and owner. Reject duplicate seats, unknown seats, already held/booked seats and confirmation by another user. A confirmed hold stays booked; repeat confirmation by its owner is idempotent. Search, theaters, show scheduling and actual payment are outside scope.

## 2. Design and responsibilities

```text
Main -> BookingService -> show seat inventory
                      -> Hold(id,owner,seats,expiry,state)
Clock -> lazy hold expiry
AVAILABLE -> HELD -> BOOKED
              \-> EXPIRED/CANCELLED -> AVAILABLE
```

## 3. Project structure and running

Create each file at the shown relative path. All imports and package declarations are included.

```text
bookmyshow-go/
  go.mod
  internal/lld/model.go
  internal/lld/booking.go
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
module example.com/bookmyshow

go 1.22
```

**How it works and why it belongs here:** The module path matches main’s import. Go 1.22 or newer is sufficient; the implementation uses only the standard library.

### 4.2. `internal/lld/model.go`

```go
package lld

import (
	"time"
)

type HoldStatus string

const (
	Active    HoldStatus = "HELD"
	Booked    HoldStatus = "BOOKED"
	Expired   HoldStatus = "EXPIRED"
	Cancelled HoldStatus = "CANCELLED"
)

type Hold struct {
	ID      int
	Owner   string
	Seats   []string
	Expires time.Time
	Status  HoldStatus
}
type Clock interface{ Now() time.Time }
type SystemClock struct{}

func (SystemClock) Now() time.Time { return time.Now() }
```

**How it works and why it belongs here:** Hold records explicit lifecycle state. Seats are copied at the service boundary because slices otherwise share backing storage.

### 4.3. `internal/lld/booking.go`

```go
package lld

import (
	"fmt"
	"sort"
	"sync"
	"time"
)

type BookingService struct {
	mu    sync.Mutex
	seats map[string]int
	holds map[int]Hold
	next  int
	clock Clock
}

func NewBookingService(seats []string, clock Clock) *BookingService {
	if clock == nil {
		panic("clock")
	}
	s := &BookingService{seats: map[string]int{}, holds: map[int]Hold{}, clock: clock}
	for _, id := range seats {
		if _, ok := s.seats[id]; ok || id == "" {
			panic("invalid seat")
		}
		s.seats[id] = 0
	}
	return s
}
func (s *BookingService) expire() {
	now := s.clock.Now()
	for id, h := range s.holds {
		if h.Status == Active && !now.Before(h.Expires) {
			h.Status = Expired
			s.holds[id] = h
			for _, seat := range h.Seats {
				s.seats[seat] = 0
			}
		}
	}
}
func (s *BookingService) Hold(owner string, seats []string, ttl time.Duration) (Hold, error) {
	s.mu.Lock()
	defer s.mu.Unlock()
	s.expire()
	if owner == "" || len(seats) == 0 || ttl <= 0 {
		return Hold{}, fmt.Errorf("invalid request")
	}
	seen := map[string]bool{}
	for _, seat := range seats {
		occupant, ok := s.seats[seat]
		if !ok || occupant != 0 || seen[seat] {
			return Hold{}, fmt.Errorf("seat unavailable")
		}
		seen[seat] = true
	}
	s.next++
	h := Hold{s.next, owner, append([]string(nil), seats...), s.clock.Now().Add(ttl), Active}
	s.holds[h.ID] = h
	for _, seat := range seats {
		s.seats[seat] = h.ID
	}
	h.Seats = append([]string(nil), h.Seats...)
	return h, nil
}
func (s *BookingService) Confirm(id int, owner string) error {
	s.mu.Lock()
	defer s.mu.Unlock()
	s.expire()
	h, ok := s.holds[id]
	if !ok || h.Owner != owner {
		return fmt.Errorf("unknown hold or wrong owner")
	}
	if h.Status == Booked {
		return nil
	}
	if h.Status != Active {
		return fmt.Errorf("hold inactive")
	}
	h.Status = Booked
	s.holds[id] = h
	return nil
}
func (s *BookingService) Cancel(id int, owner string) error {
	s.mu.Lock()
	defer s.mu.Unlock()
	s.expire()
	h, ok := s.holds[id]
	if !ok || h.Owner != owner || h.Status != Active {
		return fmt.Errorf("hold inactive or wrong owner")
	}
	h.Status = Cancelled
	s.holds[id] = h
	for _, seat := range h.Seats {
		s.seats[seat] = 0
	}
	return nil
}
func (s *BookingService) Available() []string {
	s.mu.Lock()
	defer s.mu.Unlock()
	s.expire()
	r := []string{}
	for seat, id := range s.seats {
		if id == 0 {
			r = append(r, seat)
		}
	}
	sort.Strings(r)
	return r
}
```

**How it works and why it belongs here:** Hold validates every seat before assigning any. Expiry only releases active holds. Confirm checks ownership even on the idempotent path. The zero inventory value means available; hold IDs start at one.

### 4.4. `cmd/demo/main.go`

```go
package main

import (
	"example.com/bookmyshow/internal/lld"
	"fmt"
	"time"
)

func main() {
	s := lld.NewBookingService([]string{"A1", "A2", "A3"}, lld.SystemClock{})
	h, e := s.Hold("alice", []string{"A1", "A2"}, time.Minute)
	if e != nil {
		panic(e)
	}
	if e = s.Confirm(h.ID, "alice"); e != nil {
		panic(e)
	}
	fmt.Println("Booked hold", h.ID, "available:", s.Available())
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

type fakeClock struct{ now time.Time }

func (c *fakeClock) Now() time.Time { return c.now }
func TestHolds(t *testing.T) {
	c := &fakeClock{time.Unix(0, 0)}
	s := NewBookingService([]string{"A", "B"}, c)
	h, e := s.Hold("u", []string{"A"}, time.Second)
	if e != nil {
		t.Fatal(e)
	}
	if _, e = s.Hold("v", []string{"B", "A"}, time.Second); e == nil {
		t.Fatal("double booking")
	}
	if len(s.Available()) != 1 {
		t.Fatal("partial hold")
	}
	if s.Confirm(h.ID, "other") == nil {
		t.Fatal("owner")
	}
	c.now = c.now.Add(time.Second)
	if s.Confirm(h.ID, "u") == nil {
		t.Fatal("expired confirmation")
	}
	h, e = s.Hold("v", []string{"A"}, time.Second)
	if e != nil {
		t.Fatal(e)
	}
	if s.Confirm(h.ID, "v") != nil || s.Confirm(h.ID, "v") != nil {
		t.Fatal("confirmation")
	}
}

func TestConcurrentSeatOwnership(t *testing.T) {
	s := NewBookingService([]string{"A"}, SystemClock{})
	var wg sync.WaitGroup
	var wins atomic.Int64
	for _, user := range []string{"x", "y"} {
		wg.Add(1)
		go func(u string) {
			defer wg.Done()
			if _, err := s.Hold(u, []string{"A"}, time.Hour); err == nil {
				wins.Add(1)
			}
		}(user)
	}
	wg.Wait()
	if wins.Load() != 1 {
		t.Fatal("seat acquired twice")
	}
}
```

**How it works and why it belongs here:** Covers atomic multi-seat rejection, ownership, exact expiry, released-seat reuse and idempotent confirmation. A contention test proves only one caller can acquire the same seat.

## 5. Patterns and principles: where and why

| Pattern or principle | Concrete location | Reason |
|---|---|---|
| Aggregate boundary | BookingService / ShowInventory | One lock protects a multi-seat hold as an all-or-nothing operation. |
| State-machine modeling | HoldStatus and confirm/cancel/expire | Only allowed transitions can alter availability. |
| Dependency injection | clock in service constructor | Tests can expire reservations without sleeping. |
| Idempotency | confirm on an already booked hold | Repeated confirmation does not allocate twice. |

A mutex or synchronized method is a concurrency mechanism, not a GoF design pattern. An enum is a state representation, not automatically the State pattern. Interfaces are justified by interchangeable behavior or a useful boundary; inheritance is not required to demonstrate OOP.

### Applying SOLID without unnecessary abstractions

**Single responsibility:** the main entry point assembles dependencies; the domain service owns state invariants; policy collaborators own the behavior named in the pattern table. The tests exercise behavior through the public operations rather than depending on implementation maps.

**Open/closed:** inspect `BookingService / ShowInventory` as the primary variation point. Where a policy interface exists, supply a new implementation without changing the state-transition algorithm. Where this example implements a specific data structure, do not claim its algorithm is interchangeable until you deliberately extract that boundary.

**Liskov substitution:** a replacement collaborator must preserve the documented contract, including invalid-input behavior, clock units, ownership rules and callback failure behavior. Merely matching a method signature is not enough.

**Interface segregation and dependency inversion:** interfaces expose the small question the caller needs answered. Constructors receive policies and clocks where tests or alternative behavior benefit. Simple records, enum values and internal containers remain concrete. A dedicated repository interface is useful when persistence is in scope; these samples do not pretend in-memory mutations automatically translate to database transactions.

## 6. Follow the example and the invariants

User A holds seats A1 and A2. Both seats become unavailable in one atomic operation. User B cannot hold A2. At expiry exactly, A's unconfirmed hold releases both seats, permitting B to hold A2. A cannot then confirm the expired hold. A confirmed booking is not expired by the hold cleanup, and only its owner may repeat confirmation. The demo holds A1/A2 and confirms successfully before the clock advances.

### Verified execution

The complete project above was compiled and its behavior tests executed. The following is actual validation output (paths, timing and identifiers can vary):

```text
?   	example.com/bookmyshow/cmd/demo	[no test files]
ok  	example.com/bookmyshow/internal/lld	1.629s

Booked hold 1 available: [A3]
```

## 7. Best practices, edge cases and production extensions

This intentionally centers the hard seat-ownership invariant. The service owns one show, so seat IDs need no show prefix internally; a larger system partitions inventory by show ID. Expiration scans all holds lazily on each operation, O(H). Production needs cleanup/indexing and archival of terminal holds. Every public operation returns copies or immutable values so a caller cannot change seat ownership.

Do not call a payment provider under the inventory lock. A payment integration requires an idempotent payment attempt, a durable reservation, and a transaction that verifies the reservation is still valid. If payment succeeds after expiry, compensate with a refund rather than stealing a seat from another user. Multi-server seat ownership needs database conditional writes/transactions and a consistent expiry policy. Hold creation is not idempotent here; add a client request key for network retries. Cancellation is for active holds only, not a refund API for booked tickets.

## 8. Presenting this in an interview

Start by agreeing on the scope in section 1. Draw the responsibility flow, identify the state that must remain consistent, and name the operation that owns that invariant. Implement the core model and service, then wire the collaborators in main and run a concrete example. Show at least one rejected or boundary case from the tests. Explain the pattern at the point where it solves a problem, rather than starting with a list of pattern names.

For a distributed follow-up, distinguish thread safety inside this process from coordination across replicas. In-memory objects do not survive restarts. Agree on consistency, failure recovery and storage requirements before replacing them with remote infrastructure.

## 9. Practice next

1. Add show IDs and keep inventory locks independent per show.
2. Add a request key to hold creation and detect conflicting reuse.
3. Model payment success after reservation expiry.
4. Race two callers for overlapping seat groups and assert no partial allocation.
