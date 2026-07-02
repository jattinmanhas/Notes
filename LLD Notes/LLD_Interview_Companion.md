# LLD Interview Companion — SOLID + From a Blank Page

> **Why this doc exists:** `LLD_Design_Patterns.md` is a reference for *how to implement* a pattern once you know which one fits. This doc covers what that one doesn't: *why* a design is good (SOLID), and *how you get from a vague prompt to a pattern* in the first place — which is the actual skill tested in an LLD interview. Read this alongside the pattern reference, not instead of it.

---

## Table of Contents

1. [SOLID Principles](#1-solid-principles)
2. [Composition Over Inheritance](#2-composition-over-inheritance)
3. [From a Blank Page: How to Approach Any LLD Problem](#3-from-a-blank-page-how-to-approach-any-lld-problem)
4. [Worked Problem: Parking Lot](#4-worked-problem-parking-lot)
5. [Worked Problem: Elevator System](#5-worked-problem-elevator-system)
6. [Pattern Combinations You'll Actually See](#6-pattern-combinations-youll-actually-see)
7. [Interview Practice Questions](#7-interview-practice-questions)

---

## 1. SOLID Principles

### Why this matters more than the patterns themselves

Interviewers rarely ask "implement the Strategy pattern" cold. They ask you to design something, and *while* you design it, they push on **why** you split classes the way you did. The answer they're listening for almost always traces back to one of these five principles — not "because it's a known pattern." Knowing SOLID is what lets you justify a design instead of just naming it.

### S — Single Responsibility Principle

**A class should have exactly one reason to change.**

If a class handles both "calculate an order's total" and "print an order receipt," those are two unrelated reasons to change (pricing logic vs. formatting logic) living in one place. A change to receipt formatting shouldn't risk breaking pricing.

```java
// Violates SRP — two reasons to change live in one class
class Order {
    double calculateTotal() { /* pricing logic */ }
    void printReceipt() { /* formatting/printing logic */ }
}

// Fixed — each class has one reason to change
class Order {
    double calculateTotal() { /* pricing logic */ }
}
class ReceiptPrinter {
    void print(Order order) { /* formatting/printing logic */ }
}
```

**Why it matters in interviews:** this is the principle behind almost every "why did you pull that out into its own class?" follow-up.

### O — Open/Closed Principle

**Open for extension, closed for modification.** You should be able to add new behavior without editing existing, tested code.

This is *exactly* what Strategy, Factory Method, and Decorator exist to give you. The Factory Method registry-map pattern in the main doc is a direct application: adding a new `Notification` type is a one-line registry addition, not an edit to an `if/else` chain inside the factory itself.

```java
// Violates OCP — every new shape needs an edit here
double area(Shape s) {
    if (s instanceof Circle c) return Math.PI * c.radius * c.radius;
    if (s instanceof Square sq) return sq.side * sq.side;
    // adding Triangle means editing this method
}

// Follows OCP — new shapes just implement the interface, this code never changes
interface Shape { double area(); }
class Circle implements Shape { public double area() { return Math.PI * radius * radius; } }
class Square implements Shape { public double area() { return side * side; } }
```

### L — Liskov Substitution Principle

**A subclass must be usable anywhere its parent is expected, without breaking correctness.**

The classic violation: `Square extends Rectangle`. Mathematically a square is a rectangle, but if `Rectangle` has independent `setWidth()`/`setHeight()`, a `Square` overriding both to keep sides equal breaks any code that assumes setting width doesn't change height.

```java
// Violates LSP
class Rectangle {
    void setWidth(int w)  { this.width = w; }
    void setHeight(int h) { this.height = h; }
}
class Square extends Rectangle {
    void setWidth(int w)  { this.width = w; this.height = w; } // surprises callers
    void setHeight(int h) { this.width = h; this.height = h; }
}
```

**Why it matters:** this is the principle behind "does inheritance actually make sense here, or should this be composition?" — see the Bridge and Strategy patterns, which exist partly to sidestep LSP violations by favoring composition.

### I — Interface Segregation Principle

**Don't force a class to implement methods it doesn't need.** Prefer several small, specific interfaces over one large one.

```java
// Violates ISP — a Robot has no body, forced to implement eat/sleep
interface Worker {
    void work();
    void eat();
    void sleep();
}

// Fixed — segregated interfaces
interface Workable { void work(); }
interface Restable  { void eat(); void sleep(); }

class Human implements Workable, Restable { /* implements all */ }
class Robot implements Workable { /* only work() */ }
```

**Why it matters:** directly relevant to the Composite pattern's pitfall already noted in the main doc — putting `add()`/`remove()` in the base `FileSystemItem` interface forces `File` (a leaf) to implement child-management methods that make no sense for it. That pitfall *is* an ISP violation.

### D — Dependency Inversion Principle

**Depend on abstractions, not concrete implementations.** High-level modules shouldn't directly instantiate low-level ones — inject an interface instead.

This is the principle behind nearly every pattern in the main doc that takes a constructor parameter of an interface type: Adapter, Strategy's `Sorter`, Abstract Factory's `Application`, Bridge's `Shape`. None of them do `new ConcreteThing()` internally — they receive an abstraction from outside.

```java
// Violates DIP — Sorter is hard-wired to one algorithm
class Sorter {
    private BubbleSort algo = new BubbleSort(); // concrete dependency
}

// Follows DIP — Sorter depends on an abstraction, injected from outside
class Sorter {
    private final SortStrategy algo;
    Sorter(SortStrategy algo) { this.algo = algo; }
}
```

**Why it matters:** this is *the* reason nearly every LLD pattern's usage examples pass dependencies into constructors instead of `new`-ing them up inside the class. If you find yourself explaining a design choice, "so the class doesn't depend on a concrete implementation" is very often the correct answer.

---

## 2. Composition Over Inheritance

### The principle

**Prefer "has-a" relationships (composition) over "is-a" relationships (inheritance) when behavior needs to vary or combine flexibly.** Inheritance locks in behavior at compile time through a rigid hierarchy; composition lets you assemble behavior at runtime by holding references to other objects.

### Why so many patterns in the main doc exist specifically to avoid inheritance

| Pattern | What it replaces | Why composition wins here |
|---|---|---|
| **Strategy** | Subclassing a base class per algorithm (`BubbleSortSorter`, `MergeSortSorter`, ...) | The algorithm can be swapped at *runtime*, not fixed at compile time by which subclass you picked |
| **Decorator** | Subclassing for every combination of add-on behavior (`LoggingCachingFetcher`, `CachingFetcher`, `LoggingFetcher`, ...) | Avoids combinatorial explosion — decorators stack in any order, at runtime |
| **Bridge** | A cross-product hierarchy (`VectorCircle`, `RasterCircle`, `VectorSquare`, `RasterSquare`, ...) | Two dimensions of variation multiply *class count* under inheritance but stay additive under composition |

**The tell in an interview:** if you catch yourself designing a class hierarchy where the number of subclasses would multiply every time a *new, independent* variation is added (a new shape **and** a new renderer; a new algorithm **and** a new logging option), that's the signal to switch to composition — hold a reference to the varying behavior instead of inheriting it.

**When inheritance is still fine:** genuine "is-a" relationships with behavior that doesn't need to vary independently — Template Method deliberately uses inheritance because the algorithm skeleton truly is fixed and shared; only specific steps vary, and that's exactly the case inheritance is good at.

---

## 3. From a Blank Page: How to Approach Any LLD Problem

This is the part the pattern reference can't teach you, because it starts *before* you know which pattern applies. A repeatable process:

### Step 1 — Clarify scope (2-3 minutes, out loud)

Real interview prompts ("design a parking lot," "design an elevator system") are intentionally underspecified. Ask:
- What's actually in scope? (e.g., for a parking lot: multiple floors? multiple vehicle types? payment? Do we need reservations?)
- What's explicitly out of scope? (Say it out loud — "I'll assume no reservation system unless you want that included" — so the interviewer can redirect you early instead of watching you build the wrong thing for 20 minutes.)

### Step 2 — Identify the core nouns (entities)

List the real-world "things" in the problem. For a parking lot: `ParkingLot`, `Floor`, `Spot`, `Vehicle`, `Ticket`, `Payment`. These become your first-pass classes — don't worry about patterns yet.

### Step 3 — Identify the verbs (behaviors) and where they naturally live

For each entity, what does it *do*? A `Vehicle` doesn't calculate parking fees — that's not its responsibility (SRP). A `ParkingLot` doesn't know how to process a credit card — that's a separate concern. Assign behavior to the entity that owns the relevant data, and split out anything that's really "a policy that varies" into its own thing — that's usually where a pattern shows up.

### Step 4 — Look for these specific signals, and match to a pattern

| If you notice... | Reach for... |
|---|---|
| "This needs exactly one shared instance across the system" | Singleton (but justify it — see main doc's pitfall about testability) |
| "The way we calculate/process X could vary" (fee calculation, sorting, payment processing) | Strategy |
| "We're instantiating different subtypes based on some input" (vehicle type, notification channel) | Factory Method |
| "An object's allowed actions change based on its own state" (order status, elevator moving/idle) | State |
| "Many parts of the system need to know when something happens" (spot becomes free, order confirmed) | Observer |
| "We want optional add-on behavior without a subclass explosion" (extra fees, premium features) | Decorator |
| "A request should pass through multiple checks before being handled" (validation, auth, request routing) | Chain of Responsibility |

### Step 5 — Design incrementally, narrate as you go

Don't silently write code for 10 minutes. Say what you're doing and why ("I'm making `PricingStrategy` an interface because the interviewer mentioned different rates for different vehicle types — this way adding a new rate model doesn't touch existing code, per Open/Closed"). This is where SOLID vocabulary earns its keep — it's the language you use to justify decisions live.

### Step 6 — Handle the follow-up twist

Every LLD interview has a "now what if..." twist (add concurrency, add a new requirement, scale to multiple locations). This is deliberately testing whether your design was *actually* extensible or just happened to work for the original scope. If your first design used Strategy/Factory/Observer appropriately, the twist is usually a small addition, not a rewrite — that's the whole point of designing this way.

---

## 4. Worked Problem: Parking Lot

### Prompt (as an interviewer might give it)

"Design a parking lot system. It should support multiple vehicle types, multiple floors, and calculate a fee when a vehicle leaves."

### Applying the process

**Entities:** `ParkingLot`, `Floor`, `ParkingSpot`, `Vehicle` (with subtypes), `Ticket`.

**Signal → Pattern mapping:**
- "Multiple vehicle types" needing different spot sizes → **Factory Method** to create the right `Vehicle` subtype, and spot-matching logic based on vehicle size
- "Calculate a fee" that could plausibly vary (hourly vs. flat, vehicle-type-based) → **Strategy** for `FeeStrategy`
- "One parking lot, shared entry/exit point tracking available spots" → **Singleton** for `ParkingLot` (justify it: exactly one instance should coordinate spot allocation across the whole system)
- Spots becoming free should potentially notify a "next available spot" display → **Observer**

```java
// Strategy — fee calculation varies
interface FeeStrategy {
    double calculateFee(long durationMinutes);
}
class HourlyFeeStrategy implements FeeStrategy {
    public double calculateFee(long durationMinutes) {
        return Math.ceil(durationMinutes / 60.0) * 20.0; // ₹20/hour, rounded up
    }
}

// Factory Method — vehicle-to-spot-size matching
enum VehicleType { MOTORCYCLE, CAR, TRUCK }

interface Vehicle { VehicleType getType(); }
class Car implements Vehicle { public VehicleType getType() { return VehicleType.CAR; } }

// Singleton — one lot coordinates all spot state
class ParkingLot {
    private static volatile ParkingLot instance;
    private final List<Floor> floors = new ArrayList<>();
    private final FeeStrategy feeStrategy;

    private ParkingLot(FeeStrategy feeStrategy) { this.feeStrategy = feeStrategy; }

    public static ParkingLot getInstance(FeeStrategy feeStrategy) {
        if (instance == null) {
            synchronized (ParkingLot.class) {
                if (instance == null) instance = new ParkingLot(feeStrategy);
            }
        }
        return instance;
    }

    public Ticket parkVehicle(Vehicle vehicle) {
        ParkingSpot spot = findAvailableSpot(vehicle.getType());
        spot.occupy(vehicle);
        return new Ticket(vehicle, spot, System.currentTimeMillis());
    }

    public double processExit(Ticket ticket) {
        long durationMinutes = (System.currentTimeMillis() - ticket.entryTime()) / 60000;
        ticket.spot().vacate();
        return feeStrategy.calculateFee(durationMinutes);
    }

    private ParkingSpot findAvailableSpot(VehicleType type) {
        return floors.stream()
            .flatMap(f -> f.getAvailableSpots(type).stream())
            .findFirst()
            .orElseThrow(() -> new IllegalStateException("Lot full"));
    }
}
```

**The twist an interviewer will likely add:** "now support reservations" or "now support multiple pricing tiers for members vs. non-members." Both slot cleanly into the existing `FeeStrategy` (swap or extend it) without touching `ParkingLot`'s core logic — proof the design was actually Open/Closed.

---

## 5. Worked Problem: Elevator System

### Prompt

"Design an elevator system for a building with multiple elevators and multiple floors."

### Applying the process

**Entities:** `Elevator`, `ElevatorController` (or `Dispatcher`), `Floor`, `Request` (internal button press vs. external hall call).

**Signal → Pattern mapping:**
- "An elevator's behavior changes based on whether it's idle, moving up, moving down, or under maintenance" → **State**
- "The dispatcher needs to decide which elevator responds to a hall call, and that logic could vary" (nearest-elevator vs. least-busy vs. zone-based) → **Strategy**
- "Multiple elevators, one central dispatcher coordinating them" → **Singleton** for the dispatcher (same justification pattern as the parking lot)
- "Each elevator needs to notify the dispatcher when it reaches a floor or becomes idle" → **Observer**

```java
// State — elevator behavior depends on its current state
interface ElevatorState {
    void handleRequest(Elevator elevator, int floor);
}

class IdleState implements ElevatorState {
    public void handleRequest(Elevator elevator, int floor) {
        if (floor > elevator.getCurrentFloor()) elevator.setState(new MovingUpState());
        else if (floor < elevator.getCurrentFloor()) elevator.setState(new MovingDownState());
        // if equal, doors just open — no state change needed
    }
}

class MovingUpState implements ElevatorState {
    public void handleRequest(Elevator elevator, int floor) {
        // queue the floor; continue moving up until no more requests above
    }
}
// MovingDownState, MaintenanceState follow the same shape

class Elevator {
    private int currentFloor = 0;
    private ElevatorState state = new IdleState();

    public void setState(ElevatorState state) { this.state = state; }
    public int getCurrentFloor() { return currentFloor; }
    public void requestFloor(int floor) { state.handleRequest(this, floor); }
}

// Strategy — dispatch logic varies
interface DispatchStrategy {
    Elevator selectElevator(List<Elevator> elevators, int requestFloor);
}
class NearestElevatorStrategy implements DispatchStrategy {
    public Elevator selectElevator(List<Elevator> elevators, int requestFloor) {
        return elevators.stream()
            .min(Comparator.comparingInt(e -> Math.abs(e.getCurrentFloor() - requestFloor)))
            .orElseThrow();
    }
}

// Singleton — one controller coordinates all elevators
class ElevatorController {
    private static volatile ElevatorController instance;
    private final List<Elevator> elevators = new ArrayList<>();
    private final DispatchStrategy dispatchStrategy;

    private ElevatorController(DispatchStrategy strategy) { this.dispatchStrategy = strategy; }

    public static ElevatorController getInstance(DispatchStrategy strategy) {
        if (instance == null) {
            synchronized (ElevatorController.class) {
                if (instance == null) instance = new ElevatorController(strategy);
            }
        }
        return instance;
    }

    public void handleHallCall(int floor) {
        Elevator chosen = dispatchStrategy.selectElevator(elevators, floor);
        chosen.requestFloor(floor);
    }
}
```

**The twist an interviewer will likely add:** "now handle an elevator going into maintenance mid-route" or "now optimize dispatch for peak-hour traffic (zone-based instead of nearest)." Maintenance is a new `ElevatorState`; peak-hour dispatch is a new `DispatchStrategy` implementation. Neither requires touching `Elevator` or `ElevatorController`'s core logic — again, that's the Open/Closed payoff.

---

## 6. Pattern Combinations You'll Actually See

Interview problems almost never use one pattern in isolation. Recognizing *combinations* is a stronger signal to an interviewer than reciting one pattern correctly.

| Problem | Typical combination |
|---|---|
| Parking Lot | Singleton (lot) + Strategy (fee calc) + Factory Method (vehicle types) + Observer (spot availability) |
| Elevator System | State (elevator status) + Strategy (dispatch logic) + Singleton (controller) + Observer (floor arrival events) |
| Splitwise / expense sharing | Strategy (split method: equal/percentage/exact) + Observer (notify group members) + Factory (split type creation) |
| Vending Machine | State (idle/selecting/dispensing/out-of-stock) + Factory Method (product dispensing) |
| Library Management | Factory Method (book/media types) + Observer (due-date/reservation notifications) + Strategy (fine calculation) |
| Ride-sharing (Uber-like) | Strategy (pricing/surge) + Observer (driver location updates) + State (ride status: requested/ongoing/completed) + Singleton (dispatcher) |

**The meta-lesson:** almost every LLD problem that involves "something that could plausibly change" uses Strategy, "something with distinct lifecycle stages" uses State, and "something many parts of the system need to react to" uses Observer. If you can identify which of your entities fall into those three buckets, you've mapped most of the design before writing a line of code.

---

## 7. Interview Practice Questions

1. Explain SRP using one of your own past design decisions — either from a class you split apart, or one you now realize should have been split.
2. Why does the Strategy pattern satisfy the Open/Closed Principle, but a big `if/else` chain choosing behavior does not?
3. Give an example of an LSP violation that *isn't* the classic Square/Rectangle one — from a domain you've actually worked in.
4. Why does the Composite pattern's "don't put `add()`/`remove()` in the base interface" pitfall relate to Interface Segregation?
5. Explain Dependency Inversion using the difference between `new ConcreteThing()` inside a class vs. injecting an interface through the constructor.
6. When is inheritance still the *right* choice over composition? Give a concrete example.
7. You're asked to "design a vending machine" cold. Walk through your first three steps before writing any code.
8. For the parking lot design above: if the interviewer says "now support electric vehicle charging spots that take longer and cost more," what changes, and what doesn't?
9. For the elevator design above: why is `State` a better fit than a `status` enum + `switch` statement here, given how many transitions and behaviors each state has?
10. Design (verbally, or in code) a Splitwise-style expense splitter. Which patterns would you reach for, and why?
11. What's the difference between using Singleton because "there should conceptually be one" vs. using it because "it's convenient to access from anywhere"? Which one is defensible in an interview, and why?
12. In the elevator's `DispatchStrategy`, what SOLID principle is being satisfied by injecting the strategy into `ElevatorController` rather than hardcoding `NearestElevatorStrategy` inside it?
