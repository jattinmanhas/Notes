# Object-Oriented Programming: Java first, then Go

A study guide for understanding OOP, writing better low-level designs, and answering interview questions. Each concept starts with Java and then explains the Go equivalent or the absence of an equivalent.

Use JDK 17+ and Go 1.22+ for the complete examples at the end. Earlier code blocks are focused excerpts: they illustrate one concept and are not intended to be concatenated into one source file. The final two programs are self-contained and include executable checks.

## Study map

1. Objects, classes, state and behavior
2. Encapsulation and invariants
3. Abstraction
4. Inheritance and embedding
5. Polymorphism, overriding and overloading
6. Interfaces and abstract classes
7. Association, aggregation, composition and delegation
8. Constructors, visibility, static, final and immutability
9. Equality, references, value semantics and generics
10. Exceptions, lifecycle and concurrency
11. SOLID and design patterns
12. Interview questions, runnable demonstrations and practice

## 1. What OOP actually means

Object-oriented programming organizes a program around objects that combine state with operations that act on that state. A useful object exposes behavior while protecting rules about what states are valid.

For a parking lot, a `Ticket` represents an entry event, a `ParkingLot` coordinates allocation, and a `FeePolicy` calculates charges. A well-designed program does not simply create one class for every noun: it decides which object owns each rule.

The useful questions are:

- What information does this object own?
- Which operations are allowed?
- Which conditions must remain true before and after an operation?
- Which collaborators does it need?
- What behavior should callers be able to replace?

Java is class-based and supports implementation inheritance. Go supports methods, encapsulation through packages, interfaces, polymorphism and composition, but has no classes or class inheritance. Describing Go as having “no OOP” loses these distinctions. It is more accurate to say that Go supports many object-oriented techniques without Java's class hierarchy model.

OOP is not synonymous with inheritance. A design that mostly uses interfaces and composition can be strongly object-oriented.

## 2. Class, object, state and behavior

### Java

A class defines a type, its fields, its methods and how its instances are initialized. An object is an instance of a class.

```java
final class Counter {
    private int value;

    Counter(int initial) {
        value = initial;
    }

    void increment() {
        value++;
    }

    int value() {
        return value;
    }
}

Counter first = new Counter(0);
Counter second = new Counter(10);
first.increment();
// first.value() == 1; second.value() == 10
```

`value` is state. `increment()` is behavior. Each object has its own instance fields. The class defines the operations shared by instances.

### Go

Go commonly uses a struct for state and receiver methods for behavior.

```go
type Counter struct {
    value int
}

func NewCounter(initial int) *Counter {
    return &Counter{value: initial}
}

func (c *Counter) Increment() {
    c.value++
}

func (c *Counter) Value() int {
    return c.value
}
```

A receiver says which type the method belongs to. `*Counter` is a pointer receiver, allowing the method to change the original counter. A `Counter` value receiver receives a copy of the counter value.

`NewCounter` is an ordinary function with a conventional name. Go does not reserve it as a language-level constructor, and it does not force callers to use it.

Also, Go methods are not restricted to structs: you can define methods on an appropriate defined type declared in your package, such as `type UserID string`. You cannot attach methods to arbitrary types from another package.

## 3. Encapsulation: protect invariants, not just fields

An invariant is a condition that must remain true for the object to be valid. For a wallet, an invariant might be `balance >= 0`.

### Java

```java
final class Wallet {
    private long balance;

    Wallet(long initial) {
        if (initial < 0) {
            throw new IllegalArgumentException("negative balance");
        }
        balance = initial;
    }

    boolean debit(long amount) {
        if (amount <= 0) {
            throw new IllegalArgumentException("positive amount required");
        }
        if (amount > balance) {
            return false;
        }
        balance -= amount;
        return true;
    }
}
```

The caller asks to debit; it cannot assign an arbitrary balance. A public `setBalance(long value)` that accepts any value would undermine the invariant even though the field is private.

Encapsulation is therefore about controlling state transitions, not automatically generating getters and setters for every field.

### Go

```go
type Wallet struct {
    balance int64
}

func (w *Wallet) Debit(amount int64) (bool, error) {
    if amount <= 0 {
        return false, errors.New("positive amount required")
    }
    if amount > w.balance {
        return false, nil
    }
    w.balance -= amount
    return true, nil
}
```

Lowercase `balance` is unexported: code outside the package cannot access it directly. Other files in the same package can access it. This is package-level encapsulation, not Java's class-level private access.

The example above describes sequential behavior. Concurrent debits require synchronizing the check and subtraction together; see the complete examples for locked versions. Private or unexported does not mean thread-safe.

**LLD connection:** seat holds, cache eviction and parking allocation all need an operation that protects several related pieces of state together.

## 4. Abstraction: expose the required operation

Abstraction describes a useful contract while allowing implementation details to vary.

### Java

```java
interface Sender {
    void send(String message);
}

final class EmailSender implements Sender {
    @Override
    public void send(String message) {
        System.out.println("EMAIL: " + message);
    }
}
```

The caller needs to send a message. It does not need to know SMTP commands, vendor SDK configuration or socket management.

### Go

```go
type Sender interface {
    Send(message string) error
}

type EmailSender struct{}

func (EmailSender) Send(message string) error {
    fmt.Println("EMAIL:", message)
    return nil
}
```

Go's interface is satisfied implicitly by the method set. There is no `implements` declaration. The error result is part of the contract; a method without that result would not satisfy this interface.

Abstraction can also be a function or a package API. Do not add an interface only to claim you used abstraction. Interfaces are useful when the consumer benefits from interchangeable behavior or a boundary that hides implementation.

### Abstraction versus encapsulation

| Concept | Question it answers | Example |
|---|---|---|
| Abstraction | What can I ask this collaborator to do? | `Sender.send(message)` |
| Encapsulation | Who is allowed to change its internal state, and how? | Only `Wallet.debit` updates the balance |

An interface helps abstraction but does not automatically protect the implementation's mutable fields. Private fields help encapsulation but do not automatically create a useful abstraction.

## 5. Inheritance in Java

Inheritance creates a subtype relationship and can reuse superclass implementation.

```java
class Animal {
    String speak() {
        return "animal";
    }

    String introduce() {
        return "I say " + speak();
    }
}

final class Dog extends Animal {
    @Override
    String speak() {
        return "woof";
    }
}

Animal animal = new Dog();
animal.speak();      // "woof"
animal.introduce();  // "I say woof"
```

`Dog` is an `Animal`, so a Dog reference can be used where an Animal reference is expected. The inherited `introduce()` calls the dynamically dispatched `speak()` on the same object, reaching Dog's implementation.

A Java class extends at most one class. It can implement multiple interfaces. Constructors are not inherited. Private superclass members are not directly accessible to a subclass. A subclass cannot override a final instance method, and static methods are hidden rather than overridden.

Inheritance is useful when the subtype can honor the superclass contract. Sharing fields or a few lines of code is not sufficient justification.

## 6. Go embedding is not Java inheritance

Go has no `extends`. Embedding promotes eligible fields and methods for convenient selection.

```go
type Animal struct{}

func (Animal) Speak() string {
    return "animal"
}

func (a Animal) Introduce() string {
    return "I say " + a.Speak()
}

type Dog struct {
    Animal
}

func (Dog) Speak() string {
    return "woof"
}

dog := Dog{}
dog.Speak()      // "woof"
dog.Introduce()  // "I say animal"
```

`dog.Introduce()` selects the promoted method on the embedded Animal. Inside that method, `a` is an Animal receiver. Its call to `a.Speak()` does not dynamically redirect to Dog's method.

Also, `Dog` cannot simply be assigned to a variable of type `Animal`. The embedded field `dog.Animal` is an Animal; the outer Dog is a distinct type.

For dynamic behavior, use an interface:

```go
type Speaker interface {
    Speak() string
}

func Introduce(s Speaker) string {
    return "I say " + s.Speak()
}

Introduce(Dog{}) // "I say woof"
```

This distinction is one of the most important answers in a Java-versus-Go OOP interview. Embedding can promote methods and help interface satisfaction, but it does not create superclass-style virtual calls.

Embedding two types with equally eligible methods of the same name can create an ambiguous selector. Use explicit selection or an outer method to resolve it; it is not Java multiple inheritance.

## 7. Polymorphism

Polymorphism allows code to operate through a common contract while behavior depends on the supplied implementation.

### Java: subtype polymorphism

```java
Sender sender = new EmailSender();
sender.send("hello");
```

The reference type determines which operations are available at compile time. The actual implementation determines the overriding instance method invoked at runtime.

For example, a method only declared on EmailSender is unavailable through a Sender reference unless you use another appropriate interface or a cast. Repeated casts can indicate that the abstraction is insufficient.

### Go: interface polymorphism

```go
var sender Sender = EmailSender{}
err := sender.Send("hello")
```

The interface stores a dynamic concrete type and a dynamic value. Its method call reaches the concrete implementation. Concrete types do not need a shared superclass or an explicit declaration that they implement the interface.

Polymorphism is how the notification dispatcher selects different channels, the parking lot uses different pricing policies, and the logger uses different formatters.

## 8. Overloading versus overriding

### Java overloading: chosen from declared types

```java
static String describe(Animal animal) {
    return "Animal overload";
}

static String describe(Dog dog) {
    return "Dog overload";
}

Animal animal = new Dog();
describe(animal); // "Animal overload"
```

Overloads have the same name and different parameter lists. The compiler selects an applicable method using compile-time types and overload-resolution rules. The runtime type of `animal` does not switch the selected overload to `describe(Dog)`.

Changing only a return type cannot create an overload.

### Java overriding: implementation chosen at runtime

`Dog.speak()` overrides `Animal.speak()`. Calling `animal.speak()` reaches Dog even when the reference is declared Animal.

An overriding method must use a compatible signature and return type, cannot reduce access, and cannot broaden checked exceptions beyond what the inherited contract permits. Covariant reference return types are allowed. Use `@Override` to let the compiler check your intention.

### Go

Go does not overload functions or methods by parameter types. Choose distinct names such as `FindByID` and `FindByEmail`, use a configuration struct, or use a type parameter when the operation is genuinely generic.

An outer method sharing a name with a promoted method is not Java overriding. Interface dispatch provides runtime polymorphism, as section 6 demonstrates.

| Feature | Java | Go |
|---|---|---|
| Same method name, different parameters | Overloading | Not supported |
| Subclass replaces instance implementation | Overriding | No class inheritance |
| Runtime behavior behind a contract | Interfaces and virtual instance methods | Interfaces |
| Shared operation over type parameters | Generics | Generics |

## 9. Interfaces versus abstract classes in Java

An interface primarily defines a capability or contract. An abstract class can define a partial implementation with per-instance state, constructors and protected extension points.

```java
abstract class Report {
    public final String render() {
        return "header|" + body() + "|footer";
    }

    protected abstract String body();
}

final class SalesReport extends Report {
    @Override
    protected String body() {
        return "sales";
    }
}
```

`render()` fixes a workflow while allowing a subclass to supply one step. This is the Template Method pattern. The final modifier prevents subclasses from changing the overall algorithm.

| Question | Interface | Abstract class |
|---|---|---|
| Can a class adopt multiple? | Yes | Only one direct superclass |
| Instance fields? | No ordinary per-instance fields | Yes |
| Constructor? | No | Yes, invoked during subclass construction |
| Implementation methods? | Default, static and private methods are possible | Concrete and abstract methods are possible |
| Direct instantiation? | No | No |

Interface fields are implicitly public static final. Interface default methods can provide behavior, but do not add instance fields. If unrelated interfaces provide conflicting defaults, the implementing class may need to explicitly resolve the conflict.

### Go equivalent

Go interfaces contain method contracts, not method bodies or instance state. Shared implementation goes in a concrete helper, a composed struct, or a function accepting a callback/interface.

```go
func Render(body func() string) string {
    return "header|" + body() + "|footer"
}

result := Render(func() string { return "sales" })
```

This preserves a fixed workflow using a function argument. It is an alternative design, not a Go abstract class.

## 10. Association, aggregation and composition

These describe relationships and ownership, not three different language keywords.

### Association: knows about or uses

A NotificationService knows about a Sender. Both Java and Go can represent this with a field or a method parameter. The relationship alone says little about lifecycle ownership.

### Aggregation: groups independently meaningful parts

A Team groups Employees that can exist independently of that team. It may retain references to employees, but deleting the team need not delete employees.

Aggregation terminology is often less useful than explicitly describing who creates, owns and removes the objects.

### Composition: owns or builds behavior from parts

A ParkingLot may own its configured ParkingSpots. At the domain level, removing the lot may remove its spots. Both languages still use ordinary fields/references to express the relation; ownership semantics come from the design and persistence rules.

“Prefer composition over inheritance” also uses composition more broadly: build behavior by containing and delegating to collaborators. This does not necessarily mean exclusive lifecycle ownership in the UML sense.

Garbage collection does not enforce domain deletion semantics. It reclaims unreachable memory, not business entities in a database.

## 11. Delegation and composition in practice

### Java

```java
final class NotificationService {
    private final Sender sender;

    NotificationService(Sender sender) {
        this.sender = Objects.requireNonNull(sender);
    }

    void notifyUser(String message) {
        sender.send(message);
    }
}
```

The service delegates delivery. It does not inherit from EmailSender, because a notification service is not an email sender subtype.

### Go

```go
type NotificationService struct {
    sender Sender
}

func NewNotificationService(sender Sender) *NotificationService {
    return &NotificationService{sender: sender}
}

func (s *NotificationService) NotifyUser(message string) error {
    return s.sender.Send(message)
}
```

Named fields make delegation explicit. Embedding is optional and can expose promoted methods you did not intend to make part of the outer type's API.

Constructor injection is simply passing a dependency during construction. It does not require Spring, a dependency injection container, reflection or a Go code generator.

## 12. Constructors, initialization and zero values

### Java

A constructor initializes an instance and may validate required state. Constructors can be overloaded, have no return type, and participate in superclass initialization. An abstract class constructor runs as part of creating its concrete subclass.

For the Java 17 baseline used here, an explicit `this(...)` or `super(...)` constructor invocation appears first in the constructor. Avoid calling overridable methods during construction: dynamic dispatch can reach subclass behavior before the subclass has finished initializing its fields.

Factories can return an interface or choose an implementation when ordinary construction is insufficient.

### Go

`NewWallet` is an ordinary function that can return `(*Wallet, error)`. It can validate configuration before returning a usable value.

The zero value exists even without calling a constructor:

```go
var wallet Wallet
```

Aim for useful zero values when practical. A zero `sync.Mutex` works. A nil map can be read but cannot receive element assignments until initialized. A nil channel blocks sends/receives indefinitely. A service requiring a sender is not useful until configured.

Document when construction is required. Do not assume naming a function `NewX` prevents callers from creating `X{}`.

`new(T)` allocates a zero T and returns `*T`. `make` initializes slices, maps and channels and returns their corresponding values. Neither is the same as invoking a Java constructor.

## 13. Visibility and access modifiers

| Visibility | Java | Go |
|---|---|---|
| Public API | `public` | Exported identifier begins with an uppercase letter |
| Package-scoped API | No access modifier | Unexported identifier |
| Class-private state | `private` | No direct equivalent; unexported is package-visible |
| Subclass-oriented access | `protected` | No subclass visibility mechanism |

Java protected access includes same-package access and specific subclass access rules across packages; it is not simply “public to all subclass references everywhere.” Top-level Java classes are normally public or package-private, not private/protected. Nested types have additional access possibilities.

In Go, files in the same package share access to unexported names. Moving a type into another file does not hide it. Package boundaries should reflect cohesive APIs, not a desire to imitate one Java class per package.

## 14. `this`, `super` and Go receivers

In Java, `this` denotes the current instance. `super.method()` explicitly invokes an accessible superclass implementation. `this(...)` and `super(...)` also appear in constructor chaining.

In Go, the receiver is an ordinary named parameter such as `w *Wallet`. There is no `this` or `super` keyword.

```go
dog.Animal.Speak() // explicitly select the embedded Animal's method
```

This selects a contained value's method. It is not a superclass call on the same inheritance hierarchy.

## 15. Static members and package-level functions

Java static members belong to the class rather than a particular object. Static methods have no `this` and cannot directly use instance fields without an instance.

Static methods are not dynamically overridden. Accessing them through an instance expression can be misleading; use the class name.

Go has package-level functions and variables, not static class members. A helper such as `time.Now()` is a package function. A package variable is shared state and needs synchronization if concurrently mutated.

Neither static state nor package-level state automatically makes a design a Singleton. Global mutable dependencies often make tests harder and hide coupling. Prefer explicit dependencies for services that need isolation or replacement.

## 16. `final`, immutability and defensive copying

### Java

- A final variable can be assigned once.
- A final method cannot be overridden.
- A final class cannot be subclassed.

A final reference does not make the referenced object immutable:

```java
final List<String> names = new ArrayList<>();
names.add("Alice"); // legal
```

For a value object, validate on construction, avoid setters and defensively copy mutable input/output.

```java
record Team(List<String> members) {
    Team {
        members = List.copyOf(members);
    }
}
```

Records provide final component fields, generated accessors, and generated equality/hashCode/toString. They are shallowly immutable: a record containing a mutable object does not freeze that object. `List.copyOf` prevents structural list mutation, but mutable elements inside the list can still change. With String elements, that concern does not arise.

### Go

Go has no general-purpose `final` variable or immutable-struct modifier. `const` applies to constant values, not arbitrary objects.

Use unexported state, carefully chosen methods and copying where necessary. Returning a struct by value is a shallow copy, not automatic deep immutability. Slices, maps and pointers inside it can still refer to shared state.

```go
copyOfNames := append([]string(nil), names...)
```

This copies a slice of strings into a separate backing array. For a slice of pointers, the pointed-to objects are still shared.

**LLD connection:** returning an order's internal item map or a hold's internal seat slice would let callers mutate business state outside the service lock. The guides copy those structures or return immutable records.

## 17. Identity, equality and hash-based collections

### Java

For object references, `==` checks identity: do these references denote the same object? `equals()` expresses logical equality when overridden. Primitive `==` compares primitive values.

```java
record UserId(String value) {}

UserId a = new UserId("u1");
UserId b = new UserId("u1");
a == b;       // false
a.equals(b);  // true
```

When overriding equals, override hashCode consistently. Equal objects must have equal hash codes. Unequal objects may share a hash code. Equality should be reflexive, symmetric, transitive and consistent, and comparison with null should be false.

Do not mutate fields used by equality/hashCode while an object is a HashMap key or a HashSet member. The collection may no longer locate it as expected.

### Go

Comparable types support `==`. A struct is comparable when all its fields are comparable. A struct containing strings can be a map key; one containing a slice or map cannot be a map key.

```go
type UserID struct { Value string }

a := UserID{Value: "u1"}
b := UserID{Value: "u1"}
fmt.Println(a == b) // true
```

Go has no custom operator overloading or Java-style equals/hashCode hooks for built-in maps. You can define an `Equal` method for application logic, but built-in `==` and map key comparison will not call it. Normalize keys or choose a suitable comparable key representation.

Slices, maps and functions can be compared to nil, but not to another value of the same type using `==`. Comparing interface values can panic when their dynamic contents are not comparable; an interface type alone does not guarantee safe equality for every value stored in it.

## 18. Pass-by-value, references and aliasing

Both Java and Go pass arguments by value. What is copied differs by type.

### Java

For an object argument, the copied value is a reference. A method can mutate the referenced object, but assigning its parameter to a different object does not reassign the caller's variable.

```java
void change(List<String> items) {
    items.add("visible to caller");
    items = new ArrayList<>(); // caller still refers to original list
}
```

“Java passes objects by reference” is therefore an imprecise interview answer. It passes references by value.

### Go

A struct parameter copies its fields. A pointer parameter copies the pointer, so both pointers refer to the same target. A slice parameter copies the slice descriptor, which still refers to a backing array.

```go
items := []string{"Alice"}
alias := items
alias[0] = "Bob"
// items[0] is now "Bob"
```

Appending to a slice may allocate a new backing array; it is not reliable to assume every append changes the caller's slice length or its storage. Return the updated slice when that is part of the contract.

Map values similarly refer to shared map data. Copying a map variable does not clone its entries.

Do not copy a Go mutex after it has been used. Types that own a mutex usually need pointer receivers and should not be copied as ordinary values.

## 19. Go method sets and pointer receivers

For a defined, non-interface type `T`, methods declared with receiver T belong to T's method set. The method set of `*T` includes methods with receiver T and `*T` (with embedding adding its own promotion rules).

```go
type Counter struct { value int }

func (c *Counter) Increment() { c.value++ }

type Incrementer interface { Increment() }

var c Counter
c.Increment() // allowed: c is addressable, so Go can use &c

var good Incrementer = &c
// var bad Incrementer = c // compile error
```

The convenient direct method call does not change Counter's method set for interface assignment. This is a common source of confusion.

Choose pointer receivers when methods mutate the instance, the type contains synchronization primitives, copying is expensive, or consistent reference semantics matter. Choose value receivers for small value-like types when copying matches their meaning. A value receiver containing pointers or maps can still mutate shared referenced data; “value receiver” is not a deep read-only guarantee.

## 20. The Go typed-nil interface trap

An interface is nil only when it has no dynamic type and no dynamic value.

```go
var concrete *ProviderError = nil
var err error = concrete

fmt.Println(concrete == nil) // true
fmt.Println(err == nil)      // false
```

`err` contains a dynamic type `*ProviderError` and a nil pointer value. That is different from a nil interface. Calling a method through it can panic if the implementation dereferences its nil receiver.

When a function has an error return, return literal `nil` on success instead of assigning a typed nil pointer to the error interface.

Java has null references, but does not have this exact Go interface-wrapper distinction. Java calls on null normally throw NullPointerException. Go pointer-receiver methods can explicitly handle a nil receiver, though callers should not assume they do.

## 21. Generics versus runtime interface polymorphism

Generics describe a reusable algorithm or data structure over types. Interface polymorphism describes a runtime behavioral contract. Both are useful, and they solve different problems.

### Java

```java
List<String> names = new ArrayList<>();
```

Generics provide compile-time type checking. Java generics generally use type erasure and do not accept primitive type arguments directly; use boxed types such as Integer. Generic collections are invariant: `List<Dog>` is not a subtype of `List<Animal>`.

Wildcards express bounded flexibility. The PECS mnemonic means “producer extends, consumer super”: a parameter that only reads Animals can often use `? extends Animal`; one that accepts Dogs can often use `? super Dog`. The correct signature depends on what the method actually does.

### Go

```go
type Cache[K comparable, V any] struct {
    values map[K]V
}
```

The constraint `comparable` permits keys usable by equality-based operations, and `any` is an alias for the empty interface. Type-parameter constraints can express type sets; interfaces with non-method type-set elements are used as constraints, not ordinary runtime variable types.

Do not explain Go generics as Java erasure: the languages have different semantics and implementation choices. Go methods can use their receiver type's parameters but cannot declare a new independent method type-parameter list. Use a generic function when that is needed.

Use a generic cache for key/value type safety. Use a Sender interface when email and SMS implementations should be interchangeable at runtime.

## 22. Exceptions and errors are part of the contract

### Java

Checked exceptions must be caught or declared. RuntimeException subclasses represent unchecked exceptions. A caller needs to know which failures are expected and whether retrying makes sense.

An overriding method cannot add broader checked exceptions than its inherited contract allows. A checked exception declaration is therefore part of the substitution rules.

### Go

Errors are ordinary values. Functions commonly return `(result, error)`. Handle errors at a layer that can make a meaningful decision. Use wrapping and `errors.Is`/`errors.As` when callers need to identify causes or error categories.

Panic is not a normal replacement for expected failures such as “parking lot full.” In interview samples, panicking on a programmer-supplied invalid constructor configuration can be an explicit simplification; production-facing constructors often return errors.

**LLD connection:** a retry policy needs a distinction between temporary and permanent failure. It should not blindly retry every exception/error or parse arbitrary error text.

## 23. Lifecycle and resource ownership

Memory management and resource management are different. Garbage collection does not guarantee timely release of a file, socket or database connection.

### Java

Use AutoCloseable and try-with-resources for explicit cleanup:

```java
try (var reader = Files.newBufferedReader(path)) {
    // consume the reader
}
```

Resources close in reverse order of initialization. Do not rely on finalization for timely cleanup.

### Go

Use explicit Close operations, commonly with defer after a successful acquisition:

```go
file, err := os.Open(path)
if err != nil {
    return err
}
defer file.Close()
```

For writes, Close/Flush errors can matter and may need explicit propagation; deferring a call does not automatically handle its returned error. Defer runs when the surrounding function returns, not at the end of each loop iteration.

The object that creates a resource should document whether it owns closing it. A logger wrapping stdout should not close stdout simply because that logger instance is no longer needed.

## 24. OOP does not automatically solve concurrency

```text
check balance >= amount
subtract amount
```

Two threads can both pass the check before either subtracts. Synchronize the whole operation, not just individual reads or writes.

In Java, synchronized provides mutual exclusion and memory visibility around a shared monitor. Volatile provides visibility/order guarantees for the field, but does not make compound operations such as `count++` atomic.

In Go, use mutexes, atomics or ownership through channels depending on the operation. Atomics suit certain individual counters; they do not automatically make a multi-map booking transaction atomic. Run the race detector to find exercised data races, but a passing run does not prove all interleavings are correct.

Immutable values simplify shared reading, but only if nested state is also safely immutable or otherwise controlled.

Locks inside one process do not coordinate multiple servers. Persistent seat ownership, payment posting and inventory reservations need a shared consistency mechanism.

## 25. SOLID explained through the LLD examples

### S — Single Responsibility Principle

A component should have a coherent reason to change. NotificationService coordinates notifications; EmailSender handles delivery; a repository handles persistence. Changing a provider should not require rewriting pricing or storage.

This does not mean every class gets exactly one method. A cache can coherently own get, put, expiry and recency updates because they maintain one data structure's invariant.

In Go, apply the same thinking to types and packages. Do not turn each method into a separate package.

### O — Open/Closed Principle

Keep stable orchestration extensible through the behavior that actually varies. A new FeePolicy can change parking charges without editing exit allocation logic. A new Formatter can change log output without rewriting fan-out.

Configuration and registration may still change. “Open/closed” does not mean no existing file can ever be edited. Creating speculative interfaces for every detail adds indirection without useful extensibility.

Java uses interfaces or carefully chosen inheritance. Go often uses small interfaces or function arguments.

### L — Liskov Substitution Principle

A subtype or replacement implementation should preserve the promises callers depend on.

If a Bird contract promises `fly()`, a Penguin implementation that always throws does not honor the promise. Improve the contract: define a Flying capability for things that can fly, instead of forcing every bird to pretend.

The famous mutable Square/Rectangle example has the same issue: if setting width is promised not to alter height, Square cannot enforce equal sides while preserving that contract.

Go has no superclass inheritance, but substitutability still applies to interfaces. A Sender that returns success without accepting a message, or a Clock that changes units, violates its behavioral contract even if it compiles.

### I — Interface Segregation Principle

Clients should not depend on capabilities they do not need. Prefer a small Sender contract over an interface requiring sending, billing, analytics and account provisioning.

Go interfaces are commonly defined near the consumer, because the consumer knows the minimal capability it needs. Java can follow the same principle even with explicit implements declarations.

Do not mechanically split every interface into one-method fragments when callers always use a cohesive group of operations.

### D — Dependency Inversion Principle

High-level orchestration should depend on useful abstractions rather than vendor details. Main chooses concrete dependencies and passes them in.

Dependency inversion is the design principle. Dependency injection is a technique for providing dependencies. They are related, but not synonyms: injecting a concrete vendor client can be dependency injection without a vendor-independent abstraction.

## 26. Patterns you should recognize, with Java and Go mappings

| Pattern | Java expression | Go expression | Example from the notes |
|---|---|---|---|
| Strategy | Interface plus implementations | Interface or function value | FeePolicy, Splitter, Formatter |
| Adapter | Wrapper implementing caller's interface | Struct implementing the expected methods | A sink wrapping a stream/writer |
| Decorator | Same interface, wrapping another implementation | Same interface, named `next` field | LoggingSender wraps Sender |
| Factory | Construction function/class chooses implementation | Constructor function returning a value/interface | Selecting a provider at setup |
| Registry | Map of already built implementations | Map keyed by channel/type | Sender lookup |
| Observer | Listener registrations and callback calls | Handler function registrations | In-process pub-sub |
| Template Method | Base algorithm with overridable steps | Usually composition/callback alternative | Report.render in Java |
| Repository | Persistence contract and adapter | Consumer-facing storage interface | Notification storage boundary |

A class named Factory that only retrieves existing objects from a map is more precisely a Registry. A list of sinks receiving the same log event is fan-out; it is not automatically Chain of Responsibility. An enum plus switch is not automatically State. Name the pattern from its behavior, not its filenames.

### Decorator versus inheritance

A LoggingSender contains another Sender and implements Sender itself. It adds behavior while preserving the caller's contract. You can layer wrappers such as logging, metrics and retries without creating combinations such as LoggedRetriedEmailSender and LoggedRetriedSmsSender subclasses.

Wrapping order matters. Logging outside retries records one logical request; logging inside retries may record each attempt. In production, specify the order and failure semantics deliberately.

## 27. Common mistakes and what to do instead

| Mistake | Better approach |
|---|---|
| Public setters for every field | Expose business operations that protect invariants |
| Inheritance only to reuse fields | Use composition unless there is a sound subtype contract |
| Treating Go embedding as virtual inheritance | Use explicit interfaces for dynamic dispatch |
| Assuming final/record means deep immutable | Copy mutable collections and consider their elements |
| Assuming private means thread-safe | Synchronize compound state transitions |
| Adding an interface for every class | Extract a boundary where substitution helps |
| Huge interfaces with unrelated responsibilities | Define capabilities around actual consumers |
| Global Singleton service for convenience | Wire instances explicitly in the composition root |
| Calling vendor I/O while holding a domain lock | Separate state coordination from external operations |
| Ignoring errors because the happy path compiles | Test invalid operations and failure behavior |
| Returning an internal mutable map/slice | Return a snapshot or expose focused query methods |
| Reusing mutable Java keys in HashMap | Use stable value keys with consistent equality |
| Returning typed nil as an error in Go | Return a literal nil interface on success |

## 28. Interview questions and concise answers

**Is Go object-oriented?** It supports methods, interfaces, polymorphism, encapsulation and composition. It does not use Java's classes and implementation inheritance model. Explain those capabilities rather than forcing a yes/no label.

**What is the difference between abstraction and encapsulation?** Abstraction defines the contract a caller needs. Encapsulation controls access and state transitions so invariants remain valid.

**Can Java achieve multiple inheritance?** A class has one direct superclass but can implement multiple interfaces. Default methods require conflict resolution in relevant cases; this is not multiple inherited instance-state layouts.

**Can you override a static method?** No. Static methods are hidden and selected without instance dynamic dispatch.

**Can you overload by changing only the return type?** No. The parameter list must distinguish Java overloads; Go does not support overloads this way at all.

**Why prefer composition?** It allows independent behavior changes without binding the object to a superclass implementation. Use inheritance when a stable subtype contract and shared algorithm genuinely fit.

**Is a Java final List immutable?** No. The reference cannot be reassigned, but the list can still mutate.

**Does a Go value receiver guarantee no mutation?** No. Its copied fields can include references to shared maps, slices or pointed-to objects.

**Why does `c.Increment()` compile but assigning c to an interface fail?** An addressable value allows a convenient pointer-method call, but that does not add pointer-receiver methods to the value type's interface method set.

**Why can a Go error contain a nil pointer and still be non-nil?** The interface has a concrete dynamic type even though its dynamic pointer value is nil.

**Do interfaces guarantee Liskov substitution?** They check method compatibility, not the behavioral promises. Error semantics, side effects and invariants still matter.

**When should I use an abstract class?** When related Java subtypes share a stable implementation/state model or an algorithm with controlled extension points. Use an interface for a capability implemented by otherwise unrelated types.

**Does dependency injection require a framework?** No. A constructor receiving a collaborator is dependency injection.

## 29. A practical LLD design checklist

1. Define the operations and what each returns on success or failure.
2. Identify immutable request/value types and mutable state owners.
3. State the invariants, such as one active booking per seat.
4. Put each invariant behind one coordinated operation.
5. Extract the behavior that actually varies into a policy interface or function.
6. Construct concrete collaborators in main.
7. Test an ordinary flow, an invalid transition and relevant contention/expiry cases.
8. Explain what changes when persistence or multiple servers enter the design.

For Java, express this using classes, records, interfaces and deliberate access modifiers. For Go, use cohesive packages, structs, methods, small interfaces and explicit errors. Keep the design intent consistent while respecting each language's semantics.

## 30. Complete runnable demonstrations

These programs demonstrate encapsulation, interface polymorphism, composition/decorators, value equality and defensive copying. They also deliberately contrast Java virtual dispatch with Go embedding. Each program contains explicit checks that fail if the expected behavior changes.

The Java types are nested here to make the comparison runnable by copying one file. This is an OOP experiment, not a suggested production folder structure; the system-design guides use separate files for each responsibility. The Java Sender returns a String so assertions can examine results; production delivery contracts should define acceptance and failures explicitly.

Create this small folder:

```text
oop-comparison/
  java/
    Main.java
  go/
    main.go
```

From `java/`:

```bash
javac --release 17 Main.java
java Main
```

From `go/`:

```bash
go run main.go
go run -race main.go
```

The Go program is intentionally one file, so it needs no go.mod for these explicit file-based commands. Its unexported fields are accessible elsewhere in that same main package; the visibility comparison concerns access across packages.

### Main.java — Java demonstration

```java
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Objects;

public class Main {
    interface Sender {
        String send(String message);

        default String channel() {
            return "generic";
        }
    }

    static final class EmailSender implements Sender {
        @Override
        public String send(String message) {
            return "EMAIL: " + message;
        }
    }

    static final class LoggingSender implements Sender {
        private final Sender next;

        LoggingSender(Sender next) {
            this.next = Objects.requireNonNull(next);
        }

        @Override
        public String send(String message) {
            return "LOG -> " + next.send(message);
        }
    }

    static final class NotificationService {
        private final Sender sender;

        NotificationService(Sender sender) {
            this.sender = Objects.requireNonNull(sender);
        }

        String notifyUser(String message) {
            return sender.send(message);
        }
    }

    abstract static class Report {
        public final String render() {
            return "header|" + body() + "|footer";
        }

        protected abstract String body();
    }

    static final class SalesReport extends Report {
        @Override
        protected String body() {
            return "sales";
        }
    }

    static class Animal {
        String speak() {
            return "animal";
        }

        String introduce() {
            return "I say " + speak();
        }
    }

    static final class Dog extends Animal {
        @Override
        String speak() {
            return "woof";
        }
    }

    static String describe(Animal animal) {
        return "Animal overload";
    }

    static String describe(Dog dog) {
        return "Dog overload";
    }

    static final class Wallet {
        private long balance;

        Wallet(long initial) {
            if (initial < 0) {
                throw new IllegalArgumentException("negative initial balance");
            }
            balance = initial;
        }

        synchronized boolean debit(long amount) {
            if (amount <= 0) {
                throw new IllegalArgumentException("amount must be positive");
            }
            if (amount > balance) {
                return false;
            }
            balance -= amount;
            return true;
        }

        synchronized long balance() {
            return balance;
        }
    }

    record UserId(String value) {
        UserId {
            Objects.requireNonNull(value);
        }
    }

    record Team(List<String> members) {
        Team {
            members = List.copyOf(members);
        }
    }

    static void check(boolean condition, String message) {
        if (!condition) {
            throw new AssertionError(message);
        }
    }

    public static void main(String[] args) {
        Animal animal = new Dog();
        check(animal.speak().equals("woof"), "dynamic dispatch");
        check(animal.introduce().equals("I say woof"), "inherited method dispatch");
        check(describe(animal).equals("Animal overload"), "compile-time overload");

        Wallet wallet = new Wallet(100);
        check(wallet.debit(40) && !wallet.debit(70), "debit invariant");
        check(wallet.balance() == 60, "balance invariant");

        var service = new NotificationService(new LoggingSender(new EmailSender()));
        check(service.notifyUser("hello").equals("LOG -> EMAIL: hello"), "composition");
        check(new SalesReport().render().equals("header|sales|footer"), "template method");

        Map<UserId, String> users = new HashMap<>();
        users.put(new UserId("u1"), "Alice");
        check(users.get(new UserId("u1")).equals("Alice"), "record value equality");

        var original = new java.util.ArrayList<>(List.of("Alice"));
        Team team = new Team(original);
        original.add("Bob");
        check(team.members().size() == 1, "defensive copy");

        System.out.println("Java: " + animal.introduce());
        System.out.println("Java: " + describe(animal));
        System.out.println("Java: " + service.notifyUser("hello"));
        System.out.println("Java: all checks passed");
    }
}
```

**Reading the code:** Wallet owns its balance invariant and synchronizes the whole debit. Sender is the capability, EmailSender is one implementation, and LoggingSender is a decorator because it implements the same contract while containing another Sender. NotificationService receives that contract through its constructor. Report illustrates Template Method, while Animal/Dog demonstrates virtual dispatch. The overloaded describe methods deliberately choose the Animal overload from the declared reference type. UserId demonstrates record value equality, and Team defensively copies its list.

### main.go — Go demonstration

```go
package main

import (
	"errors"
	"fmt"
	"sync"
)

type Sender interface {
	Send(string) string
}

type EmailSender struct{}

func (EmailSender) Send(message string) string { return "EMAIL: " + message }

type LoggingSender struct{ next Sender }

func (s LoggingSender) Send(message string) string {
	return "LOG -> " + s.next.Send(message)
}

type NotificationService struct{ sender Sender }

func (s NotificationService) NotifyUser(message string) string {
	return s.sender.Send(message)
}

type Animal struct{}

func (Animal) Speak() string { return "animal" }
func (a Animal) Introduce() string {
	return "I say " + a.Speak()
}

type Dog struct{ Animal }

func (Dog) Speak() string { return "woof" }

type Speaker interface{ Speak() string }

func Introduce(s Speaker) string { return "I say " + s.Speak() }

type Wallet struct {
	mu      sync.Mutex
	balance int64
}

func NewWallet(initial int64) (*Wallet, error) {
	if initial < 0 {
		return nil, errors.New("negative initial balance")
	}
	return &Wallet{balance: initial}, nil
}

func (w *Wallet) Debit(amount int64) (bool, error) {
	w.mu.Lock()
	defer w.mu.Unlock()
	if amount <= 0 {
		return false, errors.New("amount must be positive")
	}
	if amount > w.balance {
		return false, nil
	}
	w.balance -= amount
	return true, nil
}

func (w *Wallet) Balance() int64 {
	w.mu.Lock()
	defer w.mu.Unlock()
	return w.balance
}

// *Wallet has pointer-receiver methods; Wallet does not implement this interface.
type Debiter interface{ Debit(int64) (bool, error) }

var _ Debiter = (*Wallet)(nil)

type UserID struct{ Value string }

type ProviderError struct{ Message string }

func (e *ProviderError) Error() string {
	if e == nil {
		return "nil provider error pointer"
	}
	return e.Message
}

func check(condition bool, message string) {
	if !condition {
		panic(message)
	}
}

func main() {
	dog := Dog{}
	check(dog.Speak() == "woof", "direct method")
	check(dog.Introduce() == "I say animal", "embedding is not inheritance")
	check(Introduce(dog) == "I say woof", "interface dispatch")

	wallet, err := NewWallet(100)
	check(err == nil, "constructor")
	ok, err := wallet.Debit(40)
	check(ok && err == nil, "debit")
	ok, err = wallet.Debit(70)
	check(!ok && err == nil && wallet.Balance() == 60, "invariant")

	service := NotificationService{sender: LoggingSender{next: EmailSender{}}}
	check(service.NotifyUser("hello") == "LOG -> EMAIL: hello", "composition")

	users := map[UserID]string{{Value: "u1"}: "Alice"}
	check(users[UserID{Value: "u1"}] == "Alice", "value equality")

	var concrete *ProviderError
	var wrapped error = concrete
	check(concrete == nil && wrapped != nil, "typed nil interface")

	original := []string{"Alice"}
	alias := original
	alias[0] = "Bob"
	check(original[0] == "Bob", "slice copy aliases elements")
	detached := append([]string(nil), original...)
	detached[0] = "Cara"
	check(original[0] == "Bob", "detached slice")

	fmt.Println("Go embedding:", dog.Introduce())
	fmt.Println("Go interface:", Introduce(dog))
	fmt.Println("Go typed nil error is nil:", wrapped == nil)
	fmt.Println("Go: all checks passed")
}
```

**Reading the code:** The sender types satisfy Sender implicitly and use named-field composition. Wallet uses pointer receivers and a mutex; the compile-time Debiter assertion documents its method-set contract. Dog embeds Animal, but the promoted Introduce method still calls Animal.Speak. The separate Introduce function accepts Speaker and gets dynamic dispatch. ProviderError demonstrates a typed nil inside error, and the slice checks distinguish aliasing from a detached copy.

### Actual verified output

```text
Java: I say woof
Java: Animal overload
Java: LOG -> EMAIL: hello
Java: all checks passed

Go embedding: I say animal
Go interface: I say woof
Go typed nil error is nil: false
Go: all checks passed
```

Java compiled using --release 17. Go executed successfully with the race detector enabled. The assertions verify the demonstrated semantics; this does not constitute an exhaustive concurrency test suite.

## 31. What to practice after reading

1. Write a Java BankAccount and a Go BankAccount that reject an invalid withdrawal without changing state. Test concurrent withdrawals.
2. Implement a second Sender and inject it without modifying NotificationService.
3. Add a metrics decorator. Explain whether metrics counts logical requests or attempts, and how wrapper order changes that answer.
4. Run the Animal/Dog examples before reading their output. Explain every dispatch decision.
5. Intentionally return a typed nil error in Go, reproduce the problem, and fix the success return.
6. Build an immutable Java order with copied line items, then a Go order snapshot with copied maps/slices. Explain what is still shared.
7. Explain why a mutable Square may not substitute for a mutable Rectangle with independent width/height setters.
8. Take one LLD guide and identify where an interface is useful and where a concrete type is sufficient.
9. Explain why a database-backed service cannot get multi-server safety from synchronized or sync.Mutex alone.
10. Implement the same small design in both languages without translating Java inheritance into Go embedding mechanically.
