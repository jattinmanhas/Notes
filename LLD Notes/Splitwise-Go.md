# Splitwise in Go — complete LLD walkthrough

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
splitwise-go/
  go.mod
  internal/lld/split.go
  internal/lld/ledger.go
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
module example.com/splitwise

go 1.22
```

**How it works and why it belongs here:** The module path matches main’s import. Go 1.22 or newer is sufficient; the implementation uses only the standard library.

### 4.2. `internal/lld/split.go`

```go
package lld

import (
	"fmt"
)

type Splitter interface {
	Split(int64, []string) (map[string]int64, error)
}
type EqualSplit struct{}

func (EqualSplit) Split(total int64, users []string) (map[string]int64, error) {
	if total <= 0 || len(users) == 0 {
		return nil, fmt.Errorf("invalid split")
	}
	r := map[string]int64{}
	base := total / int64(len(users))
	extra := total % int64(len(users))
	for i, u := range users {
		r[u] = base
		if int64(i) < extra {
			r[u]++
		}
	}
	return r, nil
}

type ExactSplit struct{ Shares map[string]int64 }

func (s ExactSplit) Split(total int64, users []string) (map[string]int64, error) {
	r := map[string]int64{}
	for k, v := range s.Shares {
		r[k] = v
	}
	return r, nil
}
```

**How it works and why it belongs here:** Splitter supplies shares; Ledger remains the final authority on member uniqueness, nonnegativity and total equality. ExactSplit copies its map. EqualSplit places leftover minor units in participant order.

### 4.3. `internal/lld/ledger.go`

```go
package lld

import (
	"fmt"
	"sync"
)

type Ledger struct {
	mu       sync.Mutex
	balances map[string]int64
	seen     map[string]bool
}

func NewLedger(users []string) *Ledger {
	l := &Ledger{balances: map[string]int64{}, seen: map[string]bool{}}
	for _, u := range users {
		if u == "" {
			panic("empty user")
		}
		if _, ok := l.balances[u]; ok {
			panic("duplicate user")
		}
		l.balances[u] = 0
	}
	return l
}
func (l *Ledger) AddExpense(id, payer string, total int64, users []string, split Splitter) error {
	l.mu.Lock()
	defer l.mu.Unlock()
	if id == "" {
		return fmt.Errorf("id required")
	}
	if l.seen[id] {
		return nil
	}
	if _, ok := l.balances[payer]; !ok {
		return fmt.Errorf("unknown payer")
	}
	if total <= 0 || len(users) == 0 || split == nil {
		return fmt.Errorf("invalid expense")
	}
	members := map[string]bool{}
	for _, u := range users {
		_, ok := l.balances[u]
		if !ok || members[u] {
			return fmt.Errorf("invalid participant")
		}
		members[u] = true
	}
	shares, err := split.Split(total, append([]string(nil), users...))
	if err != nil {
		return err
	}
	if len(shares) != len(users) {
		return fmt.Errorf("wrong shares")
	}
	remaining := total
	for u, v := range shares {
		if !members[u] || v < 0 || v > remaining {
			return fmt.Errorf("invalid shares")
		}
		remaining -= v
	}
	if remaining != 0 {
		return fmt.Errorf("shares do not sum")
	}
	l.balances[payer] += total
	for u, v := range shares {
		l.balances[u] -= v
	}
	l.seen[id] = true
	return nil
}
func (l *Ledger) Settle(id, from, to string, amount int64) error {
	l.mu.Lock()
	defer l.mu.Unlock()
	if id == "" {
		return fmt.Errorf("id required")
	}
	if l.seen[id] {
		return nil
	}
	_, a := l.balances[from]
	_, b := l.balances[to]
	if !a || !b || from == to || amount <= 0 {
		return fmt.Errorf("invalid settlement")
	}
	l.balances[from] += amount
	l.balances[to] -= amount
	l.seen[id] = true
	return nil
}
func (l *Ledger) Balances() map[string]int64 {
	l.mu.Lock()
	defer l.mu.Unlock()
	r := map[string]int64{}
	for u, v := range l.balances {
		r[u] = v
	}
	return r
}
```

**How it works and why it belongs here:** Ledger validates the entire posting before touching balances. The remaining-total check avoids overflowing a share-sum accumulator. Balances returns a copy so callers cannot corrupt the ledger. Splitters execute under the ledger lock and must be pure and fast.

### 4.4. `cmd/demo/main.go`

```go
package main

import (
	"example.com/splitwise/internal/lld"
	"fmt"
)

func main() {
	l := lld.NewLedger([]string{"Alice", "Bob", "Cara"})
	if err := l.AddExpense("e1", "Alice", 1000, []string{"Alice", "Bob", "Cara"}, lld.EqualSplit{}); err != nil {
		panic(err)
	}
	if err := l.Settle("s1", "Bob", "Alice", 333); err != nil {
		panic(err)
	}
	fmt.Println(l.Balances())
}
```

**How it works and why it belongs here:** The composition root constructs dependencies, runs a concrete scenario, and prints observable results. Error checks make a rejected operation visible rather than silently treating it as success.

### 4.5. `internal/lld/service_test.go`

```go
package lld

import (
	"testing"
)

func TestLedger(t *testing.T) {
	l := NewLedger([]string{"A", "B", "C"})
	users := []string{"A", "B", "C"}
	if e := l.AddExpense("e", "A", 1000, users, EqualSplit{}); e != nil {
		t.Fatal(e)
	}
	_ = l.AddExpense("e", "A", 1000, users, EqualSplit{})
	b := l.Balances()
	if b["A"] != 666 || b["B"] != -333 || b["C"] != -333 {
		t.Fatal(b)
	}
	if e := l.AddExpense("bad", "A", 1000, users, ExactSplit{map[string]int64{"A": 1}}); e == nil {
		t.Fatal("invalid accepted")
	}
	_ = l.Settle("s", "B", "A", 333)
	b = l.Balances()
	if b["B"] != 0 || b["A"]+b["B"]+b["C"] != 0 {
		t.Fatal(b)
	}
}
```

**How it works and why it belongs here:** Checks deterministic remainder handling, idempotency, invalid split rejection and settlement conservation.

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
?   	example.com/splitwise/cmd/demo	[no test files]
ok  	example.com/splitwise/internal/lld	(cached)

map[Alice:333 Bob:0 Cara:-333]
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
