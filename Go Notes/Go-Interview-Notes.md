# Go Interview Notes — Mid-Level Backend (2–5 yrs)

> Target: mid-level Go backend interviews. Covers language core, concurrency & runtime,
> production/backend Go, and DSA idioms. Every section has **what they ask**, **the answer**,
> and **runnable examples**.
>
> Current Go as of writing: **Go 1.26** (released Feb 10, 2026; latest patch 1.26.6, Aug 2026).
> Notes flag version-specific behavior where it matters (esp. Go 1.22 loop vars, 1.23 iterators,
> 1.25 `testing/synctest`, 1.26 Green Tea GC).

## How to use this doc

1. Read a section, then **close it and re-explain the "Interview answer" out loud**. Interviews test recall under pressure, not recognition.
2. The ⚠️ blocks are the classic traps. Interviewers love them because they separate "used Go" from "understand Go."
3. The 🎯 blocks are the exact phrasings that score well.

---

# Table of Contents

**Part 1 — Language Core & Gotchas**
1. Type system, zero values, value semantics
2. Slices (the #1 interview topic)
3. Maps
4. Strings, runes, bytes
5. Structs, embedding, method sets
6. Interfaces (and the nil-interface trap)
7. Generics
8. Errors, panic, recover
9. defer — full semantics
10. Closures & the loop variable
11. Constants, iota, packages, init

**Part 2 — Concurrency & Runtime**
12. Goroutines & the GMP scheduler
13. Channels — complete behavior table
14. select
15. Concurrency patterns
16. sync package & atomics
17. context
18. Go memory model & happens-before
19. Race conditions, deadlocks, goroutine leaks
20. Memory: stack vs heap, escape analysis, GC

**Part 3 — Backend / Production Go**
21. net/http server & routing
22. Middleware
23. Timeouts, graceful shutdown, http.Client
24. JSON
25. database/sql
26. Project layout & dependency injection
27. Testing (table-driven, httptest, mocks, benchmarks, fuzz, synctest)
28. Profiling & pprof
29. Observability (slog, metrics, tracing)
30. Production war stories they ask about

**Part 4 — DSA in Go + Question Bank**
31. Go idioms for coding rounds
32. Classic problems, idiomatic Go
33. Rapid-fire Q&A (100+)
34. 15-minute cram sheet

---

# PART 1 — LANGUAGE CORE & GOTCHAS

## 1. Type system, zero values, value semantics

### The rule that explains half of Go
**Go is 100% pass-by-value. Always. No exceptions.**
When you "pass a pointer," you are passing a *copy of the pointer value*. When you pass a slice,
you pass a *copy of the slice header* (which contains a pointer — hence the aliasing confusion).

### Zero values (memorize this table)

| Type | Zero value | Usable when zero? |
|---|---|---|
| `int`, `float64`, etc. | `0` | yes |
| `string` | `""` | yes |
| `bool` | `false` | yes |
| pointer, `func`, `interface`, `chan`, `map`, slice | `nil` | **depends — see below** |
| struct | all fields zeroed | yes |
| array | all elements zeroed | yes |

**nil usability:**

| Zero type | Read | Write | Notes |
|---|---|---|---|
| nil slice | ✅ `len`, `cap`, `range`, `append` | ✅ `append` works | `var s []int` is idiomatic |
| nil map | ✅ read returns zero value, `len`=0, `range` = 0 iters | ❌ **panic** on write | must `make` before write |
| nil channel | blocks forever | blocks forever | useful in `select` to disable a case |
| nil pointer | ❌ panic on deref | ❌ | but methods with pointer receivers can be called if they don't deref |
| nil interface | `== nil` is true | — | see §6 trap |
| nil func | ❌ panic on call | — | |

```go
var s []int          // nil slice
fmt.Println(len(s), cap(s), s == nil) // 0 0 true
s = append(s, 1)     // fine — append allocates
fmt.Println(s)       // [1]

var m map[string]int // nil map
fmt.Println(m["a"], len(m)) // 0 0   <- reads are safe
// m["a"] = 1        // panic: assignment to entry in nil map
m = make(map[string]int)
m["a"] = 1           // fine
```

🎯 **Interview answer:** "A nil slice is fully usable — `append` handles it. A nil map is read-only;
writing panics. This asymmetry exists because `append` returns a new header, but a map write must
mutate an existing hash table."

### Type conversions vs type assertions
- **Conversion**: `T(v)` — compile-time, between compatible types. `int64(x)`, `[]byte(s)`.
- **Assertion**: `v.(T)` — runtime, only on interface values. `v.(*User)`.

```go
type Celsius float64
type Fahrenheit float64
c := Celsius(100)
// f := Fahrenheit(c)  // legal: same underlying type float64
var f Fahrenheit = Fahrenheit(c)
_ = f
```

⚠️ **Named types don't inherit methods across conversion boundaries in the way you'd expect** —
`type MyInt int` gets *no* methods from `int` (int has none) but if you do
`type MySlice []MyStruct`, MySlice does not get MyStruct's methods.

⚠️ **Defined type vs type alias:**
```go
type MyInt int      // NEW distinct type; needs conversion; can have methods
type MyAlias = int  // ALIAS; identical to int; cannot define methods on it
```
`type A = B` aliases are why `byte = uint8` and `rune = int32`. Go 1.24 added **generic type aliases**.

---

## 2. Slices — the #1 interview topic

### The slice header (know this cold)
```go
type slice struct {
    array unsafe.Pointer // pointer to backing array
    len   int            // number of accessible elements
    cap   int            // elements from array start to end of backing array
}
```
24 bytes on 64-bit. `len(s)` and `cap(s)` are O(1) reads of this struct.

### Arrays vs slices

| | Array | Slice |
|---|---|---|
| Length | part of the type (`[3]int` ≠ `[4]int`) | dynamic |
| Assignment | **copies all elements** | copies 3-word header only |
| Comparable with `==` | yes (if element type is) | **no** (only `s == nil`) |
| Passed to func | full copy | header copy, shares backing array |

```go
a := [3]int{1, 2, 3}
b := a          // FULL COPY
b[0] = 99
fmt.Println(a, b) // [1 2 3] [99 2 3]

s := []int{1, 2, 3}
t := s          // header copy, SAME backing array
t[0] = 99
fmt.Println(s, t) // [99 2 3] [99 2 3]
```

### append: the growth + aliasing rules

**Rule:** `append` writes into the existing backing array **if `cap` allows**. Otherwise it allocates
a new array, copies, and returns a header pointing at the new one.

```go
s := make([]int, 3, 5)   // len=3 cap=5 -> [0 0 0]
t := append(s, 1)        // cap allows: writes into s's array
t[0] = 100
fmt.Println(s, t)        // [100 0 0] [100 0 0 1]  <- ALIASED

u := append(t, 2, 3)     // len would be 6 > cap 5: NEW array
u[0] = 999
fmt.Println(t, u)        // [100 0 0 1] [999 0 0 1 2 3]  <- NOT aliased
```

🎯 **Interview answer:** "`append` may or may not share the backing array with its input depending on
capacity, which makes aliasing non-deterministic from the caller's view. That's why the result must
always be reassigned and why you never rely on whether a mutation is visible through the old slice."

### Growth strategy (Go 1.18+)
- Small slices (`cap < 256`): roughly **double**.
- Larger: grows by ~**1.25×** with a smoothing formula, then rounded up to a size class by the allocator.
- **Do not state "always 2x"** — that's the pre-1.18 answer and interviewers catch it.

```go
var s []int
prev := 0
for i := 0; i < 2000; i++ {
    s = append(s, i)
    if cap(s) != prev {
        fmt.Printf("len=%d cap=%d\n", len(s), cap(s))
        prev = cap(s)
    }
}
// 1,2,4,8,16,32,64,128,256,512,848,1280,1792,2560 (approx; allocator-rounded)
```

### Slicing: `s[low:high:max]`
```go
s := []int{0, 1, 2, 3, 4, 5}
a := s[1:3]      // len=2 cap=5  (cap = cap(s) - low)
b := s[1:3:3]    // len=2 cap=2  <- FULL SLICE EXPRESSION, caps it
b = append(b, 99) // forced to allocate; does NOT touch s
fmt.Println(s)    // [0 1 2 3 4 5] unchanged
```

🎯 The 3-index form is the standard defensive technique when returning an internal slice to a caller:
`return s[a:b:b]` so their `append` cannot stomp your data.

### ⚠️ Top 8 slice traps

**1. Appending in a loop while ranging**
```go
s := []int{1, 2, 3}
for i, v := range s {   // range evaluates s ONCE; len is fixed at 3
    s = append(s, v)
    _ = i
}
fmt.Println(s) // [1 2 3 1 2 3] — loop ran 3 times, not forever
```

**2. Taking the address of the loop variable (pre-Go 1.22)**
```go
// Go < 1.22: all pointers point to the SAME variable
// Go >= 1.22: each iteration gets a NEW variable — fixed
type T struct{ N int }
ts := []T{{1}, {2}, {3}}
var ptrs []*T
for _, t := range ts {
    ptrs = append(ptrs, &t)   // Go<1.22: all point to last; Go>=1.22: correct
}
```
🎯 Say: "Go 1.22 made loop variables per-iteration, which fixed the classic `&v` and goroutine-capture
bugs. Before 1.22 you needed `v := v` shadowing."

**3. Modifying elements via `range` value copy**
```go
for _, t := range ts { t.N = 0 }        // NO EFFECT — t is a copy
for i := range ts   { ts[i].N = 0 }     // correct
```

**4. Memory leak from sub-slicing a large slice**
```go
big := make([]byte, 10<<20) // 10MB
small := big[:10]           // still pins the whole 10MB!
// fix:
small = append([]byte(nil), big[:10]...)   // or slices.Clone(big[:10])
```

**5. `copy` copies `min(len(dst), len(src))` — not cap**
```go
dst := make([]int, 0, 10)
n := copy(dst, []int{1,2,3})
fmt.Println(n, dst) // 0 []  <- len(dst) is 0!
dst = make([]int, 3)
n = copy(dst, []int{1,2,3})
fmt.Println(n, dst) // 3 [1 2 3]
```

**6. nil slice vs empty slice**
```go
var a []int          // nil
b := []int{}         // non-nil, len 0
c := make([]int, 0)  // non-nil, len 0
fmt.Println(a == nil, b == nil) // true false
// JSON: a -> null,  b -> []      <- API-breaking difference!
```
🎯 This bites in APIs: always initialize response slices with `[]T{}` if the client expects `[]`.

**7. Slices are not comparable**
```go
// if a == b {}  // compile error
fmt.Println(slices.Equal(a, b))       // Go 1.21+
fmt.Println(reflect.DeepEqual(a, b))  // slower, reflection
```

**8. Passing a slice to a function**
```go
func mutate(s []int)  { s[0] = 99 }        // visible to caller
func grow(s []int)    { s = append(s, 1) } // NOT visible (header is a copy)
func growOK(s *[]int) { *s = append(*s, 1) }
```

### Removing elements
```go
// order-preserving delete of index i
s = append(s[:i], s[i+1:]...)
// or Go 1.21+
s = slices.Delete(s, i, i+1)

// fast delete, order NOT preserved
s[i] = s[len(s)-1]
s = s[:len(s)-1]
```
⚠️ If the element type contains pointers, zero the tail slot to avoid leaks:
```go
s[len(s)-1] = nil // for []*T
s = s[:len(s)-1]
```

### The `slices` package (Go 1.21+) — know these names
`slices.Contains`, `Index`, `Sort`, `SortFunc`, `SortStableFunc`, `BinarySearch`, `Equal`, `Clone`,
`Reverse`, `Max`, `Min`, `Compact`, `Insert`, `Delete`, `Grow`, `Clip`, `Chunk` (1.23), `Sorted`/`Collect` (1.23, iterators).

---

## 3. Maps

### Internals (30-second version)
- Hash table of **buckets**; each bucket holds up to **8 key/value pairs**.
- A bucket stores 8 **top-hash bytes** first for fast rejection, then 8 keys, then 8 values
  (keys and values grouped separately to avoid padding).
- Overflow buckets chain when a bucket fills.
- Load factor ~6.5 per bucket triggers a **doubling** + **incremental (evacuating) rehash**.
- Go 1.24 replaced the implementation with **Swiss Tables** — faster lookups, less memory. If asked,
  mention both: "classic bucketed chaining, replaced by Swiss Tables in 1.24."

### Key facts
```go
m := map[string]int{"a": 1}

v, ok := m["b"]        // comma-ok idiom: 0, false
delete(m, "a")         // no-op if missing, never panics
clear(m)               // Go 1.21+ builtin
fmt.Println(len(m))
```

⚠️ **Iteration order is deliberately randomized.** Not "unspecified" — actively randomized per run,
so you can't accidentally depend on it. To iterate deterministically:
```go
keys := make([]string, 0, len(m))
for k := range m { keys = append(keys, k) }
slices.Sort(keys)
for _, k := range keys { fmt.Println(k, m[k]) }
// Go 1.23+: for k, v := range maps.All(m) {} ; slices.Sorted(maps.Keys(m))
```

⚠️ **Map values are not addressable.**
```go
type P struct{ N int }
m := map[string]P{"a": {1}}
// m["a"].N = 2      // compile error: cannot assign
p := m["a"]; p.N = 2; m["a"] = p     // fix 1: copy back
m2 := map[string]*P{"a": {1}}
m2["a"].N = 2                         // fix 2: store pointers
```
Reason: the map may rehash and move the value, so a pointer into it would dangle.

⚠️ **Concurrent map access panics.** Not a silent race — the runtime detects it:
`fatal error: concurrent map writes`. This is a **fatal error, not a recoverable panic**.
```go
// fixes:
var mu sync.RWMutex           // 1) guard with a mutex
var sm sync.Map               // 2) sync.Map — only for append-mostly / disjoint-key workloads
```

⚠️ **Deleting during range is safe**; entries not yet reached may or may not be produced.
**Adding during range is safe but new entries may or may not appear.**

⚠️ **Keys must be comparable.** Slices, maps, and funcs cannot be keys. Structs/arrays can be if all
fields are comparable. `interface{}` keys compile but **panic at runtime** if you insert an
uncomparable dynamic type.
```go
var m map[any]int = map[any]int{}
m[[]int{1}] = 1 // panic: runtime error: hash of unhashable type []int
```

⚠️ **NaN keys are pathological**: `NaN != NaN`, so you can insert the same NaN twice and never read it back.

### sync.Map — when (rarely)
Use only when: (a) keys are written once and read many times, or (b) goroutines operate on
**disjoint** key sets. Otherwise a `sync.RWMutex` + plain map is faster and type-safe.
```go
var sm sync.Map
sm.Store("a", 1)
v, ok := sm.Load("a")
v2, loaded := sm.LoadOrStore("b", 2)
sm.Delete("a")
sm.Range(func(k, v any) bool { return true }) // return false to stop
_, _, _, _ = v, ok, v2, loaded
```

---

## 4. Strings, runes, bytes

- `string` is an **immutable** 2-word header: `{ptr *byte, len int}`. Not null-terminated.
- Go source is UTF-8; strings hold **arbitrary bytes** but are conventionally UTF-8.
- `len(s)` = **bytes**, not characters.
- `s[i]` = **byte** (`uint8`).
- `for i, r := range s` decodes **runes** (`int32` code points); `i` is the **byte offset**, and it
  jumps by the rune width.

```go
s := "héllo"           // 'é' is 2 bytes in UTF-8
fmt.Println(len(s))                    // 6
fmt.Println(utf8.RuneCountInString(s)) // 5
fmt.Println(s[1])                      // 195 — half of 'é'!
for i, r := range s {
    fmt.Printf("%d:%c ", i, r)         // 0:h 1:é 3:l 4:l 5:o
}
r := []rune(s)                         // allocates; len 5
b := []byte(s)                         // allocates; len 6
fmt.Println(string(r[1]))              // é
```

⚠️ **Reversing a string** — the classic trick question:
```go
func reverse(s string) string {
    r := []rune(s)                     // MUST convert to runes, not bytes
    for i, j := 0, len(r)-1; i < j; i, j = i+1, j-1 {
        r[i], r[j] = r[j], r[i]
    }
    return string(r)
}
```
🎯 Bonus point: "This still breaks on combining characters and emoji ZWJ sequences — true
grapheme-cluster reversal needs `golang.org/x/text/unicode/norm` or a segmentation library."

### Concatenation performance
```go
// BAD: O(n²), allocates each iteration
out := ""
for _, w := range words { out += w }

// GOOD: strings.Builder — amortized O(n), no copy on String()
var sb strings.Builder
sb.Grow(estimatedSize)   // optional but great
for _, w := range words { sb.WriteString(w) }
out = sb.String()

// ALSO GOOD for joining with a separator
out = strings.Join(words, ",")
```
`strings.Builder` beats `bytes.Buffer` for string output because `String()` uses `unsafe` to avoid
the final copy. (Builder is not safe to copy after first use — `go vet` catches it.)

### Zero-copy conversions
`[]byte(s)` and `string(b)` normally **allocate and copy**. Compiler optimizes some cases:
`m[string(b)]` map lookup, `string(b) == "x"` comparison, `for range string(b)`.
Go 1.20+ safe(ish) explicit versions: `unsafe.String`, `unsafe.SliceData`.

### Common string API
`strings.Split/SplitN/Fields/TrimSpace/Trim/TrimPrefix/TrimSuffix/HasPrefix/HasSuffix/Contains/Index/Replace/ReplaceAll/ToLower/EqualFold/Cut/Repeat`.
`strings.Cut` (1.18) is the idiomatic split-once:
```go
before, after, found := strings.Cut("key=value", "=")
```

---

## 5. Structs, embedding, method sets

### Struct basics
```go
type User struct {
    ID    int    `json:"id" db:"id"`
    Name  string `json:"name"`
    email string // unexported
}
u := User{ID: 1, Name: "A"}          // always use field names
p := &User{ID: 2}                    // &T{} is idiomatic; no `new` needed
```
- Structs are **comparable with `==` iff all fields are comparable**.
- **Empty struct `struct{}{}` occupies 0 bytes** — used for sets (`map[string]struct{}`) and
  signal-only channels (`chan struct{}`).

### Field alignment / memory layout (asked at mid+ level)
```go
type Bad  struct { a bool; b int64; c bool }  // 24 bytes (7+7 padding)
type Good struct { b int64; a bool; c bool }  // 16 bytes
```
🎯 "Order fields largest-to-smallest to minimize padding. `fieldalignment` in `go vet`'s
analysis tooling flags it. Matters when you have millions of structs, not otherwise."

### Embedding (composition, NOT inheritance)
```go
type Animal struct{ Name string }
func (a Animal) Speak() string { return a.Name + " makes a sound" }

type Dog struct {
    Animal        // embedded — promotes fields and methods
    Breed string
}

d := Dog{Animal{"Rex"}, "Lab"}
fmt.Println(d.Name)      // promoted field
fmt.Println(d.Speak())   // promoted method
fmt.Println(d.Animal.Name) // explicit path also works
```
⚠️ **No virtual dispatch.** If `Dog` overrides `Speak`, `Animal`'s other methods still call
`Animal.Speak`, not `Dog.Speak`. There is no polymorphism through embedding — only through interfaces.
```go
func (d Dog) Speak() string { return d.Name + " barks" }
// Animal.Describe() calling a.Speak() would still get Animal's version.
```

⚠️ **Ambiguous promotion** at the same depth is a compile error only when referenced;
shallower depth wins over deeper.

**Interface embedding** is the same mechanism:
```go
type ReadWriter interface { io.Reader; io.Writer }
```
**Embedding an interface in a struct** is a common mock trick:
```go
type partialStore struct{ Store }        // embeds the interface, all methods panic-nil
func (p partialStore) Get(id int) (*User, error) { return &User{}, nil } // override one
```

### Method sets — THE rule
> The method set of `T` contains methods with **value receivers**.
> The method set of `*T` contains methods with **value AND pointer receivers**.

Consequence: **`*T` satisfies more interfaces than `T`.**

```go
type Speaker interface{ Speak() }
type Cat struct{}
func (c *Cat) Speak() {}         // POINTER receiver

var s Speaker = &Cat{}           // OK
// var s2 Speaker = Cat{}        // COMPILE ERROR: Cat does not implement Speaker
                                 // (Speak has pointer receiver)
```
But direct calls work due to **automatic addressing** on addressable values:
```go
c := Cat{}
c.Speak()          // OK — compiler rewrites to (&c).Speak()
// Cat{}.Speak()   // ERROR — composite literal is not addressable
// m["k"].Speak()  // ERROR — map value not addressable
```

🎯 **Interview answer:** "Method sets matter only for interface satisfaction. For direct calls the
compiler auto-takes the address of addressable values. The reason `T` doesn't get pointer methods is
that an interface holding a `T` holds a copy — there'd be no original to mutate."

### Value vs pointer receiver — how to choose
Use **pointer receiver** when:
- the method mutates the receiver
- the struct is large (avoid copying)
- the type contains a `sync.Mutex` or other non-copyable field
- **consistency**: if any method needs a pointer receiver, use pointer receivers for all of them

Use **value receiver** when: small, immutable, primitive-like types (`time.Time`, small value objects).

⚠️ Copying a struct containing a `sync.Mutex` copies the lock state — `go vet` flags this.

---

## 6. Interfaces

### Implicit satisfaction
No `implements` keyword. A type satisfies an interface by having the methods. This enables
**consumer-defined interfaces**: define the interface where you *use* it, not where you implement it.

🎯 "Accept interfaces, return structs" — and keep interfaces small. `io.Reader` has one method.

```go
// GOOD: narrow interface defined by the consumer
type UserGetter interface {
    GetUser(ctx context.Context, id int) (*User, error)
}
func Handler(g UserGetter) http.HandlerFunc { /* ... */ return nil }

// Compile-time assertion that *pgStore satisfies it:
var _ UserGetter = (*pgStore)(nil)
```

### Interface internals
Two runtime representations:
```go
// non-empty interface
type iface struct {
    tab  *itab           // {interface type, concrete type, method pointers}
    data unsafe.Pointer  // pointer to the value
}
// empty interface (any)
type eface struct {
    _type *_type
    data  unsafe.Pointer
}
```
Both are **2 words**. Storing a value in an interface usually **allocates** (the value escapes to
the heap) — except small integers 0–255 and pointer-shaped values, which are optimized.

### ⚠️ THE nil interface trap (asked constantly)
```go
type MyErr struct{}
func (e *MyErr) Error() string { return "boom" }

func mayFail() error {
    var e *MyErr = nil     // typed nil pointer
    return e               // wrapped in a non-nil interface!
}

err := mayFail()
fmt.Println(err == nil)    // FALSE  😱
```
**Why:** an interface is nil only when **both** the type word and the data word are nil.
Returning a typed nil pointer sets the type word, so the interface is non-nil.

**Fix:**
```go
func mayFail() error {
    var e *MyErr
    if somethingBad { e = &MyErr{} }
    if e != nil { return e }
    return nil            // return an untyped nil literal
}
```
🎯 Bonus: "This is why you never declare a function's return as a concrete error type; always
declare it as `error`."

### Type assertion & type switch
```go
var v any = "hello"

s := v.(string)          // panics if wrong
s, ok := v.(string)      // comma-ok, never panics

switch x := v.(type) {
case nil:
    fmt.Println("nil")
case string:
    fmt.Println("string", len(x))
case int, int64:
    fmt.Println("some int", x)   // x is `any` here (multi-type case)
case error:
    fmt.Println("error", x.Error())
case fmt.Stringer:
    fmt.Println(x.String())
default:
    fmt.Printf("%T\n", x)
}
```
⚠️ In a multi-type `case int, int64:` the bound variable keeps the **interface** type.

### Interfaces & performance
- Method calls through an interface are **indirect** (itab lookup) — cheap but not inlinable.
- Prefer concrete types in hot loops.
- `any` boxing allocates. `fmt.Println(x)` allocates because of `...any`.

### Embedding + interface satisfaction combo question
```go
type Base struct{}
func (Base) Foo() {}
type Derived struct{ Base }
var _ interface{ Foo() } = Derived{}   // ✅ promoted method satisfies it
```

---

## 7. Generics (Go 1.18+)

```go
type Number interface {
    ~int | ~int64 | ~float64      // ~ means "any type whose underlying type is"
}

func Sum[T Number](xs []T) T {
    var total T
    for _, x := range xs { total += x }
    return total
}

func Map[T, U any](in []T, f func(T) U) []U {
    out := make([]U, 0, len(in))
    for _, v := range in { out = append(out, f(v)) }
    return out
}

// Generic type
type Stack[T any] struct{ items []T }
func (s *Stack[T]) Push(v T) { s.items = append(s.items, v) }
func (s *Stack[T]) Pop() (T, bool) {
    var zero T
    if len(s.items) == 0 { return zero, false }
    v := s.items[len(s.items)-1]
    s.items = s.items[:len(s.items)-1]
    return v, true
}
```

**Key points to say:**
- Constraints are interfaces. `any` and `comparable` are predeclared.
  `comparable` = usable with `==` (Go 1.20 loosened it to allow interface types that are
  comparable at runtime).
- `~T` = **underlying type**, so `type MyInt int` satisfies `~int`.
- **Type inference** usually lets you omit `[T]` at call sites.
- **Methods cannot have their own type parameters.** `func (s Stack[T]) Map[U any](...)` is illegal.
  This is why `Map` is a free function.
- Implementation: **GC shape stenciling** — one instantiation per "shape" (all pointer types share
  one), with a dictionary passed for type-specific info. So generics are *not* free but not
  full monomorphization either. Generic code is often **slightly slower** than concrete code and
  can block inlining.
- `golang.org/x/exp/constraints` has `Ordered`, `Integer`, `Float`; `cmp.Ordered` is the stdlib
  version (Go 1.21).

**When to use generics:** container types, and functions over slices/maps/channels.
**When not to:** if an interface with a method works, use the interface. Generics don't replace
polymorphism.

### Go 1.23 iterators (range-over-func) — worth knowing
```go
func Backwards[T any](s []T) iter.Seq2[int, T] {
    return func(yield func(int, T) bool) {
        for i := len(s) - 1; i >= 0; i-- {
            if !yield(i, s[i]) { return }
        }
    }
}
for i, v := range Backwards([]string{"a","b","c"}) { fmt.Println(i, v) }
```
Signatures: `iter.Seq[V] = func(yield func(V) bool)`, `iter.Seq2[K,V] = func(yield func(K,V) bool)`.
Used by `maps.Keys`, `maps.Values`, `slices.All`, `slices.Values`, `slices.Sorted`.

---

## 8. Errors, panic, recover

### Errors are values
```go
type error interface { Error() string }
```

### Creating errors
```go
errors.New("not found")
fmt.Errorf("loading user %d: %w", id, err)   // %w WRAPS
fmt.Errorf("loading user %d: %v", id, err)   // %v does NOT wrap (chain broken)
```

### Sentinel errors
```go
var ErrNotFound = errors.New("not found")

func Get(id int) (*User, error) {
    return nil, fmt.Errorf("get user %d: %w", id, ErrNotFound)
}

if errors.Is(err, ErrNotFound) { /* handles wrapped too */ }
```

### Custom error types
```go
type ValidationError struct {
    Field string
    Msg   string
}
func (e *ValidationError) Error() string {
    return fmt.Sprintf("field %q: %s", e.Field, e.Msg)
}

var ve *ValidationError
if errors.As(err, &ve) {
    fmt.Println(ve.Field)
}
// Go 1.26: errors.AsType — generic, type-safe:
// if ve, ok := errors.AsType[*ValidationError](err); ok { ... }
```

**`errors.Is` vs `errors.As`:**
- `Is` — "is this error (or anything it wraps) **equal to** this sentinel?" (or matches its `Is` method)
- `As` — "is there an error in the chain of **this type**? if so assign it to my variable"

### Wrapping multiple errors (Go 1.20+)
```go
err := errors.Join(err1, err2)          // nil-safe; skips nils; nil if all nil
fmt.Errorf("a: %w, b: %w", e1, e2)      // multiple %w allowed since 1.20
errors.Unwrap(err)                       // single unwrap; nil if not wrapped
```

### Error-handling style
```go
// idiomatic: handle immediately, keep the happy path un-indented
u, err := repo.Get(ctx, id)
if err != nil {
    return nil, fmt.Errorf("service.GetUser: %w", err)
}
```
🎯 Guidelines that score well:
- **Handle once**: either log it or return it, not both.
- Add context at each layer, but **don't include the word "error"** — `fmt.Errorf("read config: %w", err)`.
- **Wrap by default** unless you're deliberately hiding an implementation detail from callers
  (wrapping makes the inner error part of your API contract).
- Sentinel errors for control-flow decisions; typed errors when the caller needs structured data.

### panic / recover
Use panic for **programmer errors** (impossible states), not for expected failures.

```go
func safeDiv(a, b int) (result int, err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("recovered: %v", r)
        }
    }()
    return a / b, nil   // panics if b == 0
}
```
Rules:
- `recover()` only works **inside a deferred function**, called directly by it.
- Recovering converts the panic into a normal return; **named return values** are how you surface
  it as an error.
- Panic runs deferred functions up the stack; unrecovered, it kills the program.
- ⚠️ **A panic in one goroutine crashes the whole program.** You cannot recover another goroutine's
  panic from the parent. Every goroutine that runs untrusted work needs its own recover.
```go
go func() {
    defer func() { if r := recover(); r != nil { log.Println("worker panic:", r) } }()
    doWork()
}()
```
- ⚠️ **Fatal errors are NOT recoverable**: concurrent map write, all goroutines asleep (deadlock),
  stack exhaustion, out of memory.
- `net/http` already recovers panics **per request** (logs and closes the connection), but you
  usually add your own middleware so you can return a 500 and record metrics.

---

## 9. defer — full semantics

**Four rules:**
1. **LIFO** — deferred calls run in reverse order.
2. **Arguments are evaluated at defer time**, the call happens later.
3. Runs when the **function** returns (not the block) — including on panic.
4. Can **modify named return values**.

```go
func demo() (result int) {
    defer fmt.Println("1st deferred, runs 3rd")
    defer fmt.Println("2nd deferred, runs 2nd")
    defer func() { result *= 2 }()     // runs 1st, modifies return
    return 5                            // result=5, then defers -> returns 10
}
```

```go
i := 0
defer fmt.Println("arg evaluated now:", i)  // prints 0
i = 10
// output: arg evaluated now: 0
defer func() { fmt.Println("closure sees:", i) }() // prints 10
```

⚠️ **defer in a loop** accumulates until the *function* returns:
```go
// BAD: holds every file open until the function ends
for _, name := range names {
    f, _ := os.Open(name)
    defer f.Close()
}
// GOOD: wrap in a function scope
for _, name := range names {
    func() {
        f, _ := os.Open(name)
        defer f.Close()
        process(f)
    }()
}
```

⚠️ **Don't ignore Close errors on writes:**
```go
func write(path string, data []byte) (err error) {
    f, err := os.Create(path)
    if err != nil { return err }
    defer func() {
        if cerr := f.Close(); cerr != nil && err == nil {
            err = cerr        // a failed flush must surface
        }
    }()
    _, err = f.Write(data)
    return err
}
```

⚠️ **defer on a nil resource** panics when it runs. Always check the error first:
```go
resp, err := http.Get(url)
if err != nil { return err }     // MUST come before the defer
defer resp.Body.Close()
```

**Performance:** defer used to cost ~50ns; since Go 1.14 **open-coded defers** make it near-free
(~1ns) when the defer count is static and there's no defer in a loop. Don't avoid defer for perf.

`return` in a function with named results is: `assign results` → `run defers` → `actually return`.

---

## 10. Closures & the loop variable

```go
func counter() func() int {
    count := 0                 // captured BY REFERENCE, escapes to heap
    return func() int { count++; return count }
}
c := counter()
fmt.Println(c(), c(), c())     // 1 2 3
```

### The famous goroutine-loop bug
```go
// Go < 1.22: prints 3 3 3 (or nondeterministic)
// Go >= 1.22: prints 0 1 2 in some order
for i := 0; i < 3; i++ {
    go func() { fmt.Println(i) }()
}
```
**Pre-1.22 fixes** (still good to show you know them):
```go
for i := 0; i < 3; i++ {
    i := i                                // shadow
    go func() { fmt.Println(i) }()
}
for i := 0; i < 3; i++ {
    go func(i int) { fmt.Println(i) }(i)  // pass as arg
}
```
🎯 "Go 1.22 changed `for` loop variables to be per-iteration. It's gated by the `go` directive in
go.mod, so a module declaring `go 1.21` keeps the old semantics even on a new toolchain."

⚠️ Still a bug in any Go version — no `WaitGroup`:
```go
var wg sync.WaitGroup
for i := 0; i < 3; i++ {
    wg.Add(1)
    go func() { defer wg.Done(); fmt.Println(i) }()
}
wg.Wait()
// Go 1.25+: wg.Go(func(){ fmt.Println(i) })  — Add/Done handled for you
```

---

## 11. Constants, iota, packages, init

### Constants
Untyped constants have **arbitrary precision** and adapt to context.
```go
const Big = 1 << 62          // fine as an untyped constant
const Pi = 3.14159265358979323846264338327950288419716939937510582097494459
var f float64 = Pi           // truncated at use
// var i int = Big << 10     // compile error: overflows int
```

### iota
```go
type Weekday int
const (
    Sunday Weekday = iota   // 0
    Monday                  // 1
    Tuesday                 // 2
)

// Skipping and expressions
const (
    _  = iota             // skip 0
    KB = 1 << (10 * iota) // 1024
    MB                    // 1048576
    GB
)

// Bit flags
type Perm uint8
const (
    Read Perm = 1 << iota  // 1
    Write                  // 2
    Exec                   // 4
)
```
⚠️ `iota` resets to 0 at each `const` block and increments **per ConstSpec line**, including blank lines
that contain a spec — but not comments/empty lines.

🎯 Pair enums with `String()` via `stringer`: `//go:generate stringer -type=Weekday`.

### Packages & visibility
- Capitalized identifier = **exported**. That's the entire access-control system.
- Package name should be short, lowercase, no underscores; the import path's last element.
- ⚠️ **Import cycles are a compile error** — no forward declarations. Break them by extracting a
  shared package or by defining the interface in the consumer.
- `internal/` — importable only by code rooted at the parent of `internal`. Real enforcement.
- Blank import `import _ "github.com/lib/pq"` runs the package's `init` for side effects (driver registration).

### Initialization order
1. Imported packages initialize first (depth-first, each exactly once).
2. Within a package: package-level **variables** in dependency order (not declaration order).
3. Then **all `init()` functions** in the package, in file order (files sorted by name as given to
   the compiler).
4. Finally `main()`.

```go
var a = b + 1   // a = 3 (b initialized first, because a depends on b)
var b = 2

func init() { fmt.Println("init 1") }
func init() { fmt.Println("init 2") }   // multiple init() per file allowed
```
🎯 Prefer explicit constructors over `init()`; `init` makes testing and ordering hard. Acceptable
uses: driver registration, `flag` setup in `main`, precomputed tables.
---

# PART 2 — CONCURRENCY & RUNTIME

> This is where mid-level Go interviews are won or lost. Expect at least 40% of the interview here.

## 12. Goroutines & the GMP scheduler

### Goroutines vs OS threads

| | OS thread | Goroutine |
|---|---|---|
| Initial stack | 1–8 MB, fixed | **2 KB**, grows/shrinks |
| Created by | kernel | Go runtime (user space) |
| Context switch | ~1–2 µs, kernel trap | ~100–200 ns, 3 register saves |
| Scheduling | preemptive, kernel | cooperative + async preemption, runtime |
| Practical count | thousands | **millions** |
| Identity | has a TID | **no ID exposed** (deliberately) |

🎯 "Goroutines are M:N green threads. The runtime multiplexes many goroutines onto few OS threads,
so blocking on a channel costs a user-space switch, not a syscall."

### GMP model — know the three letters
- **G** = goroutine: stack, program counter, status.
- **M** = machine: an OS thread. Must hold a P to run Go code.
- **P** = processor: a scheduling context holding a **local run queue** (256 slots) and an mcache.
  Count = `GOMAXPROCS`. **P is what limits parallelism of Go code**, not M.

```
        ┌── P0 ──[local runq: G G G]──> M0 (OS thread) ──> CPU
GRQ ────┤
[G G G] └── P1 ──[local runq: G G]────> M1 (OS thread) ──> CPU

           M2 (blocked in syscall, no P)
```

**Key mechanics to name-drop:**
- **Work stealing**: an idle P steals **half** of another P's local queue; also checks the global
  run queue and netpoller. Every ~61 scheduler ticks it checks the global queue to avoid starvation.
- **runnext slot**: a newly created/unblocked goroutine goes into a special 1-slot `runnext` for
  locality (helps ping-pong patterns).
- **Handoff on blocking syscall**: if an M blocks in a syscall >~20µs, the sysmon thread detaches
  the P and hands it to another M so work continues.
- **Netpoller**: network I/O does **not** block an M. `epoll`/`kqueue`/IOCP parks the goroutine and
  wakes it when ready. This is why Go handles 100k connections cheaply — but **file I/O and cgo
  calls DO block an M**.
- **Async preemption (Go 1.14+)**: sysmon sends `SIGURG` to preempt goroutines running >10ms.
  Before 1.14, a tight loop with no function calls could hang the scheduler forever.
- **sysmon**: a dedicated M with no P; retakes Ps from long syscalls, forces preemption, triggers GC.
- **Spinning threads**: a few Ms spin looking for work rather than sleeping, to cut wakeup latency.

### GOMAXPROCS
```go
runtime.GOMAXPROCS(0)   // read current
runtime.NumCPU()        // machine/cgroup CPUs
runtime.NumGoroutine()  // live goroutines
```
Default = number of CPUs. ⚠️ **Classic production bug:** in a container with a 0.5-CPU limit but a
64-core host, pre-Go-1.25 Go set GOMAXPROCS=64 → massive scheduler churn and CPU throttling. Fix was
`automaxprocs` (Uber). **Go 1.25 made the runtime cgroup-aware**, reading the CPU limit by default.

🎯 If asked "how many goroutines for CPU-bound work?" → `GOMAXPROCS` (or `NumCPU`). For I/O-bound →
much higher; bound it by the downstream resource (DB connections, rate limit), not by CPU.

### Goroutine stacks
Start at 2 KB. On overflow, the runtime allocates a **2× stack, copies it, and rewrites pointers**
(contiguous stacks, Go 1.4+; the old segmented stacks caused "hot split" thrashing). Max default
1 GB (64-bit) — exceed it and you get `stack overflow`, a **fatal, unrecoverable** error. GC can also
shrink stacks.

---

## 13. Channels — complete behavior table

### MEMORIZE THIS

| Operation | nil channel | open & empty | open & has data | open & full | closed |
|---|---|---|---|---|---|
| **Receive** `<-ch` | **block forever** | block | returns value | returns value | **returns zero value immediately, ok=false** |
| **Send** `ch<-v` | **block forever** | succeeds (buffered) / blocks until receiver (unbuffered) | succeeds if buffered | block | **PANIC** |
| **Close** | **PANIC** | ok | ok | ok | **PANIC** |
| `len` / `cap` | 0 / 0 | 0 / cap | n / cap | cap / cap | remaining / cap |

🎯 Three panics to recite: **send on closed, close of closed, close of nil.**

### Unbuffered vs buffered
```go
ch := make(chan int)      // unbuffered: SYNCHRONOUS rendezvous
                          // sender blocks until a receiver is ready — a handoff + a sync point
bch := make(chan int, 3)  // buffered: ASYNCHRONOUS until full
```
🎯 "Unbuffered channels give you a happens-before guarantee in both directions — the send
completes only when the receive begins. Buffered channels decouple the two, so you get
back-pressure but weaker synchronization."

### Ownership rule (the single most useful heuristic)
> **The goroutine that creates a channel owns it: it is the only one that writes to it and the only
> one that closes it.** Consumers only receive.

```go
func produce(n int) <-chan int {         // returns receive-only: enforces ownership
    out := make(chan int)
    go func() {
        defer close(out)                  // producer closes
        for i := 0; i < n; i++ { out <- i }
    }()
    return out
}
for v := range produce(3) { fmt.Println(v) }  // range ends when closed
```

### Detecting closure
```go
v, ok := <-ch      // ok == false means closed AND drained
for v := range ch  // exits automatically on close; PANICS never — but blocks forever if never closed
```
⚠️ **`range ch` over a channel nobody closes is a goroutine leak / deadlock.**

### Directional channel types
```go
func send(ch chan<- int)  {}   // send-only
func recv(ch <-chan int)  {}   // receive-only
```
A bidirectional `chan T` converts implicitly to either; not the reverse. Use these in signatures —
free compile-time documentation and it prevents a consumer from closing your channel.

### Closing with multiple senders
⚠️ You can't have N senders close the channel (double-close panic). Patterns:
- **Add a `done` channel**; senders `select` on it and return; a coordinator closes `done`.
- **Use a `sync.WaitGroup`** and have a separate goroutine close after `wg.Wait()`.
```go
var wg sync.WaitGroup
out := make(chan int)
for i := 0; i < 3; i++ {
    wg.Add(1)
    go func(i int) { defer wg.Done(); out <- i }(i)
}
go func() { wg.Wait(); close(out) }()   // closer goroutine
for v := range out { fmt.Println(v) }
```

### `chan struct{}` for signaling
```go
done := make(chan struct{})
go func() { defer close(done); work() }()
<-done                      // wait
```
Zero bytes per element, and **closing broadcasts to every waiter** — that's the trick: close is the
only way to notify N receivers at once with a channel.

---

## 14. select

```go
select {
case v := <-ch1:
    use(v)
case ch2 <- x:
    // sent
case <-time.After(time.Second):
    // timeout
case <-ctx.Done():
    return ctx.Err()
default:
    // non-blocking: runs immediately if nothing else is ready
}
```

**Rules:**
- All cases are evaluated; if **multiple are ready, one is chosen uniformly at random** (prevents
  starvation, and stops you from relying on ordering).
- With `default`, select **never blocks**.
- **`select {}` with no cases blocks forever** — `fatal error: all goroutines are asleep`.
- **nil channel cases are never selected** — the idiom for dynamically disabling a case:

```go
// Disable a case once its channel is drained (classic fan-in)
func merge(a, b <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for a != nil || b != nil {
            select {
            case v, ok := <-a:
                if !ok { a = nil; continue }   // disable this case
                out <- v
            case v, ok := <-b:
                if !ok { b = nil; continue }
                out <- v
            }
        }
    }()
    return out
}
```

### Non-blocking send/receive
```go
select {
case ch <- v:
default:            // drop if nobody's listening (e.g. metrics, best-effort notify)
}

select {
case v := <-ch: use(v)
default:            // no data right now
}
```

### Timeouts
```go
select {
case v := <-ch: use(v)
case <-time.After(2 * time.Second): return ErrTimeout
}
```
⚠️ `time.After` in a **loop leaks a timer** until it fires (Go <1.23 kept it alive for the full
duration). Use a reusable timer:
```go
t := time.NewTimer(2 * time.Second)
defer t.Stop()
for {
    select {
    case v := <-ch:
        use(v)
        if !t.Stop() { <-t.C }        // drain before reset
        t.Reset(2 * time.Second)
    case <-t.C:
        return ErrTimeout
    }
}
```
🎯 "Go 1.23 changed timers so unreferenced ones are GC'd immediately and `t.C` is unbuffered, which
removed the stale-value problem after `Reset`. But the reusable-timer pattern is still what I write
for long-lived loops."

---

## 15. Concurrency patterns (know these by name and be able to write them)

### 15.1 Worker pool (bounded concurrency) — the most-asked one
```go
func workerPool(ctx context.Context, jobs []Job, n int) ([]Result, error) {
    jobCh := make(chan Job)
    resCh := make(chan Result)

    var wg sync.WaitGroup
    for i := 0; i < n; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for j := range jobCh {
                select {
                case resCh <- process(j):
                case <-ctx.Done():
                    return
                }
            }
        }()
    }

    // feeder
    go func() {
        defer close(jobCh)
        for _, j := range jobs {
            select {
            case jobCh <- j:
            case <-ctx.Done():
                return
            }
        }
    }()

    go func() { wg.Wait(); close(resCh) }()

    var out []Result
    for r := range resCh { out = append(out, r) }
    return out, ctx.Err()
}
```

### 15.2 Semaphore (bounded concurrency, simpler)
```go
sem := make(chan struct{}, 10)      // max 10 in flight
var wg sync.WaitGroup
for _, u := range urls {
    wg.Add(1)
    go func(u string) {
        defer wg.Done()
        sem <- struct{}{}            // acquire
        defer func() { <-sem }()     // release
        fetch(u)
    }(u)
}
wg.Wait()
```
Also: `golang.org/x/sync/semaphore` for weighted semaphores.

### 15.3 errgroup — what you should actually reach for
```go
import "golang.org/x/sync/errgroup"

g, ctx := errgroup.WithContext(ctx)
g.SetLimit(10)                       // bounded concurrency, built in
results := make([]Result, len(items))
for i, item := range items {
    i, item := i, item               // unnecessary in Go 1.22+
    g.Go(func() error {
        r, err := fetch(ctx, item)
        if err != nil { return err }  // first error cancels ctx for everyone
        results[i] = r                // safe: disjoint indices, no mutex needed
        return nil
    })
}
if err := g.Wait(); err != nil { return nil, err }
```
🎯 Two things that score: **`WithContext` cancels siblings on first error**, and **writing to
disjoint slice indices needs no lock** (distinct memory locations, and `Wait` is the
happens-before edge for the read).

### 15.4 Pipeline (stages)
```go
func gen(nums ...int) <-chan int {
    out := make(chan int)
    go func() { defer close(out); for _, n := range nums { out <- n } }()
    return out
}
func sq(in <-chan int) <-chan int {
    out := make(chan int)
    go func() { defer close(out); for n := range in { out <- n * n } }()
    return out
}
for v := range sq(sq(gen(1, 2, 3))) { fmt.Println(v) }  // 1 16 81
```

### 15.5 Fan-out / fan-in
Fan-out = multiple goroutines read from one channel. Fan-in = merge N channels into one (see the
`merge` example in §14, or a WaitGroup-based merge for N inputs).

### 15.6 Rate limiting
```go
lim := rate.NewLimiter(rate.Limit(10), 20)  // golang.org/x/time/rate: 10/s, burst 20
if err := lim.Wait(ctx); err != nil { return err }

// stdlib-only ticker version
tick := time.NewTicker(100 * time.Millisecond)
defer tick.Stop()
for range requests { <-tick.C; send() }
```

### 15.7 singleflight (dedupe concurrent identical work — great answer for cache stampede)
```go
import "golang.org/x/sync/singleflight"
var g singleflight.Group
v, err, shared := g.Do(key, func() (any, error) { return loadFromDB(key) })
```

### 15.8 Timeout / cancellation of a blocking op
```go
func doWithTimeout(ctx context.Context, d time.Duration) error {
    ctx, cancel := context.WithTimeout(ctx, d)
    defer cancel()
    done := make(chan error, 1)              // BUFFERED — prevents goroutine leak
    go func() { done <- slowOp() }()
    select {
    case err := <-done: return err
    case <-ctx.Done():  return ctx.Err()     // slowOp keeps running but doesn't leak
    }
}
```
⚠️ The buffer size 1 is the whole point: if the timeout fires, nobody receives, and an unbuffered
send would block that goroutine forever.

### 15.9 Graceful "or-done" wrapper
```go
func orDone[T any](ctx context.Context, c <-chan T) <-chan T {
    out := make(chan T)
    go func() {
        defer close(out)
        for {
            select {
            case <-ctx.Done(): return
            case v, ok := <-c:
                if !ok { return }
                select { case out <- v: case <-ctx.Done(): return }
            }
        }
    }()
    return out
}
```

---

## 16. sync package & atomics

### sync.Mutex
```go
type Counter struct {
    mu sync.Mutex
    n  int
}
func (c *Counter) Inc() { c.mu.Lock(); defer c.mu.Unlock(); c.n++ }
func (c *Counter) Get() int { c.mu.Lock(); defer c.mu.Unlock(); return c.n }
```
- **Not reentrant.** Locking twice from the same goroutine deadlocks. (Go has no recursive mutex by
  design — "if you need reentrancy, your critical section is wrong".)
- **Not tied to a goroutine**: goroutine A can `Lock` and B can `Unlock` (used by semaphore-ish code),
  but unlocking an unlocked mutex is a **fatal error**.
- **Never copy a locked mutex** — embed it by pointer or always use pointer receivers. `go vet` catches this.
- Implementation: spins briefly, then falls back to a futex-like park. Has **normal mode**
  (barging — a new arrival can steal the lock, better throughput) and **starvation mode**
  (kicks in after a waiter waits >1ms; strict FIFO handoff, better tail latency).

### sync.RWMutex
```go
var mu sync.RWMutex
mu.RLock(); _ = cache[k]; mu.RUnlock()      // many concurrent readers
mu.Lock();  cache[k] = v; mu.Unlock()       // exclusive writer
```
- A pending `Lock()` **blocks new `RLock()`s** to prevent writer starvation.
- ⚠️ **RLock is not upgradeable.** `RLock` then `Lock` in the same goroutine = deadlock.
- ⚠️ RWMutex is **slower than Mutex** for short critical sections (more atomic ops + cache-line
  contention on the reader counter). Only wins with genuinely read-heavy, non-trivial critical
  sections. Benchmark before claiming it's faster.

### sync.WaitGroup
```go
var wg sync.WaitGroup
for i := 0; i < 3; i++ {
    wg.Add(1)                              // Add BEFORE `go`
    go func() { defer wg.Done(); work() }()
}
wg.Wait()
// Go 1.25+: wg.Go(work)  — does Add(1)/Done() for you
```
⚠️ Calling `wg.Add` **inside** the goroutine races with `Wait` and can let `Wait` return early.
⚠️ Counter going negative = **panic**.
⚠️ Don't reuse a WaitGroup for a new round until `Wait` has returned.

### sync.Once
```go
var (
    once sync.Once
    db   *sql.DB
)
func DB() *sql.DB {
    once.Do(func() { db = connect() })      // exactly once, even under concurrency
    return db
}
```
- Blocks other callers until the first `f` returns.
- ⚠️ If `f` panics, the Once is **still considered done** — it won't retry.
- Go 1.21 added `sync.OnceFunc`, `OnceValue`, `OnceValues` — cleaner:
```go
var DB = sync.OnceValue(func() *sql.DB { return connect() })
```

### sync.Pool
```go
var bufPool = sync.Pool{
    New: func() any { return new(bytes.Buffer) },
}
func handle(w io.Writer) {
    b := bufPool.Get().(*bytes.Buffer)
    defer func() { b.Reset(); bufPool.Put(b) }()  // ALWAYS Reset before Put
    b.WriteString("hi")
    b.WriteTo(w)
}
```
- Purpose: **reduce GC pressure** for short-lived, frequently allocated objects.
- ⚠️ **Items may be evicted at any GC.** Never store anything you need to survive.
- ⚠️ Don't pool objects of wildly varying size (a giant buffer keeps its capacity forever).
- Per-P local storage, so it scales, but `Get`/`Put` still costs ~20ns — only worth it for real churn.

### sync.Cond (rare, but asked)
```go
c := sync.NewCond(&sync.Mutex{})
// waiter
c.L.Lock()
for !condition() { c.Wait() }     // ALWAYS in a for loop — spurious/stale wakeups
c.L.Unlock()
// signaler
c.L.Lock(); setCondition(); c.L.Unlock()
c.Broadcast()                      // or c.Signal()
```
🎯 In Go you almost always prefer channels; `Cond` is for cases where a channel can't express
"broadcast to N waiters on a shared mutable state."

### sync/atomic
```go
var n int64
atomic.AddInt64(&n, 1)
atomic.LoadInt64(&n)
atomic.StoreInt64(&n, 5)
ok := atomic.CompareAndSwapInt64(&n, 5, 6)

// Go 1.19+ typed atomics — PREFER THESE (no alignment bugs, no & typos)
var c atomic.Int64
c.Add(1); c.Load(); c.CompareAndSwap(1, 2)

var v atomic.Value          // any, but type must be consistent
var p atomic.Pointer[Config]
p.Store(&Config{})
cfg := p.Load()
_ = cfg
```
⚠️ **64-bit alignment**: with the old `atomic.AddInt64(&x, …)` API, `x` must be 64-bit aligned —
on 32-bit platforms the *first word of an allocated struct* is guaranteed aligned, so put int64
fields first. The `atomic.Int64` type embeds an aligner and removes this footgun entirely.

🎯 "Atomics are for single-word operations. The moment you need two variables consistent with each
other, you need a mutex."

**Lock-free counter benchmark intuition:** `atomic.AddInt64` ≈ 5–10ns uncontended;
`Mutex` Lock/Unlock ≈ 15–25ns uncontended, but hundreds of ns under contention. Sharded counters
(one cache-line-padded counter per P) is the answer for extreme contention.

### False sharing (bonus points)
```go
type paddedCounter struct {
    n   atomic.Int64
    _   [56]byte      // pad to a 64-byte cache line
}
```

---

## 17. context

```go
type Context interface {
    Deadline() (deadline time.Time, ok bool)
    Done() <-chan struct{}
    Err() error
    Value(key any) any
}
```

### Constructors
```go
ctx := context.Background()                                  // root, in main/tests/init
ctx  = context.TODO()                                        // placeholder, "fix later"
ctx, cancel := context.WithCancel(parent)
ctx, cancel  = context.WithTimeout(parent, 5*time.Second)
ctx, cancel  = context.WithDeadline(parent, t)
ctx          = context.WithValue(parent, ctxKey("reqID"), id)
ctx, cancel  = context.WithCancelCause(parent)               // Go 1.20; cancel(err) + context.Cause(ctx)
ctx          = context.WithoutCancel(parent)                 // Go 1.21; keeps values, drops cancellation
ctx, cancel  = context.WithDeadlineCause(parent, t, err)     // Go 1.21
defer cancel()                                                // ALWAYS
```

### Rules
1. **First parameter**, always named `ctx`, type `context.Context`. Never store it in a struct
   (exception: request-scoped structs that are themselves per-request).
2. **Never pass nil** — use `context.TODO()`.
3. **Always `defer cancel()`** even when the context times out. Not calling it **leaks the timer
   and the parent's child list** until the deadline. `go vet`'s `lostcancel` catches it.
4. Cancellation propagates **down** the tree only. Cancelling a child doesn't affect the parent.
5. `ctx.Err()` returns `context.Canceled` or `context.DeadlineExceeded`.
6. **Values are for request-scoped metadata only** — request ID, trace span, auth subject.
   Not for optional parameters, not for dependencies.
7. **Keys must be an unexported custom type** to avoid collisions:
```go
type ctxKey struct{}                       // or: type ctxKey string
func WithUser(ctx context.Context, u *User) context.Context {
    return context.WithValue(ctx, ctxKey{}, u)
}
func UserFrom(ctx context.Context) (*User, bool) {
    u, ok := ctx.Value(ctxKey{}).(*User)
    return u, ok
}
```
⚠️ `ctx.Value` is a **linear walk up the parent chain** — O(depth). Don't put it in a hot loop.

### Respecting cancellation in your own code
```go
func work(ctx context.Context, items []Item) error {
    for _, it := range items {
        select {
        case <-ctx.Done():
            return ctx.Err()          // check between units of work
        default:
        }
        if err := process(ctx, it); err != nil { return err }
    }
    return nil
}
```
🎯 "Context doesn't kill goroutines — there's no such thing in Go. It's a cooperative signal;
your code has to check `Done()` or pass ctx down to something that does (`http`, `database/sql`,
`grpc` all do)."

---

## 18. Go memory model & happens-before

**The guarantee:** "A read of a variable is guaranteed to observe a write only if the write
happens-before the read; otherwise it's a data race and the behavior is undefined."

### Happens-before edges you should be able to list
1. Within a single goroutine, program order.
2. `go f()` — the `go` statement **happens-before** f's execution starts.
3. **A send on a channel happens-before the corresponding receive completes.**
4. **A close happens-before a receive that returns the zero value.**
5. For **unbuffered** channels: **a receive happens-before the send completes** (the extra
   direction — this is what makes unbuffered channels a two-way sync point).
6. For a buffered channel of capacity C: the *k*-th receive happens-before the *(k+C)*-th send completes.
7. `mu.Unlock()` happens-before a subsequent `mu.Lock()` returns.
8. `once.Do(f)` — f's return happens-before any `Do` call returns.
9. `wg.Done()` calls happen-before `wg.Wait()` returns.
10. Atomics (Go 1.19 formalized them as sequentially consistent): an atomic read observing an
    atomic write establishes the edge.

⚠️ **A goroutine's exit is NOT synchronized with anything.** `go f()` gives you no guarantee that
f ever runs before main exits. `main` returning kills everything.

```go
// classic race, no happens-before edge
var done bool
var msg string
go func() { msg = "hello"; done = true }()
for !done {}                 // may loop forever OR see msg == "" — UB
fmt.Println(msg)
```
Fix with a channel, a mutex, or `atomic.Bool`.

🎯 Interviewer's follow-up: "Is `int64` assignment atomic in Go?" → **No guarantee.** Go's memory
model gives no atomicity for ordinary variables of any size. Word-sized writes won't tear in practice
on mainstream hardware, but a race is still a race and the compiler may reorder or cache the value
in a register.

---

## 19. Race conditions, deadlocks, goroutine leaks

### Race detector
```bash
go test -race ./...
go run -race main.go
go build -race
```
- Uses ThreadSanitizer; ~5–10× slower, 5–10× more memory.
- **Only detects races that actually execute** — it's a dynamic detector, not a prover.
- Run it in CI on the full test suite, and consider a canary instance with `-race` in staging.

### The four deadlock conditions (they may ask)
Mutual exclusion, hold-and-wait, no preemption, circular wait. Break any one.

**Common Go deadlocks:**
```go
// 1. Unbuffered channel, no other goroutine
ch := make(chan int)
ch <- 1                     // fatal error: all goroutines are asleep - deadlock!

// 2. WaitGroup never reaching zero (missing Done, or Add inside goroutine)

// 3. Lock ordering inversion
// G1: mu1.Lock(); mu2.Lock()
// G2: mu2.Lock(); mu1.Lock()
// FIX: always acquire locks in a globally consistent order (e.g., by address or ID)

// 4. Reentrant lock
mu.Lock(); defer mu.Unlock()
helper()   // helper also does mu.Lock() -> deadlock

// 5. RLock then Lock in one goroutine
```
🎯 The runtime detects only the case where **all** goroutines are blocked. A partial deadlock
(2 goroutines stuck, others running) shows up as a hang, not a crash — you find it with
`SIGQUIT` (`kill -QUIT pid`) to dump all goroutine stacks, or `/debug/pprof/goroutine?debug=2`.
**Go 1.26 added an experimental goroutine leak profile** (`GOEXPERIMENT=goroutineleakprofile`)
that detects goroutines blocked on unreachable channels/mutexes.

### Goroutine leaks — the top 5 causes
1. **Sending to a channel nobody reads** (timeout fired, receiver gone). → buffer the channel, or `select` with `ctx.Done()`.
2. **Receiving from a channel nobody closes.** → producer must always `defer close`.
3. **`for range ch` where the producer errored out before closing.**
4. **Forgetting `cancel()`** on a `WithCancel` context → the goroutine watching `Done()` never wakes.
5. **`time.Tick`** (no way to stop it) — always use `time.NewTicker` + `defer Stop()`.

**Detecting them:**
```go
// in tests
import "go.uber.org/goleak"
func TestMain(m *testing.M) { goleak.VerifyTestMain(m) }
```
```go
// in prod: alert on this metric
runtime.NumGoroutine()
```
```bash
curl localhost:6060/debug/pprof/goroutine?debug=2   # full stacks, find the common blocked line
```

### How to answer "how do you debug a goroutine leak in production?"
1. Confirm with the `go_goroutines` metric trending up monotonically.
2. Grab `/debug/pprof/goroutine?debug=1` twice, minutes apart; diff the counts by stack.
3. The growing stack shows exactly the blocked line — usually `chan send` or `chan receive` or `sync.WaitGroup.Wait`.
4. Trace back to the missing `close`, missing `cancel`, or unbuffered result channel.

---

## 20. Memory: stack vs heap, escape analysis, GC

### Stack vs heap
Go decides **at compile time** via **escape analysis**. There is no `new` vs `make` distinction for
allocation location — `new(T)` can be stack-allocated.

```bash
go build -gcflags='-m -m' ./...     # shows escape decisions
go build -gcflags='-m' ./... 2>&1 | grep escapes
```

**Things that cause escape:**
- Returning a pointer to a local (usually — sometimes inlining saves it).
- Storing a pointer in an interface (`fmt.Println(x)` escapes x).
- Value captured by a closure that outlives the frame.
- Slice/map with a **non-constant** size, or too large for the stack (>~64KB → heap).
- Sending a pointer on a channel.
- Anything the compiler can't prove — escape analysis is conservative.

```go
func stackAlloc() int   { x := 42; return x }        // no escape
func heapAlloc() *int   { x := 42; return &x }       // x escapes to heap
func alsoHeap()         { x := 42; fmt.Println(x) }  // escapes: ...any boxing
```
🎯 "Go doesn't have a stack/heap distinction in the language — the compiler decides. If you want
to know, `-gcflags=-m`. And 'returning a pointer to a local' is safe in Go, unlike C."

### The garbage collector
- **Concurrent, tri-color, mark-and-sweep, non-generational, non-compacting.**
- **Write barrier**: Dijkstra-style deletion/insertion hybrid (yuasa + dijkstra) to maintain the
  tri-color invariant while mutators run.
- **Two stop-the-world pauses**, both **sub-millisecond** (typically 10s–100s of µs): one to enable
  the write barrier / start marking, one to terminate marking. Sweeping is lazy & concurrent.
- **Pacer** targets ~25% of CPU for GC and aims to start marking early enough that the heap hits
  the goal exactly when marking finishes.
- **Non-compacting** ⇒ no pointer rewriting (except stack copies), but fragmentation is possible;
  mitigated by size-class allocation (tcmalloc-derived: mcache per P → mcentral → mheap).
- **Go 1.26: the "Green Tea" GC is on by default** — 10–40% lower GC overhead via better locality
  for small objects; disable with `GOEXPERIMENT=nogreenteagc`.

### Tuning
```go
// GOGC: heap growth % before next GC. Default 100 = collect when heap doubles.
// GOGC=off disables GC. Higher = fewer GCs, more RAM.
debug.SetGCPercent(200)

// GOMEMLIMIT (Go 1.19): a SOFT memory limit. GC works harder as you approach it.
// The single best knob for containers — prevents OOMKill from a large spike.
debug.SetMemoryLimit(3 << 30)   // 3 GiB

runtime.GC()                     // force; basically only for tests/benchmarks
```
🎯 **The production answer:** "Set `GOMEMLIMIT` to about 80–90% of the container memory limit, and
often `GOGC=off`. Then GC is driven purely by the memory limit rather than by heap-growth ratio —
you use the memory you paid for and you don't get OOM-killed. That's the standard recipe since 1.19."

### Reducing allocations (they'll ask "how would you optimize this")
1. `make([]T, 0, n)` / `sb.Grow(n)` — **preallocate with known capacity**.
2. Reuse buffers via `sync.Pool`.
3. Pass small structs **by value**; pass large ones by pointer — but note a pointer often forces a heap escape.
4. Avoid `interface{}`/`any` in hot paths (boxing allocates).
5. Avoid `fmt.Sprintf` in hot paths → `strconv.Itoa`, `strings.Builder`, `append`-based formatting.
6. Use `[]byte` end-to-end instead of converting to `string` repeatedly.
7. Struct field ordering to cut padding.
8. **Measure first**: `go test -bench . -benchmem`, then `-memprofile`.

### Finalizers / weak pointers
`runtime.SetFinalizer` — avoid; unpredictable, prevents collection cycles. Go 1.24 added
`runtime.AddCleanup` (better) and `weak.Pointer[T]` for caches.
---

# PART 3 — BACKEND / PRODUCTION GO

## 21. net/http server & routing

### Minimal server
```go
func main() {
    mux := http.NewServeMux()
    mux.HandleFunc("GET /users/{id}", getUser)      // Go 1.22+ method + wildcards
    mux.HandleFunc("POST /users", createUser)
    mux.Handle("/static/", http.StripPrefix("/static/", http.FileServer(http.Dir("assets"))))

    srv := &http.Server{
        Addr:              ":8080",
        Handler:           mux,
        ReadHeaderTimeout: 5 * time.Second,
        ReadTimeout:       10 * time.Second,
        WriteTimeout:      30 * time.Second,
        IdleTimeout:       120 * time.Second,
        MaxHeaderBytes:    1 << 20,
    }
    log.Fatal(srv.ListenAndServe())
}

func getUser(w http.ResponseWriter, r *http.Request) {
    id := r.PathValue("id")                          // Go 1.22+
    u, err := store.Get(r.Context(), id)             // ALWAYS propagate r.Context()
    if err != nil {
        if errors.Is(err, ErrNotFound) {
            http.Error(w, "not found", http.StatusNotFound)
            return
        }
        http.Error(w, "internal", http.StatusInternalServerError)
        return
    }
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusOK)                     // MUST come after Header().Set
    _ = json.NewEncoder(w).Encode(u)
}
```

**Go 1.22 routing patterns** (big deal — you may not need chi/gorilla anymore):
```
"GET /items/{id}"        method + wildcard, id via r.PathValue("id")
"/files/{path...}"       trailing wildcard matches the rest, including slashes
"/exact/{$}"             {$} anchors: matches ONLY "/exact/", not "/exact/sub"
"example.com/path"       host-specific patterns
```
Precedence: **the most specific pattern wins** (not registration order); genuinely ambiguous
patterns **panic at registration**.

### The handler interface
```go
type Handler interface { ServeHTTP(http.ResponseWriter, *http.Request) }
type HandlerFunc func(http.ResponseWriter, *http.Request)   // adapter
func (f HandlerFunc) ServeHTTP(w http.ResponseWriter, r *http.Request) { f(w, r) }
```
🎯 `http.HandlerFunc` is the canonical example of **an adapter type that gives a func a method** —
good answer to "show me an interesting use of methods on non-struct types."

**Each request runs in its own goroutine.** Anything a handler touches concurrently must be safe.

### ⚠️ Handler gotchas
- `w.WriteHeader` can only be called **once**; the first `w.Write` implicitly calls `WriteHeader(200)`.
  Setting headers after writing is silently ignored.
- **Always `defer r.Body.Close()`** for client responses; for server requests the server closes it,
  but you should still drain it if you want connection reuse.
- Limit request bodies: `r.Body = http.MaxBytesReader(w, r.Body, 1<<20)`.
- Returning early without a response leaves the client with a 200 and empty body.
- `http.Error` writes `Content-Type: text/plain` — for a JSON API, write your own error helper.

### A handler that returns an error (nicer style)
```go
type apiFunc func(http.ResponseWriter, *http.Request) error

func (f apiFunc) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    if err := f(w, r); err != nil {
        var ae *APIError
        if errors.As(err, &ae) { writeJSON(w, ae.Status, ae); return }
        slog.ErrorContext(r.Context(), "handler failed", "err", err)
        writeJSON(w, 500, map[string]string{"error": "internal"})
    }
}
```

### Dependency injection into handlers — closures over a struct
```go
type Server struct {
    store  Store
    logger *slog.Logger
}
func (s *Server) handleGetUser() http.HandlerFunc {
    // per-handler setup runs once, at wiring time
    validate := buildValidator()
    return func(w http.ResponseWriter, r *http.Request) {
        _ = validate
        _ = s.store
    }
}
```
🎯 This "handler returns a HandlerFunc" pattern (Mat Ryer's) is a well-known idiom — mentioning it
signals you've read real Go server code.

---

## 22. Middleware

```go
type Middleware func(http.Handler) http.Handler

func Chain(h http.Handler, ms ...Middleware) http.Handler {
    for i := len(ms) - 1; i >= 0; i-- { h = ms[i](h) }  // reverse so ms[0] is outermost
    return h
}

func Logging(logger *slog.Logger) Middleware {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            start := time.Now()
            rw := &statusWriter{ResponseWriter: w, status: http.StatusOK}
            next.ServeHTTP(rw, r)
            logger.InfoContext(r.Context(), "request",
                "method", r.Method, "path", r.URL.Path,
                "status", rw.status, "dur", time.Since(start))
        })
    }
}

// You need a wrapper to capture the status code:
type statusWriter struct {
    http.ResponseWriter
    status int
    n      int64
}
func (w *statusWriter) WriteHeader(code int) { w.status = code; w.ResponseWriter.WriteHeader(code) }
func (w *statusWriter) Write(b []byte) (int, error) {
    n, err := w.ResponseWriter.Write(b); w.n += int64(n); return n, err
}
```
⚠️ **Wrapping `ResponseWriter` breaks optional interfaces** (`http.Flusher`, `http.Hijacker`,
`io.ReaderFrom`, `http.Pusher`). Either implement passthroughs or use
`net/http/httptest`-style helpers / `github.com/felixge/httpsnoop`.

### Recovery + Request ID + Timeout middleware
```go
func Recover(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if rec := recover(); rec != nil {
                slog.Error("panic", "err", rec, "stack", string(debug.Stack()))
                http.Error(w, "internal server error", http.StatusInternalServerError)
            }
        }()
        next.ServeHTTP(w, r)
    })
}

func RequestID(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        id := r.Header.Get("X-Request-ID")
        if id == "" { id = uuid.NewString() }
        ctx := context.WithValue(r.Context(), reqIDKey{}, id)
        w.Header().Set("X-Request-ID", id)
        next.ServeHTTP(w, r.WithContext(ctx))     // r is immutable — WithContext returns a copy
    })
}

// stdlib timeout middleware
handler = http.TimeoutHandler(handler, 10*time.Second, "request timeout")
```
⚠️ `recover()` in middleware does **not** catch a panic in a goroutine the handler spawned.
⚠️ `http.ErrAbortHandler` should be re-panicked, not swallowed, in recovery middleware.

---

## 23. Timeouts, graceful shutdown, http.Client

### Server timeouts — what each one covers
| Field | Covers |
|---|---|
| `ReadHeaderTimeout` | reading request headers (Slowloris defense) |
| `ReadTimeout` | headers + body |
| `WriteTimeout` | from end of header read to end of response write |
| `IdleTimeout` | keep-alive idle between requests |
| `http.TimeoutHandler` | per-handler wall clock, returns 503 |
| `context.WithTimeout` in the handler | the actual business-logic budget |

🎯 "`WriteTimeout` doesn't understand streaming, so for SSE/long-poll endpoints you set it to 0 on
that server or use a separate one, and rely on context deadlines instead."

### Graceful shutdown (near-guaranteed interview question)
```go
func main() {
    srv := &http.Server{Addr: ":8080", Handler: h}

    go func() {
        if err := srv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
            log.Fatalf("listen: %v", err)
        }
    }()

    ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
    defer stop()
    <-ctx.Done()
    log.Println("shutting down...")

    shutdownCtx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    if err := srv.Shutdown(shutdownCtx); err != nil {
        log.Printf("forced shutdown: %v", err)
        _ = srv.Close()                       // hard close
    }
    // then: close DB pool, flush traces, drain workers
    log.Println("bye")
}
```
**Points that score:**
- `Shutdown` stops listeners, closes idle keep-alives, waits for in-flight handlers. It does **not**
  wait for hijacked (WebSocket) connections — you must track those yourself via `RegisterOnShutdown`.
- `ListenAndServe` returns `http.ErrServerClosed` immediately on Shutdown — don't treat it as fatal.
- In Kubernetes, **fail the readiness probe first, sleep ~5s, then Shutdown** — otherwise the load
  balancer keeps sending traffic to a socket you already closed.
- Order matters: stop accepting → drain HTTP → stop consumers → close DB → flush telemetry.

### http.Client — the biggest production footguns
```go
// ⚠️ NEVER use http.DefaultClient in production: NO TIMEOUT. A hung server hangs you forever.
var client = &http.Client{
    Timeout: 10 * time.Second,     // total: dial + TLS + request + response body read
    Transport: &http.Transport{
        Proxy:                 http.ProxyFromEnvironment,
        MaxIdleConns:          100,
        MaxIdleConnsPerHost:   100,   // DEFAULT IS 2 — the classic cause of connection churn
        MaxConnsPerHost:       0,
        IdleConnTimeout:       90 * time.Second,
        TLSHandshakeTimeout:   5 * time.Second,
        ExpectContinueTimeout: 1 * time.Second,
        ResponseHeaderTimeout: 5 * time.Second,
        DialContext: (&net.Dialer{Timeout: 3 * time.Second, KeepAlive: 30 * time.Second}).DialContext,
    },
}

func fetch(ctx context.Context, url string) ([]byte, error) {
    req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
    if err != nil { return nil, err }
    resp, err := client.Do(req)
    if err != nil { return nil, fmt.Errorf("get %s: %w", url, err) }
    defer func() {
        io.Copy(io.Discard, resp.Body)   // DRAIN so the connection can be reused
        resp.Body.Close()
    }()
    if resp.StatusCode != http.StatusOK {
        return nil, fmt.Errorf("unexpected status %d", resp.StatusCode)
    }
    return io.ReadAll(io.LimitReader(resp.Body, 10<<20))
}
```
**Recite these three:**
1. **Reuse one `http.Client`** — it's safe for concurrent use and owns the connection pool.
   Creating one per request destroys keep-alive and leaks FDs/TIME_WAIT sockets.
2. **`MaxIdleConnsPerHost` defaults to 2.** For a service calling one backend hard, raise it.
3. **Always close AND drain the body**, even on non-200, or the connection won't be reused.

Retries: exponential backoff **with jitter**, only on idempotent methods / 5xx / network errors,
capped, and respect `Retry-After`. Mention **circuit breakers** (`sony/gobreaker`) for the
"how do you stop a cascading failure" follow-up.

---

## 24. JSON

```go
type User struct {
    ID        int       `json:"id"`
    Name      string    `json:"name"`
    Email     string    `json:"email,omitempty"`   // omit if zero value
    Password  string    `json:"-"`                  // never marshal
    CreatedAt time.Time `json:"created_at"`
    Meta      any       `json:"meta,omitempty"`
    Count     int       `json:"count,string"`       // encode number as JSON string
}
```

```go
b, err := json.Marshal(u)                 // []byte
err = json.Unmarshal(b, &u)               // needs a POINTER
enc := json.NewEncoder(w); enc.Encode(u)  // streaming; appends '\n'
dec := json.NewDecoder(r.Body)
dec.DisallowUnknownFields()               // strict: reject unexpected keys
err = dec.Decode(&u)
```

### ⚠️ JSON gotchas (very commonly asked)
1. **Only exported fields are marshaled.** Unexported fields are silently skipped.
2. **`omitempty` uses the *zero value*, not "unset"** — `false`, `0`, and `""` disappear. To
   distinguish "absent" from "false", use a **pointer** (`*bool`) or `json.RawMessage`.
   (Go 1.24 added `omitzero`, which is the cleaner "truly zero" semantic, and works with `time.Time`.)
3. **`nil` slice → `null`, empty slice → `[]`.** Initialize response slices to avoid breaking clients.
4. **Numbers decode into `any` as `float64`** — a large int64 loses precision past 2^53.
   Fix: `dec.UseNumber()` then `json.Number.Int64()`.
5. **Unmarshal into a struct ignores unknown fields** by default and leaves missing fields at their
   current value (it does **not** zero the struct first).
6. Field matching is **case-insensitive** on unmarshal when there's no exact tag match.
7. `time.Time` marshals as RFC 3339. `time.Duration` marshals as an **integer nanoseconds** —
   surprising; wrap it if you want "5s".
8. **`map[string]any` output has sorted keys**; struct output follows field order.
9. Marshaling a **cyclic structure** causes a stack overflow (fatal).
10. `[]byte` marshals as **base64**.

### Custom marshaling
```go
type Duration time.Duration
func (d Duration) MarshalJSON() ([]byte, error) {
    return json.Marshal(time.Duration(d).String())
}
func (d *Duration) UnmarshalJSON(b []byte) error {
    var s string
    if err := json.Unmarshal(b, &s); err != nil { return err }
    v, err := time.ParseDuration(s)
    if err != nil { return err }
    *d = Duration(v)
    return nil
}
```
⚠️ `MarshalJSON` on a **value receiver** works for both `T` and `*T`; on a **pointer receiver** it's
skipped when you marshal a non-addressable `T`. Use value receivers for `MarshalJSON`, pointer for
`UnmarshalJSON`.

Performance: `encoding/json` uses reflection and is comparatively slow. Alternatives worth naming:
`github.com/goccy/go-json`, `bytedance/sonic`, `easyjson` (codegen). Go 1.25 shipped
**`encoding/json/v2`** behind `GOEXPERIMENT=jsonv2` — much faster and fixes several of the gotchas above.

---

## 25. database/sql

```go
db, err := sql.Open("pgx", dsn)   // does NOT connect — lazy
if err != nil { return err }
defer db.Close()
if err := db.PingContext(ctx); err != nil { return err }   // this actually connects

db.SetMaxOpenConns(25)                    // hard cap; 0 = unlimited (dangerous)
db.SetMaxIdleConns(25)                    // keep == MaxOpenConns to avoid churn
db.SetConnMaxLifetime(5 * time.Minute)    // rotate; plays well with LBs & failover
db.SetConnMaxIdleTime(1 * time.Minute)
```
🎯 **`*sql.DB` is a pool, not a connection.** It's safe for concurrent use — create **one** and pass
it around; never open one per request.

### Query patterns
```go
// single row
var u User
err := db.QueryRowContext(ctx, `SELECT id, name FROM users WHERE id=$1`, id).
    Scan(&u.ID, &u.Name)
if errors.Is(err, sql.ErrNoRows) { return nil, ErrNotFound }

// many rows
rows, err := db.QueryContext(ctx, `SELECT id, name FROM users WHERE age > $1`, age)
if err != nil { return nil, err }
defer rows.Close()                       // ⚠️ MANDATORY — else the conn leaks
var users []User
for rows.Next() {
    var u User
    if err := rows.Scan(&u.ID, &u.Name); err != nil { return nil, err }
    users = append(users, u)
}
if err := rows.Err(); err != nil { return nil, err }   // ⚠️ CHECK THIS — catches mid-iteration errors

// exec
res, err := db.ExecContext(ctx, `UPDATE users SET name=$1 WHERE id=$2`, name, id)
n, _ := res.RowsAffected()
```

### ⚠️ The three classic database/sql bugs
1. **Not calling `rows.Close()`** → the connection is never returned to the pool → pool exhaustion →
   every request hangs. (`defer rows.Close()` is idempotent and also called implicitly when
   `rows.Next()` returns false, but an early `return` inside the loop skips that.)
2. **Not checking `rows.Err()`** → you silently truncate a result set on a network hiccup.
3. **Starting a transaction and not always committing/rolling back** → connection held forever.

### Transactions
```go
func transfer(ctx context.Context, db *sql.DB, from, to int, amt int64) (err error) {
    tx, err := db.BeginTx(ctx, &sql.TxOptions{Isolation: sql.LevelSerializable})
    if err != nil { return err }
    defer func() {
        if p := recover(); p != nil { _ = tx.Rollback(); panic(p) }
        if err != nil { _ = tx.Rollback(); return }
        err = tx.Commit()
    }()

    if _, err = tx.ExecContext(ctx, `UPDATE acct SET bal=bal-$1 WHERE id=$2`, amt, from); err != nil {
        return fmt.Errorf("debit: %w", err)
    }
    if _, err = tx.ExecContext(ctx, `UPDATE acct SET bal=bal+$1 WHERE id=$2`, amt, to); err != nil {
        return fmt.Errorf("credit: %w", err)
    }
    return nil
}
```
⚠️ A `*sql.Tx` **pins one connection** for its entire life. Never do slow work (HTTP calls) inside
a transaction, and never run tx queries concurrently from multiple goroutines.

### SQL injection
```go
// ALWAYS parameterize
db.QueryContext(ctx, "SELECT * FROM users WHERE name = $1", name)  // safe
// NEVER
db.QueryContext(ctx, "SELECT * FROM users WHERE name = '"+name+"'") // injectable
```
For dynamic identifiers (table/column names) you can't parameterize — use an **allow-list**.
For `IN (...)`, build the placeholders, or use `pq.Array` / `sqlx.In`.

### NULL handling
```go
var email sql.NullString
row.Scan(&email)
if email.Valid { use(email.String) }
// Go 1.22+: sql.Null[T] generic;  or use *string
```

### Ecosystem, and what to say
- `database/sql` + driver (`pgx/v5` stdlib mode, `go-sql-driver/mysql`).
- **`pgx` native mode** (not through database/sql) — faster, real Postgres types.
- **`sqlx`** — thin helpers: `Get`, `Select`, `StructScan`, named params.
- **`sqlc`** — generates type-safe Go from your SQL. Very popular answer; "SQL stays SQL."
- **`ent` / `gorm`** — ORMs. GORM is common but be ready to discuss N+1 queries and magic.
- **Migrations**: `golang-migrate`, `goose`, `atlas`.

---

## 26. Project layout & dependency injection

### Layout that won't get you challenged
```
myapp/
  cmd/
    api/main.go            # wiring only: flags, config, construct deps, run
    worker/main.go
  internal/                # unimportable from outside the module — enforced by the compiler
    user/                  # domain package: service + interfaces + errors
      service.go
      store.go             # the Store interface DEFINED HERE (consumer-side)
    postgres/
      user_store.go        # the implementation
    http/
      server.go handlers.go middleware.go
    config/config.go
  pkg/                     # only if you truly intend external reuse (often an anti-pattern)
  migrations/
  go.mod
```
🎯 Talking points:
- **`internal/` is real enforcement**, not a convention. Default everything there.
- **Package by feature/domain, not by layer.** `internal/user` beats `internal/models` + `internal/handlers`.
- **Avoid a `utils` or `common` package** — it becomes a dependency magnet and causes import cycles.
- **Interfaces live with the consumer**, implementations elsewhere. That's how you avoid cycles and
  keep interfaces small.
- `golang-standards/project-layout` is *not* official; say "it's a community convention, and the
  Go team has explicitly not endorsed it."

### Dependency injection — just use constructors
```go
type Service struct {
    store  Store
    cache  Cache
    logger *slog.Logger
    clock  func() time.Time        // inject time for testability
}
func NewService(s Store, c Cache, l *slog.Logger) *Service {
    return &Service{store: s, cache: c, logger: l, clock: time.Now}
}
```
Frameworks: `google/wire` (compile-time codegen, no reflection — the safe answer),
`uber-go/fx` and `dig` (runtime reflection). For most services **manual wiring in `main` is
correct and preferred**; say that.

### Configuration
```go
type Config struct {
    Port     int           `env:"PORT" envDefault:"8080"`
    DSN      string        `env:"DATABASE_URL,required"`
    Timeout  time.Duration `env:"TIMEOUT" envDefault:"5s"`
}
```
12-factor: env vars > files > flags for containers. Validate at startup and **fail fast**.
`kelseyhightower/envconfig`, `caarlos0/env`, `spf13/viper` (heavy), or hand-rolled `os.Getenv`.

### Go modules quick reference
```bash
go mod init github.com/me/app
go mod tidy          # add missing + drop unused; run before every commit
go mod vendor        # vendor/ dir
go mod why -m pkg    # why is this dependency here
go get -u ./...      # upgrade
go list -m all       # full module graph
GOFLAGS=-mod=readonly
```
- **Semantic Import Versioning**: v2+ must have `/v2` in the module path.
- **MVS (Minimal Version Selection)**: Go picks the *lowest* version satisfying all requirements —
  deliberately reproducible, unlike npm/cargo's "latest compatible."
- `go.sum` = cryptographic hashes; `GONOSUMDB`/`GOPRIVATE` for private repos; `GOPROXY` and the
  checksum database (`sum.golang.org`).
- `replace` directives for local dev; ⚠️ they're **ignored in dependent modules**.

---

## 27. Testing

### Table-driven tests — the Go idiom
```go
func TestSum(t *testing.T) {
    tests := []struct {
        name    string
        input   []int
        want    int
        wantErr bool
    }{
        {"empty", nil, 0, false},
        {"single", []int{5}, 5, false},
        {"multiple", []int{1, 2, 3}, 6, false},
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {      // subtest: isolated, named, filterable
            t.Parallel()                          // subtests run concurrently
            got, err := Sum(tt.input)
            if (err != nil) != tt.wantErr {
                t.Fatalf("Sum() error = %v, wantErr %v", err, tt.wantErr)
            }
            if got != tt.want {
                t.Errorf("Sum(%v) = %d, want %d", tt.input, got, tt.want)
            }
        })
    }
}
```
- `t.Error` continues; `t.Fatal` stops **that goroutine's** test. ⚠️ Never call `t.Fatal` from a
  spawned goroutine — it calls `runtime.Goexit` on the wrong goroutine. Use `t.Error` + return.
- `t.Helper()` in assertion helpers so failures report the caller's line.
- `t.Cleanup(fn)` — LIFO teardown, better than `defer` in a helper.
- `t.TempDir()`, `t.Setenv()` (implies non-parallel).
- ⚠️ Pre-Go 1.22 `t.Parallel()` + loop variable required `tt := tt`.

### Commands
```bash
go test ./...
go test -v -run 'TestSum/empty' ./pkg
go test -race ./...
go test -cover -coverprofile=c.out ./... && go tool cover -html=c.out
go test -count=1 ./...          # disable the test result cache
go test -short ./...            # skip long tests via testing.Short()
go test -bench=. -benchmem -benchtime=5s
go test -fuzz=FuzzParse -fuzztime=60s
go test -timeout 30s
go vet ./...
```

### Mocking = interfaces
```go
// production code depends on this small interface
type Store interface { GetUser(ctx context.Context, id int) (*User, error) }

// hand-rolled fake — usually better than a generated mock
type fakeStore struct {
    users map[int]*User
    err   error
    calls int
}
func (f *fakeStore) GetUser(_ context.Context, id int) (*User, error) {
    f.calls++
    if f.err != nil { return nil, f.err }
    u, ok := f.users[id]
    if !ok { return nil, ErrNotFound }
    return u, nil
}
```
🎯 "I prefer hand-written fakes for small interfaces and `mockery`/`gomock` when the interface is
large or I need strict call-order assertions. The real design lever is keeping the interface small
enough that a fake is 10 lines."

### httptest — both directions
```go
// testing a handler
func TestGetUser(t *testing.T) {
    req := httptest.NewRequest(http.MethodGet, "/users/1", nil)
    req.SetPathValue("id", "1")
    rec := httptest.NewRecorder()

    h := NewServer(&fakeStore{users: map[int]*User{1: {ID: 1, Name: "A"}}})
    h.ServeHTTP(rec, req)

    if rec.Code != http.StatusOK { t.Fatalf("got %d", rec.Code) }
    var got User
    if err := json.Unmarshal(rec.Body.Bytes(), &got); err != nil { t.Fatal(err) }
    if got.Name != "A" { t.Errorf("got %q", got.Name) }
}

// testing a client (fake upstream server)
func TestFetch(t *testing.T) {
    srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(200)
        fmt.Fprint(w, `{"ok":true}`)
    }))
    defer srv.Close()
    body, err := fetch(context.Background(), srv.URL)
    _, _ = body, err
}
```

### Benchmarks
```go
func BenchmarkConcat(b *testing.B) {
    words := make([]string, 100)
    for i := range words { words[i] = "x" }
    b.ReportAllocs()
    b.ResetTimer()                   // exclude setup
    for i := 0; i < b.N; i++ {
        var sb strings.Builder
        for _, w := range words { sb.WriteString(w) }
        _ = sb.String()
    }
}
// Go 1.24+: for b.Loop() { ... }  — prevents the compiler optimizing the body away
```
⚠️ Assign results to a **package-level sink** variable (or use `b.Loop`) so dead-code elimination
doesn't delete your benchmark. Compare with `benchstat` over `-count=10`.

### Fuzzing (Go 1.18+)
```go
func FuzzParse(f *testing.F) {
    f.Add("key=value")                        // seed corpus
    f.Fuzz(func(t *testing.T, s string) {
        k, v, ok := strings.Cut(s, "=")
        if ok && k+"="+v != s { t.Fatalf("round-trip failed for %q", s) }
    })
}
```

### testing/synctest (Go 1.25, GA) — worth a mention
Runs a bubble of goroutines with a **fake clock**, so tests of timeouts/retries are instant and
deterministic instead of `time.Sleep`-flaky.
```go
synctest.Test(t, func(t *testing.T) {
    // time.Sleep, time.After, context deadlines advance virtually
})
```

### Other things to name
- `t.Setenv`, `testing.Short()`, `TestMain(m *testing.M)` for suite setup/teardown.
- **Golden files**: `-update` flag rewriting `testdata/*.golden`.
- **`testcontainers-go`** for real Postgres/Redis in integration tests. Great answer to
  "how do you test DB code?"
- `//go:build integration` tags to separate slow tests.
- `testify/require` vs `assert`: `require` stops on failure. Fine to use; be ready to say the
  stdlib-only approach also works.
- Coverage as a signal, not a target. Test behavior, not implementation.

---

## 28. Profiling & pprof

### Enabling
```go
import _ "net/http/pprof"                     // registers on DefaultServeMux

go func() { log.Println(http.ListenAndServe("localhost:6060", nil)) }()
```
⚠️ Never expose `/debug/pprof` publicly — it leaks stacks and lets anyone stall your process.
Bind it to localhost or an admin port with auth.

### Profile types
| Profile | Answers |
|---|---|
| `profile` (CPU, 30s) | where is CPU time going |
| `heap` | what's using memory **now** (inuse) and total allocations (alloc) |
| `goroutine` | how many goroutines, blocked where — **leak hunting** |
| `allocs` | all past allocations — GC pressure |
| `block` | time blocked on sync primitives (needs `SetBlockProfileRate`) |
| `mutex` | lock contention (needs `SetMutexProfileFraction`) |
| `threadcreate` | OS threads |
| trace | scheduler, GC, syscalls timeline |

```bash
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30
go tool pprof -http=:8080 http://localhost:6060/debug/pprof/heap
curl -o trace.out 'localhost:6060/debug/pprof/trace?seconds=5' && go tool trace trace.out

# in pprof:  top -cum   list funcName   web   peek   traces
```
```go
runtime.SetBlockProfileRate(1)        // 1 = every event; costly, sample in prod
runtime.SetMutexProfileFraction(5)
```

### Reading a heap profile
- `inuse_space` — live memory. Use for "why is RSS high."
- `alloc_space` — cumulative. Use for "why is GC burning CPU."
🎯 "High GC CPU with low inuse_space means high allocation *rate*, not a leak. That points to
per-request allocations — sync.Pool, preallocation, or avoiding interface boxing."

### Continuous profiling
Pyroscope / Grafana Cloud Profiles / Datadog. Mentioning it shows production maturity.

### Compiler diagnostics worth knowing
```bash
go build -gcflags='-m'           # escape analysis
go build -gcflags='-m -m'        # verbose
go build -gcflags='-S'           # assembly
go build -gcflags='-d=ssa/check_bce/debug=1'   # bounds-check elimination
GODEBUG=gctrace=1 ./app          # per-GC line: heap sizes, pause times
GODEBUG=schedtrace=1000 ./app    # scheduler state every second
GODEBUG=inittrace=1 ./app        # init cost per package
```

---

## 29. Observability

### Structured logging with `log/slog` (Go 1.21 stdlib)
```go
logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
    Level:     slog.LevelInfo,
    AddSource: true,
}))
slog.SetDefault(logger)

logger.Info("user created", "user_id", id, "duration", time.Since(start))
logger.LogAttrs(ctx, slog.LevelInfo, "typed & alloc-free",
    slog.Int("user_id", id), slog.Duration("dur", d))

reqLogger := logger.With("request_id", reqID)      // child logger with fixed attrs
logger.ErrorContext(ctx, "db failed", "err", err)  // handlers can pull trace IDs from ctx
```
Rules: **structured, leveled, to stdout, one event per line, no PII/secrets.** Don't log and
return the same error. Sample high-volume logs. Alternatives: `zap` (fastest), `zerolog`.

### Metrics — the four golden signals
Latency, Traffic, Errors, Saturation. (Or USE: Utilization, Saturation, Errors.)
```go
var reqDuration = prometheus.NewHistogramVec(
    prometheus.HistogramOpts{
        Name:    "http_request_duration_seconds",
        Buckets: prometheus.DefBuckets,
    },
    []string{"method", "route", "status"},   // ⚠️ NEVER label with user_id/path — cardinality explosion
)
http.Handle("/metrics", promhttp.Handler())
```
Go runtime metrics you get for free and should alert on: `go_goroutines`, `go_memstats_heap_inuse_bytes`,
`go_gc_duration_seconds`, `go_threads`. Prefer the `runtime/metrics` package over the older
`runtime.ReadMemStats` (which stops the world).

### Tracing
OpenTelemetry: `otelhttp` middleware, propagate `traceparent`, span per outbound call,
attach `trace_id` to logs so you can pivot between the three signals.

---

## 30. Production war stories they ask about

**"A service's memory grows until OOM. Walk me through it."**
1. Is it a leak or a rate problem? Check `go_memstats_heap_inuse_bytes` vs `heap_alloc_rate`.
2. Grab two heap profiles minutes apart, `pprof -base old new` to diff.
3. Check `go_goroutines` — a goroutine leak looks like a memory leak (each holds a 2KB+ stack plus captures).
4. Usual culprits: unbounded in-memory cache/map with no eviction, sub-slicing a large buffer,
   goroutines blocked on a channel, `sync.Pool` misuse, unclosed response bodies, string interning
   of unbounded input, a slice that only ever grows.
5. Mitigate with `GOMEMLIMIT`, then fix.

**"Latency p99 spiked but p50 is fine."**
GC pauses (check `gctrace`), lock contention (mutex profile), a slow downstream with no timeout,
connection pool exhaustion (`MaxOpenConns`, `MaxIdleConnsPerHost`), noisy-neighbor CPU throttling
(cgroup `cpu.stat` `nr_throttled`), or head-of-line blocking on a single-threaded consumer.

**"CPU is at 100% but throughput is flat."**
Contention: goroutines spinning on a mutex, `GOMAXPROCS` mismatched to the cgroup limit, a busy
loop without a yield, excessive GC (`GOGC` too low / allocation churn), or regex/JSON in a hot path.

**"How do you handle a poison-pill message in a consumer?"**
Bounded retries with backoff → dead-letter queue → alert. Never block the partition forever.
Make handlers **idempotent**; use an idempotency key table.

**"How do you deploy safely?"**
Health + readiness probes, graceful shutdown, `preStop` sleep, rolling/canary, feature flags,
backward-compatible DB migrations (expand → migrate → contract), and a rollback plan.

**Common security answers**
- Parameterized SQL; `html/template` (auto-escaping) not `text/template` for HTML.
- `crypto/rand` not `math/rand` for tokens; `bcrypt`/`argon2id` for passwords.
- `subtle.ConstantTimeCompare` for secrets.
- `http.MaxBytesReader`, timeouts everywhere, `govulncheck ./...` in CI.
- Don't log secrets; redact via a custom `LogValuer` on the type.
---

# PART 4 — DSA IN GO + QUESTION BANK

## 31. Go idioms for coding rounds

Interviewers judge **idiomatic Go**, not just correctness. These are the moves.

### Stack / queue / deque with slices
```go
// STACK
stack := []int{}
stack = append(stack, 1)                   // push
top := stack[len(stack)-1]                 // peek
stack = stack[:len(stack)-1]               // pop

// QUEUE (fine for interviews; amortized O(1) but the head memory isn't reclaimed until regrow)
queue := []int{1, 2, 3}
front := queue[0]
queue = queue[1:]                          // dequeue

// QUEUE with index (avoids the reslice-leak concern)
q, head := []int{1, 2, 3}, 0
front = q[head]; head++

_ = top; _ = front
```
`container/list` exists but is rarely worth it — slice-backed is faster and simpler.

### Set
```go
set := map[string]struct{}{}
set["a"] = struct{}{}
if _, ok := set["a"]; ok { /* present */ }
delete(set, "a")
// map[string]bool is also fine and reads better: if set["a"] {}
```

### Sorting
```go
slices.Sort(nums)                                   // Go 1.21+, generic, pdqsort
slices.SortFunc(people, func(a, b Person) int {     // return <0, 0, >0
    return cmp.Compare(a.Age, b.Age)
})
slices.SortStableFunc(people, func(a, b Person) int { return strings.Compare(a.Name, b.Name) })
sort.Slice(nums, func(i, j int) bool { return nums[i] < nums[j] })   // older API, less-func

i, found := slices.BinarySearch(nums, 7)
idx := sort.SearchInts(nums, 7)
_, _, _ = i, found, idx
```
Note `sort.Slice` is **not stable** and uses reflection; `slices.SortFunc` is faster and type-safe.
`sort.Sort` needs `Len/Less/Swap` — know the interface, but reach for `slices`.

### Heap (`container/heap`)
```go
type MinHeap []int
func (h MinHeap) Len() int            { return len(h) }
func (h MinHeap) Less(i, j int) bool  { return h[i] < h[j] }   // > for a max-heap
func (h MinHeap) Swap(i, j int)       { h[i], h[j] = h[j], h[i] }
func (h *MinHeap) Push(x any)         { *h = append(*h, x.(int)) }
func (h *MinHeap) Pop() any {
    old := *h
    n := len(old)
    v := old[n-1]
    *h = old[:n-1]
    return v
}

h := &MinHeap{5, 2, 8}
heap.Init(h)
heap.Push(h, 1)
smallest := heap.Pop(h).(int)          // 1
_ = smallest
```
⚠️ The classic trap: `heap.Push`/`heap.Pop` (package funcs, maintain the invariant) vs
`h.Push`/`h.Pop` (your methods, raw slice ops). **Never call the methods directly.**
Also: `Push`/`Pop` need **pointer receivers** because they change the length.

### math / conversions
```go
math.MaxInt, math.MinInt, math.MaxInt64, math.Inf(1)
min(a, b), max(a, b)                        // Go 1.21 BUILTINS, variadic, work on any ordered type
abs := func(x int) int { if x < 0 { return -x }; return x }   // no int abs in stdlib!
n, err := strconv.Atoi("42")
s := strconv.Itoa(42)
f, _ := strconv.ParseFloat("3.14", 64)
b, _ := strconv.ParseBool("true")
s = strconv.FormatInt(255, 16)              // "ff"
_, _, _, _ = n, err, f, b
```

### Multi-dimensional slices
```go
grid := make([][]int, rows)
for i := range grid { grid[i] = make([]int, cols) }

// single-allocation version (better locality, common optimization to mention)
flat := make([]int, rows*cols)
grid2 := make([][]int, rows)
for i := range grid2 { grid2[i] = flat[i*cols : (i+1)*cols : (i+1)*cols] }
```

### Struct as a composite map key
```go
type point struct{ x, y int }
visited := map[point]bool{}
visited[point{1, 2}] = true      // structs of comparable fields are valid keys
```

### Value swap and multiple return
```go
a, b = b, a
q, r := div(7, 2)
_, _ = q, r
```

### Bit tricks
```go
x & (x - 1)          // clear lowest set bit
x & -x               // isolate lowest set bit
bits.OnesCount(uint(x))     // popcount — math/bits
bits.LeadingZeros64(uint64(x))
bits.TrailingZeros(uint(x))
bits.Len(uint(x))    // position of highest set bit + 1
```

---

## 32. Classic problems, idiomatic Go

### Two Sum — hash map, O(n)
```go
func twoSum(nums []int, target int) []int {
    seen := make(map[int]int, len(nums))   // value -> index
    for i, n := range nums {
        if j, ok := seen[target-n]; ok { return []int{j, i} }
        seen[n] = i
    }
    return nil
}
```

### Sliding window — longest substring without repeating chars
```go
func lengthOfLongestSubstring(s string) int {
    last := make(map[byte]int)      // byte -> last index+1
    best, start := 0, 0
    for i := 0; i < len(s); i++ {
        if j, ok := last[s[i]]; ok && j > start { start = j }
        if l := i - start + 1; l > best { best = l }
        last[s[i]] = i + 1
    }
    return best
}
```

### Two pointers — valid palindrome (rune-safe)
```go
func isPalindrome(s string) bool {
    r := []rune(strings.ToLower(s))
    i, j := 0, len(r)-1
    for i < j {
        for i < j && !unicode.IsLetter(r[i]) && !unicode.IsDigit(r[i]) { i++ }
        for i < j && !unicode.IsLetter(r[j]) && !unicode.IsDigit(r[j]) { j-- }
        if r[i] != r[j] { return false }
        i, j = i+1, j-1
    }
    return true
}
```

### Binary search (write it by hand once)
```go
func search(nums []int, target int) int {
    lo, hi := 0, len(nums)-1
    for lo <= hi {
        mid := lo + (hi-lo)/2       // avoids overflow
        switch {
        case nums[mid] == target: return mid
        case nums[mid] < target:  lo = mid + 1
        default:                  hi = mid - 1
        }
    }
    return -1
}
```

### Linked list — reverse
```go
type ListNode struct {
    Val  int
    Next *ListNode
}
func reverseList(head *ListNode) *ListNode {
    var prev *ListNode
    for head != nil {
        head.Next, prev, head = prev, head, head.Next
        // careful: Go evaluates RHS first, so this one-liner is correct
    }
    return prev
}
```

### Cycle detection — Floyd
```go
func hasCycle(head *ListNode) bool {
    slow, fast := head, head
    for fast != nil && fast.Next != nil {
        slow, fast = slow.Next, fast.Next.Next
        if slow == fast { return true }
    }
    return false
}
```

### Binary tree — DFS + BFS
```go
type TreeNode struct {
    Val         int
    Left, Right *TreeNode
}

func inorder(root *TreeNode, out *[]int) {
    if root == nil { return }
    inorder(root.Left, out)
    *out = append(*out, root.Val)
    inorder(root.Right, out)
}

func levelOrder(root *TreeNode) [][]int {
    if root == nil { return nil }
    var res [][]int
    q := []*TreeNode{root}
    for len(q) > 0 {
        n := len(q)
        level := make([]int, 0, n)
        for i := 0; i < n; i++ {
            node := q[0]; q = q[1:]
            level = append(level, node.Val)
            if node.Left != nil  { q = append(q, node.Left) }
            if node.Right != nil { q = append(q, node.Right) }
        }
        res = append(res, level)
    }
    return res
}
```

### Graph — BFS shortest path on a grid
```go
func shortestPath(grid [][]int, start, end [2]int) int {
    rows, cols := len(grid), len(grid[0])
    dirs := [4][2]int{{0,1},{0,-1},{1,0},{-1,0}}
    visited := make([][]bool, rows)
    for i := range visited { visited[i] = make([]bool, cols) }

    type node struct{ r, c, d int }
    q := []node{{start[0], start[1], 0}}
    visited[start[0]][start[1]] = true

    for len(q) > 0 {
        cur := q[0]; q = q[1:]
        if cur.r == end[0] && cur.c == end[1] { return cur.d }
        for _, d := range dirs {
            nr, nc := cur.r+d[0], cur.c+d[1]
            if nr < 0 || nr >= rows || nc < 0 || nc >= cols { continue }
            if visited[nr][nc] || grid[nr][nc] == 1 { continue }
            visited[nr][nc] = true
            q = append(q, node{nr, nc, cur.d + 1})
        }
    }
    return -1
}
```

### LRU Cache — the most common "design a data structure" question
```go
type entry struct{ key, value int }

type LRUCache struct {
    cap   int
    ll    *list.List                       // container/list, front = most recent
    items map[int]*list.Element
}

func NewLRU(capacity int) *LRUCache {
    return &LRUCache{cap: capacity, ll: list.New(), items: make(map[int]*list.Element, capacity)}
}

func (c *LRUCache) Get(key int) (int, bool) {
    el, ok := c.items[key]
    if !ok { return 0, false }
    c.ll.MoveToFront(el)
    return el.Value.(*entry).value, true
}

func (c *LRUCache) Put(key, value int) {
    if el, ok := c.items[key]; ok {
        c.ll.MoveToFront(el)
        el.Value.(*entry).value = value
        return
    }
    el := c.ll.PushFront(&entry{key, value})
    c.items[key] = el
    if c.ll.Len() > c.cap {
        last := c.ll.Back()
        c.ll.Remove(last)
        delete(c.items, last.Value.(*entry).key)
    }
}
```
🎯 Follow-up they always ask: **"make it thread-safe."** Answer: embed a `sync.Mutex` (not RWMutex —
`Get` mutates the list, so reads aren't read-only). For higher throughput, **shard by key hash**
into N independent LRUs, each with its own lock. Mention `hashicorp/golang-lru` and `ristretto`.

### Top-K frequent elements — heap
```go
func topKFrequent(nums []int, k int) []int {
    freq := map[int]int{}
    for _, n := range nums { freq[n]++ }
    keys := make([]int, 0, len(freq))
    for n := range freq { keys = append(keys, n) }
    slices.SortFunc(keys, func(a, b int) int { return cmp.Compare(freq[b], freq[a]) })
    return keys[:min(k, len(keys))]
    // O(n log n). The heap version is O(n log k); bucket sort is O(n) — mention both.
}
```

### Merge intervals
```go
func merge(intervals [][]int) [][]int {
    if len(intervals) == 0 { return nil }
    slices.SortFunc(intervals, func(a, b []int) int { return cmp.Compare(a[0], b[0]) })
    out := [][]int{intervals[0]}
    for _, iv := range intervals[1:] {
        last := out[len(out)-1]
        if iv[0] <= last[1] {
            last[1] = max(last[1], iv[1])
        } else {
            out = append(out, iv)
        }
    }
    return out
}
```

### DP — coin change
```go
func coinChange(coins []int, amount int) int {
    dp := make([]int, amount+1)
    for i := 1; i <= amount; i++ {
        dp[i] = amount + 1
        for _, c := range coins {
            if c <= i { dp[i] = min(dp[i], dp[i-c]+1) }
        }
    }
    if dp[amount] > amount { return -1 }
    return dp[amount]
}
```

### Concurrency-flavored coding questions (Go-specific — expect at least one)

**Alternate printing between two goroutines**
```go
func alternate(n int) {
    odd, even := make(chan struct{}), make(chan struct{})
    var wg sync.WaitGroup
    wg.Add(2)
    go func() { defer wg.Done()
        for i := 1; i <= n; i += 2 {
            <-odd
            fmt.Println("odd", i)
            if i+1 <= n { even <- struct{}{} }    // guard: don't signal a goroutine that already exited
        }
    }()
    go func() { defer wg.Done()
        for i := 2; i <= n; i += 2 {
            <-even
            fmt.Println("even", i)
            if i+1 <= n { odd <- struct{}{} }
        }
    }()
    odd <- struct{}{}     // kick off
    wg.Wait()
}
```
⚠️ Both `if i+1 <= n` guards are load-bearing. Drop either one and the last handoff sends to a
goroutine that has already returned → `fatal error: all goroutines are asleep - deadlock!`
Interviewers run this; the guard is the whole test.

**Concurrent-safe counter, three ways**
```go
// 1) mutex
type C1 struct{ mu sync.Mutex; n int }
func (c *C1) Inc() { c.mu.Lock(); c.n++; c.mu.Unlock() }

// 2) atomic — fastest for a single counter
type C2 struct{ n atomic.Int64 }
func (c *C2) Inc() { c.n.Add(1) }

// 3) channel-owned state (the "share memory by communicating" answer)
func counterService(inc <-chan int, get <-chan chan int, done <-chan struct{}) {
    n := 0
    for {
        select {
        case d := <-inc:      n += d
        case reply := <-get:  reply <- n
        case <-done:          return
        }
    }
}
```

**Fetch N URLs concurrently, preserve order, bound to 5, fail fast** → the `errgroup` snippet in §15.3.
That single answer covers ~4 possible questions; have it memorized.

**Implement a timeout for a function** → §15.8.

**Bounded parallel map** → §15.2 semaphore.

---

## 33. Rapid-fire Q&A

### Language
1. **`new` vs `make`?** `new(T)` allocates zeroed memory and returns `*T`, for any type.
   `make` initializes **slices, maps, channels only** and returns `T` (not a pointer) because those
   types need internal setup.
2. **`var s []int` vs `s := []int{}`?** nil vs non-nil empty. Same behavior except `== nil` and
   JSON (`null` vs `[]`).
3. **Can you compare two structs?** Yes if all fields are comparable. Not if any field is a slice,
   map, or func.
4. **Is Go OOP?** It has encapsulation and composition, no inheritance, and polymorphism via
   interfaces. Composition over inheritance, enforced by the language.
5. **Why no ternary?** Readability by design; use `if`/`else` or a small helper.
6. **Value vs pointer receiver?** See §5.
7. **What is a rune?** An alias for `int32`, one Unicode code point.
8. **Can a method be defined on any type?** Any **named type declared in the same package**.
   Not on `int` directly, but on `type MyInt int`. Not on pointer types or interface types.
9. **What does `_` do?** Blank identifier: discard a value, force interface checks
   (`var _ I = (*T)(nil)`), or import for side effects.
10. **Struct tags?** Metadata strings read via reflection: `json`, `db`, `validate`, `yaml`.
11. **Shadowing?** `:=` in an inner scope creates a new variable. Classic bug with `err`.
    `go vet -vettool=shadow` finds it.
12. **Labeled break/continue?**
    ```go
    outer:
    for i := range m {
        for j := range n { if cond { break outer } }
    }
    ```
    Also `goto` exists (rarely used, can't jump over declarations).
13. **`switch` without a condition?** `switch { case x > 5: ... }` — a cleaner if/else chain.
    Go switches **don't fall through** by default; use `fallthrough` explicitly. `case` supports
    comma lists.
14. **Variadic?** `func f(xs ...int)`; `f(s...)` to expand. Inside, `xs` is a slice — and it
    **aliases** the caller's slice when expanded with `...`, so mutating it is visible.
15. **Function values / first-class funcs?** Yes. Funcs are values, can be fields, map values,
    and closures.
16. **Multiple return values, named returns?** Named returns document intent and enable defer to
    modify them — but naked `return` in a long function hurts readability.

### Concurrency
17. **Goroutine vs thread?** §12 table.
18. **Buffered vs unbuffered channel?** §13.
19. **What happens on send to a closed channel?** Panic.
20. **How do you know a channel is closed?** `v, ok := <-ch`; `ok == false`.
21. **Can you close a channel from the receiver?** You *can*, but don't — it breaks the ownership
    rule and races with senders (they'd panic).
22. **How do you stop a goroutine?** You can't kill it. Signal it: `context`, a `done` channel, or
    closing its input channel. The goroutine must cooperate.
23. **Mutex vs channel — when?** Mutex for protecting shared state (a cache, a counter).
    Channel for transferring ownership / coordinating / pipelines. "Don't communicate by sharing
    memory; share memory by communicating" — but the Go team also says use a mutex when a mutex
    is simpler.
24. **What is a data race?** Two goroutines access the same memory concurrently, at least one
    writes, with no synchronization.
25. **Race condition vs data race?** A data race is a memory-model violation. A race condition is
    a logic bug from timing — you can have one without the other (e.g. check-then-act under
    separate lock acquisitions).
26. **`sync.Map` vs `map` + mutex?** §3.
27. **What's `GOMAXPROCS`?** §12.
28. **Does Go have thread-local storage?** No, deliberately. That's why `context` is passed
    explicitly.
29. **Are channel operations atomic?** Yes, the runtime locks the channel internally; and they
    establish happens-before edges.
30. **What's the zero value of a channel and what does it do?** nil; blocks forever on send/recv,
    panics on close. Useful in `select` to disable a case.
31. **Can `select` have duplicate channels?** Yes; the uniform-random choice still applies.
32. **How does `time.After` leak?** §14.
33. **How many goroutines can you run?** Limited by memory: ~2KB each, so millions. The real
    limits are downstream (DB conns, file descriptors, memory).
34. **Worker pool vs unbounded goroutines?** Bound concurrency to protect downstream resources and
    memory; unbounded goroutine spawning per request is a classic outage.

### Runtime / performance
35. **Stack or heap?** Escape analysis decides. §20.
36. **Describe Go's GC.** §20 — concurrent tri-color mark-sweep, non-generational, non-compacting,
    sub-ms STW, write barriers, GOGC/GOMEMLIMIT pacing, Green Tea in 1.26.
37. **How do you reduce GC pressure?** Preallocate, pool, avoid boxing, fewer pointers per object
    (the GC scans pointers, so `[]int` is cheaper to scan than `[]*int`).
38. **What's `GOMEMLIMIT`?** Soft memory limit added in 1.19; the standard container fix.
39. **How would you find a memory leak?** §30.
40. **Is Go compiled or interpreted?** Compiled to a **static native binary**, no VM, no runtime
    dependency (unless cgo). Fast compile times were an explicit design goal.
41. **What is cgo and why avoid it?** C interop. Costs: slow builds, loses cross-compilation
    simplicity, ~a few hundred ns per call (~30% cheaper in Go 1.26), blocks an M during the call,
    breaks the race detector's view, complicates static linking.
42. **How do you cross-compile?** `GOOS=linux GOARCH=arm64 go build`. Trivial without cgo;
    `CGO_ENABLED=0` for a fully static binary.
43. **What's inlining?** The compiler copies small function bodies inline. Budget-based
    (cost ≤ 80 nodes); functions with `defer`/`recover`/loops (until 1.22-ish work) or that are too
    big won't inline. `-gcflags='-m'` shows decisions.
44. **PGO?** Profile-Guided Optimization (Go 1.21+): drop a `default.pgo` next to `main` and the
    compiler uses the profile to inline better — typically 2–7% faster.
45. **Build tags?** `//go:build linux && amd64` — conditional compilation; also `_test.go`,
    `_linux.go`, `_amd64.go` filename suffixes.
46. **`go build` flags for size?** `-ldflags="-s -w"` strips debug info/DWARF. Inject version:
    `-ldflags="-X main.version=$(git rev-parse HEAD)"`.

### Backend
47. **Is `http.Handler` safe for concurrent use?** Your handler is called from many goroutines
    concurrently, so any shared state it touches must be safe.
48. **How do you version an API?** URL path (`/v1/`), header, or media type. Path is simplest.
49. **Idempotency?** Idempotency-Key header + a dedup table, so retries don't double-charge.
50. **gRPC vs REST?** gRPC: HTTP/2, protobuf, streaming, codegen, strong contracts — good
    service-to-service. REST/JSON: browser/debuggability. Mention `grpc-gateway` for both.
51. **How do you handle config secrets?** Env vars injected from a secret manager (Vault, AWS
    Secrets Manager, K8s secrets); never in the repo; rotate.
52. **Caching strategy?** Cache-aside is the default; TTL + jitter; singleflight to prevent
    stampede; explicit invalidation on write; consider negative caching.
53. **How do you test code that uses time?** Inject a clock (`func() time.Time` or a `Clock`
    interface), or use `testing/synctest` (Go 1.25).
54. **How do you test code that uses randomness?** Inject a `*rand.Rand` with a fixed seed.
55. **Structured logging library?** `log/slog` in stdlib since 1.21; `zap`/`zerolog` if you need
    the last bit of performance.
56. **Where do you put your interfaces?** In the consuming package. §26.
57. **How do you avoid import cycles?** Consumer-side interfaces, extract shared types into a
    leaf package, or invert the dependency.

### Trick questions they love
58. **What does this print?**
    ```go
    func main() {
        defer fmt.Println("1")
        defer fmt.Println("2")
        panic("boom")
    }
    ```
    → `2`, `1`, then the panic trace. Defers run during panic unwinding, LIFO.
59. **What does this print?**
    ```go
    func modify(s []int) { s = append(s, 4); s[0] = 100 }

    s := []int{1, 2, 3}
    modify(s)
    fmt.Println(s)
    ```
    → **`[1 2 3]`**. Trace `len`/`cap` out loud: `len==cap==3`, so `append` must allocate a **new**
    backing array and reassign the local `s`. `s[0] = 100` therefore writes into the copy, which the
    caller never sees.
    🎯 The killer follow-up: change it to `s := make([]int, 3, 4)`. Now `append` writes in place, the
    local `s` still points at the caller's array, and the output becomes **`[100 2 3]`**. Same code,
    different answer — that's the whole lesson about slice aliasing.
60. **Is `map[string]int` write safe if each goroutine uses a different key?** **No.** Different
    keys can land in the same bucket, and the map can rehash. Fatal error.
61. **Does `defer` run if the program calls `os.Exit`?** **No.** Also not on `log.Fatal` (which
    calls `os.Exit(1)`). Defers run on panic, not on exit.
62. **What's the output of `fmt.Println(len("日本語"))`?** `9` — 3 runes × 3 bytes.
63. **What's `interface{}` vs `any`?** Identical; `any` is an alias added in Go 1.18.
64. **Can you have a method on a pointer to a pointer?** No — the receiver base type cannot itself
    be a pointer type.
65. **Two nil errors compare equal?** Two `errors.New("x")` values are **never** equal even with
    the same text (pointer comparison). That's why sentinels are package-level `var`s.
66. **What does `var x any = nil; x == nil` give?** `true`. But `var p *int; var x any = p;
    x == nil` gives `false`. §6.
67. **Can a deferred function see the panic value?** Only via `recover()`.
68. **Does `for range` over a channel end?** Only when the channel is **closed**.

---

## 34. 15-minute cram sheet

**Say these sentences and you'll sound like a Go engineer:**

- "A slice is a 3-word header — pointer, len, cap. Append shares the backing array if cap allows,
  otherwise it copies. That's why you always reassign the result."
- "A nil map is fine to read, panics on write. A nil slice appends fine."
- "Method set: `*T` has both value and pointer methods; `T` has only value methods. It only matters
  for interface satisfaction."
- "An interface is nil only when both its type word and value word are nil — so a typed nil pointer
  in an error interface is not nil."
- "Accept interfaces, return structs. Define the interface where you consume it."
- "Wrap errors with `%w`, match with `errors.Is`, extract with `errors.As`. Handle once."
- "defer is LIFO, arguments evaluate immediately, and it can modify named returns."
- "Go 1.22 made loop variables per-iteration."
- "GMP: goroutines onto Ps onto OS threads, with work stealing, a netpoller so I/O doesn't block a
  thread, and async preemption since 1.14."
- "Send on a closed channel panics, receive returns the zero value with ok=false, close of a closed
  or nil channel panics."
- "select picks uniformly at random among ready cases; nil channels are never ready, which is how
  you disable a case."
- "The goroutine that creates a channel owns it and is the only one that closes it."
- "Context is a cooperative cancellation signal, always the first parameter, always `defer cancel()`."
- "A goroutine leak looks like a memory leak — check `/debug/pprof/goroutine` and diff the counts."
- "Set `GOMEMLIMIT` to ~85% of the container limit; that's the standard containerized Go recipe."
- "Reuse one `http.Client`, set every timeout, and raise `MaxIdleConnsPerHost` — the default is 2."
- "Graceful shutdown: fail readiness, sleep, `srv.Shutdown(ctx)`, then close the DB pool."
- "`*sql.DB` is a pool. Always `defer rows.Close()` and always check `rows.Err()`."
- "Table-driven tests with subtests, hand-written fakes over small interfaces, `-race` in CI."
- "Profile before optimizing: `pprof` CPU and heap, `alloc_space` for GC pressure vs `inuse_space`
  for leaks."

**Commands to have at your fingertips**
```bash
go test -race -cover ./...
go test -bench=. -benchmem
go vet ./... && staticcheck ./... && govulncheck ./...
go build -gcflags='-m' ./...
go tool pprof -http=:8080 http://localhost:6060/debug/pprof/heap
GODEBUG=gctrace=1 ./app
go mod tidy && go mod why -m <pkg>
```

**Three questions to ask THEM (shows seniority)**
- "How do you handle graceful shutdown and in-flight requests during deploys?"
- "Where has Go's concurrency model bitten you in this codebase?"
- "Do you use `sqlc`/`pgx` directly or an ORM, and how has that aged?"

---

## Practice plan before the mock

| Day | Focus |
|---|---|
| 1 | Part 1 §2–3 (slices/maps) + §5–6 (method sets, nil interface). Re-derive the traps from scratch. |
| 2 | Part 2 §12–14. Write the channel behavior table from memory. |
| 3 | Part 2 §15. Type out the worker pool, errgroup, and semaphore patterns without looking. |
| 4 | Part 2 §16–20. Explain GC + escape analysis out loud in 90 seconds. |
| 5 | Part 3. Write a full server with middleware, timeouts, graceful shutdown, and one test. |
| 6 | Part 4 §31–32. Solve 5 problems in a plain editor, no autocomplete. |
| 7 | §33 rapid-fire out loud + §34 cram sheet. |

---

*Notes prepared for a mid-level Go backend interview loop. Verified against Go 1.26 (Feb 2026);
code samples compile under Go 1.24+ except where a newer version is explicitly noted.*
