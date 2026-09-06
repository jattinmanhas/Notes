# Food Delivery System in Go — complete LLD walkthrough

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
food-delivery-go/
  go.mod
  internal/lld/model.go
  internal/lld/assignment.go
  internal/lld/service.go
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
module example.com/food-delivery

go 1.22
```

**How it works and why it belongs here:** The module path matches main’s import. Go 1.22 or newer is sufficient; the implementation uses only the standard library.

### 4.2. `internal/lld/model.go`

```go
package lld

type Status string

const (
	Placed    Status = "PLACED"
	Accepted  Status = "ACCEPTED"
	Ready     Status = "READY"
	Out       Status = "OUT_FOR_DELIVERY"
	Delivered Status = "DELIVERED"
	Cancelled Status = "CANCELLED"
)

type MenuItem struct {
	Price int64
	Stock int
}
type Order struct {
	ID     int
	Items  map[string]int
	Total  int64
	Status Status
	Driver string
}
```

**How it works and why it belongs here:** Status expresses the agreed lifecycle. Order includes a price snapshot and quantities needed to restore stock on an allowed cancellation.

### 4.3. `internal/lld/assignment.go`

```go
package lld

import (
	"sort"
)

type DriverPolicy interface{ Choose([]string) (string, bool) }
type FirstAvailable struct{}

func (FirstAvailable) Choose(free []string) (string, bool) {
	if len(free) == 0 {
		return "", false
	}
	sort.Strings(free)
	return free[0], true
}
```

**How it works and why it belongs here:** The policy receives a detached list of free IDs, so sorting cannot mutate service state. It is intentionally simple and replaceable.

### 4.4. `internal/lld/service.go`

```go
package lld

import (
	"fmt"
	"sync"
)

type FoodService struct {
	mu      sync.Mutex
	menu    map[string]MenuItem
	drivers map[string]bool
	orders  map[int]Order
	next    int
	policy  DriverPolicy
}

func NewFoodService(menu map[string]MenuItem, drivers []string, p DriverPolicy) *FoodService {
	if p == nil {
		panic("policy")
	}
	s := &FoodService{menu: map[string]MenuItem{}, drivers: map[string]bool{}, orders: map[int]Order{}, policy: p}
	for id, m := range menu {
		if id == "" || m.Price < 0 || m.Price > 1_000_000_000 || m.Stock < 0 {
			panic("menu")
		}
		s.menu[id] = m
	}
	for _, id := range drivers {
		if id == "" {
			panic("driver")
		}
		if _, ok := s.drivers[id]; ok {
			panic("duplicate driver")
		}
		s.drivers[id] = false
	}
	return s
}
func copyOrder(o Order) Order {
	items := map[string]int{}
	for k, v := range o.Items {
		items[k] = v
	}
	o.Items = items
	return o
}
func (s *FoodService) Place(items map[string]int) (Order, error) {
	s.mu.Lock()
	defer s.mu.Unlock()
	if len(items) == 0 {
		return Order{}, fmt.Errorf("empty order")
	}
	var total int64
	for id, q := range items {
		m, ok := s.menu[id]
		if !ok || q <= 0 || q > 10000 || q > m.Stock {
			return Order{}, fmt.Errorf("invalid or unavailable item")
		}
		total += m.Price * int64(q)
	}
	s.next++
	o := Order{ID: s.next, Items: map[string]int{}, Total: total, Status: Placed}
	for id, q := range items {
		m := s.menu[id]
		m.Stock -= q
		s.menu[id] = m
		o.Items[id] = q
	}
	s.orders[o.ID] = o
	return copyOrder(o), nil
}
func (s *FoodService) transition(id int, from, to Status) error {
	o, ok := s.orders[id]
	if !ok || o.Status != from {
		return fmt.Errorf("invalid transition")
	}
	o.Status = to
	s.orders[id] = o
	return nil
}
func (s *FoodService) Accept(id int) error {
	s.mu.Lock()
	defer s.mu.Unlock()
	return s.transition(id, Placed, Accepted)
}
func (s *FoodService) Ready(id int) error {
	s.mu.Lock()
	defer s.mu.Unlock()
	return s.transition(id, Accepted, Ready)
}
func (s *FoodService) Dispatch(id int) error {
	s.mu.Lock()
	defer s.mu.Unlock()
	o, ok := s.orders[id]
	if !ok || o.Status != Ready {
		return fmt.Errorf("not ready")
	}
	free := []string{}
	for id, busy := range s.drivers {
		if !busy {
			free = append(free, id)
		}
	}
	driver, ok := s.policy.Choose(free)
	busy, exists := s.drivers[driver]
	if !ok || !exists || busy {
		return fmt.Errorf("no eligible driver")
	}
	s.drivers[driver] = true
	o.Driver = driver
	o.Status = Out
	s.orders[id] = o
	return nil
}
func (s *FoodService) Deliver(id int) error {
	s.mu.Lock()
	defer s.mu.Unlock()
	o, ok := s.orders[id]
	if !ok || o.Status != Out {
		return fmt.Errorf("not out for delivery")
	}
	o.Status = Delivered
	s.orders[id] = o
	s.drivers[o.Driver] = false
	return nil
}
func (s *FoodService) Cancel(id int) error {
	s.mu.Lock()
	defer s.mu.Unlock()
	o, ok := s.orders[id]
	if !ok || (o.Status != Placed && o.Status != Accepted) {
		return fmt.Errorf("cannot cancel")
	}
	for id, q := range o.Items {
		m := s.menu[id]
		m.Stock += q
		s.menu[id] = m
	}
	o.Status = Cancelled
	s.orders[id] = o
	return nil
}
func (s *FoodService) Get(id int) (Order, bool) {
	s.mu.Lock()
	defer s.mu.Unlock()
	o, ok := s.orders[id]
	return copyOrder(o), ok
}
```

**How it works and why it belongs here:** FoodService validates the entire order before reserving stock. Driver selection is revalidated before assignment. Every transition has an expected prior state, and callers receive copied item maps. Private transition requires the public caller to hold the lock.

### 4.5. `cmd/demo/main.go`

```go
package main

import (
	"example.com/food-delivery/internal/lld"
	"fmt"
)

func main() {
	s := lld.NewFoodService(map[string]lld.MenuItem{"burger": {Price: 500, Stock: 2}}, []string{"driver1"}, lld.FirstAvailable{})
	o, e := s.Place(map[string]int{"burger": 1})
	if e != nil {
		panic(e)
	}
	for _, step := range []func(int) error{s.Accept, s.Ready, s.Dispatch, s.Deliver} {
		if e = step(o.ID); e != nil {
			panic(e)
		}
	}
	o, _ = s.Get(o.ID)
	fmt.Println("Order", o.ID, o.Status, "total", o.Total)
}
```

**How it works and why it belongs here:** The composition root constructs dependencies, runs a concrete scenario, and prints observable results. Error checks make a rejected operation visible rather than silently treating it as success.

### 4.6. `internal/lld/service_test.go`

```go
package lld

import (
	"testing"
)

func TestOrders(t *testing.T) {
	s := NewFoodService(map[string]MenuItem{"x": {100, 2}}, []string{"d"}, FirstAvailable{})
	a, _ := s.Place(map[string]int{"x": 1})
	if s.Deliver(a.ID) == nil {
		t.Fatal("illegal jump")
	}
	if s.Cancel(a.ID) != nil {
		t.Fatal("cancel")
	}
	if s.Cancel(a.ID) == nil {
		t.Fatal("double cancel")
	}
	a, e := s.Place(map[string]int{"x": 2})
	if e != nil {
		t.Fatal("stock not restored")
	}
	_ = s.Accept(a.ID)
	_ = s.Ready(a.ID)
	if s.Dispatch(a.ID) != nil {
		t.Fatal("dispatch")
	}
	if s.Cancel(a.ID) == nil {
		t.Fatal("cancel after pickup")
	}
	if s.Deliver(a.ID) != nil {
		t.Fatal("deliver")
	}
	o, _ := s.Get(a.ID)
	if o.Total != 200 || o.Status != Delivered {
		t.Fatal(o)
	}
}

func TestExclusiveDriverAndRelease(t *testing.T) {
	s := NewFoodService(map[string]MenuItem{"x": {100, 2}}, []string{"d"}, FirstAvailable{})
	a, _ := s.Place(map[string]int{"x": 1})
	b, _ := s.Place(map[string]int{"x": 1})
	for _, id := range []int{a.ID, b.ID} {
		if err := s.Accept(id); err != nil {
			t.Fatal(err)
		}
		if err := s.Ready(id); err != nil {
			t.Fatal(err)
		}
	}
	if s.Dispatch(a.ID) != nil {
		t.Fatal("first assignment")
	}
	if s.Dispatch(b.ID) == nil {
		t.Fatal("busy driver reassigned")
	}
	if s.Deliver(a.ID) != nil || s.Dispatch(b.ID) != nil {
		t.Fatal("driver not released")
	}
}
```

**How it works and why it belongs here:** Checks illegal transitions, exactly-once restocking through cancellation and complete order progression. An additional test prevents assigning a busy driver twice and verifies delivery releases the driver.

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
?   	example.com/food-delivery/cmd/demo	[no test files]
ok  	example.com/food-delivery/internal/lld	1.642s

Order 1 DELIVERED total 500
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
