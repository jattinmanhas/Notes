# LLD Design Patterns — Implementation Reference (Java)

> **How to use this doc:** When you're stuck on *how* to structure a class, scroll to the pattern that matches your problem. Each entry has a one-line "use it when", the core idea, a Java example, and common pitfalls.

---

## Table of Contents

1. [Creational Patterns](#creational-patterns)
   - [Singleton](#1-singleton)
   - [Factory Method](#2-factory-method)
   - [Abstract Factory](#3-abstract-factory)
   - [Builder](#4-builder)
   - [Prototype](#5-prototype)
2. [Structural Patterns](#structural-patterns)
   - [Adapter](#6-adapter)
   - [Decorator](#7-decorator)
   - [Facade](#8-facade)
   - [Proxy](#9-proxy)
   - [Composite](#10-composite)
   - [Bridge](#11-bridge)
   - [Flyweight](#12-flyweight)
3. [Behavioral Patterns](#behavioral-patterns)
   - [Strategy](#13-strategy)
   - [Observer](#14-observer)
   - [Command](#15-command)
   - [State](#16-state)
   - [Chain of Responsibility](#17-chain-of-responsibility)
   - [Template Method](#18-template-method)
   - [Iterator](#19-iterator)
   - [Mediator](#20-mediator)
4. [LLD Interview Cheat Sheet](#lld-interview-cheat-sheet)

---

## Creational Patterns

> Control **how objects are created**.

---

### 1. Singleton

**Use it when:** You need exactly one instance of a class shared across the system (config, DB connection pool, logger).

**Core idea:** Keep a private static instance; use double-checked locking for thread safety.

```java
public class DatabaseConnection {

    private static volatile DatabaseConnection instance;
    private String connection;

    private DatabaseConnection() {
        this.connection = "connected"; // simulate DB connect
    }

    public static DatabaseConnection getInstance() {
        if (instance == null) {
            synchronized (DatabaseConnection.class) {  // thread-safe
                if (instance == null) {
                    instance = new DatabaseConnection();
                }
            }
        }
        return instance;
    }

    public String query(String sql) {
        return "Running: " + sql;
    }
}

// Usage
DatabaseConnection db1 = DatabaseConnection.getInstance();
DatabaseConnection db2 = DatabaseConnection.getInstance();
System.out.println(db1 == db2); // true — same object
```

**Pitfalls:**
- `volatile` keyword is required to prevent instruction reordering in Java memory model.
- Hard to unit-test; consider dependency injection instead for testability.
- Don't use for mutable shared state — leads to hidden coupling.

---

### 2. Factory Method

**Use it when:** You want to create objects but let subclasses/callers decide *which* class to instantiate.

**Core idea:** Define a `create()` method; use a registry map so adding a new type doesn't require touching the factory.

```java
// Product interface
public interface Notification {
    void send(String message);
}

// Concrete products
public class EmailNotification implements Notification {
    public void send(String message) { System.out.println("Email: " + message); }
}

public class SMSNotification implements Notification {
    public void send(String message) { System.out.println("SMS: " + message); }
}

public class PushNotification implements Notification {
    public void send(String message) { System.out.println("Push: " + message); }
}

// Factory
import java.util.HashMap;
import java.util.Map;
import java.util.function.Supplier;

public class NotificationFactory {
    private static final Map<String, Supplier<Notification>> REGISTRY = new HashMap<>();

    static {
        REGISTRY.put("email", EmailNotification::new);
        REGISTRY.put("sms",   SMSNotification::new);
        REGISTRY.put("push",  PushNotification::new);
    }

    public static Notification create(String channel) {
        Supplier<Notification> supplier = REGISTRY.get(channel);
        if (supplier == null) throw new IllegalArgumentException("Unknown channel: " + channel);
        return supplier.get();
    }
}

// Usage
Notification notif = NotificationFactory.create("email");
notif.send("Your OTP is 1234");
```

**Pitfalls:**
- Don't hard-code `if/else` chains — use the registry map so adding a new type is just one line.

---

### 3. Abstract Factory

**Use it when:** You need to create *families* of related objects that must be used together (e.g., UI components for different OSes).

**Core idea:** An abstract factory declares creation methods for each product type; concrete factories implement them for a specific family.

```java
// Abstract products
public interface Button   { void render(); }
public interface Checkbox { void render(); }

// Windows family
public class WindowsButton   implements Button   { public void render() { System.out.println("Windows Button");   } }
public class WindowsCheckbox implements Checkbox { public void render() { System.out.println("Windows Checkbox"); } }

// Mac family
public class MacButton   implements Button   { public void render() { System.out.println("Mac Button");   } }
public class MacCheckbox implements Checkbox { public void render() { System.out.println("Mac Checkbox"); } }

// Abstract factory
public interface UIFactory {
    Button   createButton();
    Checkbox createCheckbox();
}

// Concrete factories
public class WindowsFactory implements UIFactory {
    public Button   createButton()   { return new WindowsButton();   }
    public Checkbox createCheckbox() { return new WindowsCheckbox(); }
}

public class MacFactory implements UIFactory {
    public Button   createButton()   { return new MacButton();   }
    public Checkbox createCheckbox() { return new MacCheckbox(); }
}

// Client — doesn't know which OS
public class Application {
    private final UIFactory factory;

    public Application(UIFactory factory) { this.factory = factory; }

    public void renderUI() {
        factory.createButton().render();
        factory.createCheckbox().render();
    }
}

// Usage
new Application(new WindowsFactory()).renderUI();
new Application(new MacFactory()).renderUI();
```

**Pitfalls:**
- Adding a new product type (e.g., `Scrollbar`) forces changes to *all* concrete factories.
- Use only when products truly belong together as a family.

---

### 4. Builder

**Use it when:** An object has many optional parameters, and constructing it step-by-step is cleaner than a 10-argument constructor.

**Core idea:** Separate construction from representation; use method chaining to set fields, then call `build()`.

```java
public class QueryBuilder {
    private final String table;
    private List<String> columns   = List.of("*");
    private List<String> conditions = new ArrayList<>();
    private String orderBy = null;
    private int limit = -1;

    public QueryBuilder(String table) { this.table = table; }

    public QueryBuilder select(String... cols) {
        this.columns = Arrays.asList(cols);
        return this;  // enables chaining
    }

    public QueryBuilder where(String condition) {
        this.conditions.add(condition);
        return this;
    }

    public QueryBuilder orderBy(String col) {
        this.orderBy = col + " ASC";
        return this;
    }

    public QueryBuilder orderBy(String col, String direction) {
        this.orderBy = col + " " + direction;
        return this;
    }

    public QueryBuilder limit(int n) {
        this.limit = n;
        return this;
    }

    public String build() {
        String cols = String.join(", ", columns);
        StringBuilder sql = new StringBuilder("SELECT " + cols + " FROM " + table);
        if (!conditions.isEmpty())
            sql.append(" WHERE ").append(String.join(" AND ", conditions));
        if (orderBy != null)
            sql.append(" ORDER BY ").append(orderBy);
        if (limit > 0)
            sql.append(" LIMIT ").append(limit);
        return sql.toString();
    }
}

// Usage
String query = new QueryBuilder("users")
    .select("id", "name", "email")
    .where("age > 18")
    .where("active = true")
    .orderBy("name")
    .limit(50)
    .build();

System.out.println(query);
// SELECT id, name, email FROM users WHERE age > 18 AND active = true ORDER BY name ASC LIMIT 50
```

**Pitfalls:**
- Don't use Builder for simple objects — it adds boilerplate for no gain.
- Always validate in `build()`, not in individual setters.

---

### 5. Prototype

**Use it when:** Creating a new object is expensive (DB call, heavy init), and you can clone an existing one instead.

**Core idea:** Implement `Cloneable` or a custom `clone()` method that returns a deep copy.

```java
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

public class ReportTemplate implements Cloneable {
    private String title;
    private List<String> sections;    // mutable — needs deep copy
    private Map<String, String> styles; // mutable — needs deep copy

    public ReportTemplate(String title, List<String> sections, Map<String, String> styles) {
        this.title    = title;
        this.sections = new ArrayList<>(sections);
        this.styles   = new HashMap<>(styles);
    }

    @Override
    public ReportTemplate clone() {
        // Deep copy — new lists/maps, not shared references
        return new ReportTemplate(this.title, new ArrayList<>(sections), new HashMap<>(styles));
    }

    public void setTitle(String title)       { this.title = title; }
    public void addSection(String section)   { this.sections.add(section); }
    public List<String> getSections()        { return sections; }
}

// Usage
ReportTemplate base = new ReportTemplate(
    "Q1 Report",
    new ArrayList<>(List.of("Summary", "Details")),
    new HashMap<>(Map.of("font", "Arial"))
);

ReportTemplate q2 = base.clone();
q2.setTitle("Q2 Report");
q2.addSection("Forecast");

System.out.println(base.getSections()); // [Summary, Details]  — unaffected
System.out.println(q2.getSections());   // [Summary, Details, Forecast]
```

**Pitfalls:**
- Java's default `super.clone()` is a shallow copy — always deep-copy mutable fields manually.

---

## Structural Patterns

> Control **how classes and objects are composed**.

---

### 6. Adapter

**Use it when:** You need to make two incompatible interfaces work together without changing their source code.

**Core idea:** Wrap the incompatible class in an adapter that translates calls to the expected interface.

```java
// Existing class with incompatible interface
public class LegacyPaymentProcessor {
    public void makePayment(int amountCents, String cardNumber) {
        System.out.println("Legacy: Charging " + amountCents + " cents to " + cardNumber);
    }
}

// Interface your system expects
public interface PaymentGateway {
    void pay(double amountUsd, String card);
}

// Adapter
public class LegacyPaymentAdapter implements PaymentGateway {
    private final LegacyPaymentProcessor legacy;

    public LegacyPaymentAdapter(LegacyPaymentProcessor legacy) {
        this.legacy = legacy;
    }

    @Override
    public void pay(double amountUsd, String card) {
        int amountCents = (int) (amountUsd * 100);
        legacy.makePayment(amountCents, card);
    }
}

// Usage
PaymentGateway gateway = new LegacyPaymentAdapter(new LegacyPaymentProcessor());
gateway.pay(29.99, "4111-1111-1111-1111");
```

**Pitfalls:**
- Adapter is a *wrapper* — don't add business logic inside it.
- If you control both sides, just refactor instead of adapting.

---

### 7. Decorator

**Use it when:** You want to add behavior to an object dynamically without subclassing (logging, caching, auth, rate-limiting).

**Core idea:** Wrap the object in a decorator that implements the same interface, adds behavior, then delegates to the wrapped object.

```java
// Component interface
public interface DataFetcher {
    String fetch(String url);
}

// Concrete component
public class HttpFetcher implements DataFetcher {
    public String fetch(String url) {
        return "<data from " + url + ">";
    }
}

// Decorator: caching
public class CachingDecorator implements DataFetcher {
    private final DataFetcher fetcher;
    private final Map<String, String> cache = new HashMap<>();

    public CachingDecorator(DataFetcher fetcher) { this.fetcher = fetcher; }

    @Override
    public String fetch(String url) {
        return cache.computeIfAbsent(url, fetcher::fetch);
    }
}

// Decorator: logging
public class LoggingDecorator implements DataFetcher {
    private final DataFetcher fetcher;

    public LoggingDecorator(DataFetcher fetcher) { this.fetcher = fetcher; }

    @Override
    public String fetch(String url) {
        System.out.println("[LOG] Fetching: " + url);
        String result = fetcher.fetch(url);
        System.out.println("[LOG] Done");
        return result;
    }
}

// Usage — stack decorators
DataFetcher fetcher = new LoggingDecorator(new CachingDecorator(new HttpFetcher()));
fetcher.fetch("https://api.example.com/data");
fetcher.fetch("https://api.example.com/data"); // served from cache, still logged
```

**Pitfalls:**
- Deep decorator stacks are hard to debug — keep them shallow.
- In Java, don't confuse this with Spring AOP proxies (same idea, different mechanism).

---

### 8. Facade

**Use it when:** You want to provide a simple interface to a complex subsystem (e.g., a `VideoConverter` that hides codec, bitrate, muxer classes).

**Core idea:** Create a single class that orchestrates subsystem calls so clients don't need to know the internals.

```java
// Complex subsystem classes
class VideoDecoder {
    String decode(String file) { return "decoded(" + file + ")"; }
}

class AudioDecoder {
    String decode(String file) { return "audio_decoded(" + file + ")"; }
}

class VideoEncoder {
    String encode(String data, String fmt) { return "encoded_to_" + fmt + "(" + data + ")"; }
}

class FileSaver {
    void save(String data, String path) { System.out.println("Saved " + data + " -> " + path); }
}

// Facade
public class VideoConverterFacade {
    private final VideoDecoder vDec = new VideoDecoder();
    private final AudioDecoder aDec = new AudioDecoder();
    private final VideoEncoder enc  = new VideoEncoder();
    private final FileSaver    saver = new FileSaver();

    public void convert(String inputFile, String outputPath, String fmt) {
        String video   = vDec.decode(inputFile);
        String audio   = aDec.decode(inputFile);
        String encoded = enc.encode(video + "+" + audio, fmt);
        saver.save(encoded, outputPath);
    }
}

// Usage — client needs to know nothing about the subsystem
VideoConverterFacade converter = new VideoConverterFacade();
converter.convert("movie.avi", "/output/movie.mp4", "mp4");
```

**Pitfalls:**
- Facade should be a convenience layer, not a god class — don't add business logic here.
- Keep subsystem classes accessible (package-private or public) for advanced users.

---

### 9. Proxy

**Use it when:** You want to control access to an object — for lazy initialization, access control, remote calls, or logging.

**Core idea:** Proxy implements the same interface as the real subject and intercepts calls before/after delegating.

```java
// Subject interface
public interface Image {
    void display();
}

// Real subject — expensive to create
public class RealImage implements Image {
    private final String path;

    public RealImage(String path) {
        this.path = path;
        load();  // expensive
    }

    private void load() {
        System.out.println("Loading image from disk: " + path);
    }

    @Override
    public void display() {
        System.out.println("Displaying: " + path);
    }
}

// Virtual proxy — defers loading until display() is called
public class ImageProxy implements Image {
    private final String path;
    private RealImage realImage = null;  // not loaded yet

    public ImageProxy(String path) { this.path = path; }

    @Override
    public void display() {
        if (realImage == null) {
            realImage = new RealImage(path);  // lazy init
        }
        realImage.display();
    }
}

// Usage
Image img = new ImageProxy("photo.jpg");  // no disk read yet
img.display();  // loads now
img.display();  // reuses cached instance
```

**Pitfalls:**
- Don't add business logic in the proxy — it's an access control/loading layer.
- Virtual proxy (lazy load), protection proxy (auth), remote proxy (RPC) are the 3 main variants.

---

### 10. Composite

**Use it when:** You have a tree structure where leaves and branches should be treated uniformly (file systems, UI component trees, org charts).

**Core idea:** Both leaf and composite nodes implement the same interface; composite delegates to its children.

```java
import java.util.ArrayList;
import java.util.List;

// Component
public abstract class FileSystemItem {
    protected final String name;

    public FileSystemItem(String name) { this.name = name; }

    public abstract int getSize();

    public void display(int indent) {
        System.out.println(" ".repeat(indent) + name);
    }
}

// Leaf
public class File extends FileSystemItem {
    private final int size;

    public File(String name, int size) {
        super(name);
        this.size = size;
    }

    @Override
    public int getSize() { return size; }
}

// Composite
public class Folder extends FileSystemItem {
    private final List<FileSystemItem> children = new ArrayList<>();

    public Folder(String name) { super(name); }

    public void add(FileSystemItem item) { children.add(item); }

    @Override
    public int getSize() {
        return children.stream().mapToInt(FileSystemItem::getSize).sum();
    }

    @Override
    public void display(int indent) {
        super.display(indent);
        children.forEach(c -> c.display(indent + 2));
    }
}

// Usage
Folder root = new Folder("root");
Folder src  = new Folder("src");
src.add(new File("Main.java", 1200));
src.add(new File("Utils.java", 800));
root.add(src);
root.add(new File("README.md", 300));

root.display(0);
System.out.println("Total size: " + root.getSize() + " bytes");
```

**Pitfalls:**
- Avoid putting child-management methods (`add`, `remove`) in the base interface — it breaks leaf nodes.

---

### 11. Bridge

**Use it when:** You have two orthogonal dimensions of variation (e.g., Shape × Renderer) and want to avoid a class explosion.

**Core idea:** Separate the abstraction (Shape) from the implementation (Renderer) and hold a reference instead of inheriting.

```java
// Implementation hierarchy
public interface Renderer {
    void renderCircle(int x, int y, int radius);
}

public class VectorRenderer implements Renderer {
    public void renderCircle(int x, int y, int radius) {
        System.out.println("Vector circle at (" + x + "," + y + ") r=" + radius);
    }
}

public class RasterRenderer implements Renderer {
    public void renderCircle(int x, int y, int radius) {
        System.out.println("Raster circle pixels at (" + x + "," + y + ") r=" + radius);
    }
}

// Abstraction hierarchy
public abstract class Shape {
    protected final Renderer renderer;

    public Shape(Renderer renderer) { this.renderer = renderer; }

    public abstract void draw();
}

public class Circle extends Shape {
    private final int x, y, radius;

    public Circle(Renderer renderer, int x, int y, int radius) {
        super(renderer);
        this.x = x; this.y = y; this.radius = radius;
    }

    @Override
    public void draw() {
        renderer.renderCircle(x, y, radius);
    }
}

// Usage
Shape c1 = new Circle(new VectorRenderer(), 0, 0, 5);
Shape c2 = new Circle(new RasterRenderer(), 0, 0, 5);
c1.draw();
c2.draw();
```

**Pitfalls:**
- Don't use Bridge when you only have one dimension of variation — it adds unnecessary complexity.

---

### 12. Flyweight

**Use it when:** You have a *huge* number of similar objects and memory is a concern. Share intrinsic (unchanging) state; store extrinsic (contextual) state outside.

**Core idea:** Cache and reuse shared objects; pass context at runtime.

```java
import java.util.HashMap;
import java.util.Map;

// Intrinsic state — shared, immutable
public final class CharacterStyle {
    private final String font;
    private final int size;
    private final String color;

    public CharacterStyle(String font, int size, String color) {
        this.font = font; this.size = size; this.color = color;
    }
}

// Flyweight factory
public class StyleFactory {
    private static final Map<String, CharacterStyle> cache = new HashMap<>();

    public static CharacterStyle get(String font, int size, String color) {
        String key = font + "-" + size + "-" + color;
        return cache.computeIfAbsent(key, k -> new CharacterStyle(font, size, color));
    }

    public static int cacheSize() { return cache.size(); }
}

// Extrinsic state — unique per character instance
public class Character {
    private final char c;
    private final int x, y;
    private final CharacterStyle style;  // shared flyweight

    public Character(char c, int x, int y, CharacterStyle style) {
        this.c = c; this.x = x; this.y = y; this.style = style;
    }
}

// Usage — 1 million characters, only 1 style object
List<Character> chars = new ArrayList<>();
for (int i = 0; i < 1_000_000; i++) {
    chars.add(new Character('A', i, 0, StyleFactory.get("Arial", 12, "black")));
}
System.out.println("Style objects in cache: " + StyleFactory.cacheSize()); // 1
```

**Pitfalls:**
- Flyweight adds complexity — only worth it if profiling shows memory is genuinely a bottleneck.
- Never store mutable state in the flyweight.

---

## Behavioral Patterns

> Control **how objects communicate and distribute responsibility**.

---

### 13. Strategy

**Use it when:** You have multiple algorithms for the same task and want to swap them at runtime without `if/else` chains.

**Core idea:** Define a family of algorithms behind a common interface; inject the desired one at runtime.

```java
import java.util.Arrays;
import java.util.List;

// Strategy interface
public interface SortStrategy {
    List<Integer> sort(List<Integer> data);
}

// Concrete strategies
public class BubbleSort implements SortStrategy {
    public List<Integer> sort(List<Integer> data) {
        Integer[] arr = data.toArray(new Integer[0]);
        for (int i = 0; i < arr.length; i++)
            for (int j = 0; j < arr.length - i - 1; j++)
                if (arr[j] > arr[j+1]) { int t = arr[j]; arr[j] = arr[j+1]; arr[j+1] = t; }
        return Arrays.asList(arr);
    }
}

public class MergeSort implements SortStrategy {
    public List<Integer> sort(List<Integer> data) {
        if (data.size() <= 1) return data;
        int mid = data.size() / 2;
        List<Integer> left  = sort(data.subList(0, mid));
        List<Integer> right = sort(data.subList(mid, data.size()));
        return merge(left, right);
    }

    private List<Integer> merge(List<Integer> l, List<Integer> r) {
        // standard merge implementation
        List<Integer> result = new ArrayList<>();
        int i = 0, j = 0;
        while (i < l.size() && j < r.size())
            result.add(l.get(i) <= r.get(j) ? l.get(i++) : r.get(j++));
        while (i < l.size()) result.add(l.get(i++));
        while (j < r.size()) result.add(r.get(j++));
        return result;
    }
}

// Context
public class Sorter {
    private SortStrategy strategy;

    public Sorter(SortStrategy strategy) { this.strategy = strategy; }

    public void setStrategy(SortStrategy strategy) { this.strategy = strategy; }

    public List<Integer> sort(List<Integer> data) { return strategy.sort(data); }
}

// Usage
Sorter sorter = new Sorter(new BubbleSort());
System.out.println(sorter.sort(List.of(5, 3, 1, 4, 2)));

sorter.setStrategy(new MergeSort());
System.out.println(sorter.sort(List.of(5, 3, 1, 4, 2)));
```

**Pitfalls:**
- If you only have 2 strategies that never change, a simple boolean flag is fine — don't over-engineer.

---

### 14. Observer

**Use it when:** One object's state change should automatically notify many dependents (event systems, pub/sub, MVC).

**Core idea:** Subject maintains a list of observers and calls `update()` on each when state changes.

```java
import java.util.*;

// Observer interface
public interface Observer {
    void update(String event, Object data);
}

// Event bus (Subject)
public class EventBus {
    private final Map<String, List<Observer>> listeners = new HashMap<>();

    public void subscribe(String event, Observer observer) {
        listeners.computeIfAbsent(event, k -> new ArrayList<>()).add(observer);
    }

    public void unsubscribe(String event, Observer observer) {
        listeners.getOrDefault(event, Collections.emptyList()).remove(observer);
    }

    public void publish(String event, Object data) {
        listeners.getOrDefault(event, Collections.emptyList())
                 .forEach(o -> o.update(event, data));
    }
}

// Concrete observers
public class EmailService implements Observer {
    public void update(String event, Object data) {
        System.out.println("Email sent for " + event + ": " + data);
    }
}

public class AnalyticsService implements Observer {
    public void update(String event, Object data) {
        System.out.println("Analytics tracked " + event + ": " + data);
    }
}

// Usage
EventBus bus = new EventBus();
bus.subscribe("user_signup", new EmailService());
bus.subscribe("user_signup", new AnalyticsService());

bus.publish("user_signup", Map.of("userId", 42, "email", "x@x.com"));
```

**Pitfalls:**
- Observers should not modify the subject during `update()` — causes cascading updates.
- Always provide `unsubscribe()` to avoid memory leaks (especially with long-lived subjects).

---

### 15. Command

**Use it when:** You want to encapsulate a request as an object — for undo/redo, queuing, logging, or transactional behavior.

**Core idea:** Wrap each action in a Command object with `execute()` and `undo()`; store history for replay.

```java
import java.util.ArrayDeque;
import java.util.Deque;

// Command interface
public interface Command {
    void execute();
    void undo();
}

// Receiver
public class TextEditor {
    private StringBuilder text = new StringBuilder();

    public void append(String s)  { text.append(s); }
    public void removeLast(int n) { text.delete(text.length() - n, text.length()); }
    public String getText()       { return text.toString(); }
}

// Concrete command
public class InsertTextCommand implements Command {
    private final TextEditor editor;
    private final String text;

    public InsertTextCommand(TextEditor editor, String text) {
        this.editor = editor;
        this.text   = text;
    }

    @Override public void execute() { editor.append(text); }
    @Override public void undo()    { editor.removeLast(text.length()); }
}

// Invoker
public class CommandHistory {
    private final Deque<Command> history = new ArrayDeque<>();

    public void execute(Command cmd) {
        cmd.execute();
        history.push(cmd);
    }

    public void undo() {
        if (!history.isEmpty()) history.pop().undo();
    }
}

// Usage
TextEditor editor   = new TextEditor();
CommandHistory hist = new CommandHistory();

hist.execute(new InsertTextCommand(editor, "Hello"));
hist.execute(new InsertTextCommand(editor, " World"));
System.out.println(editor.getText()); // Hello World

hist.undo();
System.out.println(editor.getText()); // Hello
```

**Pitfalls:**
- Storing full object state per command can be memory-intensive — store deltas instead.

---

### 16. State

**Use it when:** An object's behavior changes dramatically based on its internal state, and you have many `if/else state == X` blocks.

**Core idea:** Each state is its own class; the context delegates behavior to the current state object.

```java
// State interface
public interface OrderState {
    void confirm(Order order);
    void ship(Order order);
    void deliver(Order order);
}

// Concrete states
public class PendingState implements OrderState {
    public void confirm(Order order) {
        System.out.println("Order confirmed!");
        order.setState(new ConfirmedState());
    }
    public void ship(Order order)    { System.out.println("Cannot ship — not confirmed yet"); }
    public void deliver(Order order) { System.out.println("Cannot deliver — not shipped yet"); }
}

public class ConfirmedState implements OrderState {
    public void confirm(Order order) { System.out.println("Already confirmed"); }
    public void ship(Order order) {
        System.out.println("Order shipped!");
        order.setState(new ShippedState());
    }
    public void deliver(Order order) { System.out.println("Cannot deliver — not shipped yet"); }
}

public class ShippedState implements OrderState {
    public void confirm(Order order) { System.out.println("Already confirmed"); }
    public void ship(Order order)    { System.out.println("Already shipped"); }
    public void deliver(Order order) {
        System.out.println("Order delivered!");
        order.setState(new DeliveredState());
    }
}

public class DeliveredState implements OrderState {
    public void confirm(Order order) { System.out.println("Order is already delivered"); }
    public void ship(Order order)    { System.out.println("Order is already delivered"); }
    public void deliver(Order order) { System.out.println("Order is already delivered"); }
}

// Context
public class Order {
    private OrderState state = new PendingState();

    public void setState(OrderState state) { this.state = state; }
    public void confirm() { state.confirm(this); }
    public void ship()    { state.ship(this); }
    public void deliver() { state.deliver(this); }
}

// Usage
Order order = new Order();
order.ship();      // Cannot ship — not confirmed yet
order.confirm();   // Order confirmed!
order.ship();      // Order shipped!
order.deliver();   // Order delivered!
```

**Pitfalls:**
- If states are simple (2-3 states, rare transitions), an enum + `switch` is cleaner.

---

### 17. Chain of Responsibility

**Use it when:** You want to pass a request along a chain of handlers until one handles it (middleware, validation pipelines, support tiers).

**Core idea:** Each handler either handles the request or passes it to the next handler.

```java
import java.util.Map;

// Handler abstract class
public abstract class Handler {
    private Handler next;

    public Handler setNext(Handler next) {
        this.next = next;
        return next;  // enables chaining: auth.setNext(rate).setNext(biz)
    }

    public abstract void handle(Map<String, Object> request);

    protected void passToNext(Map<String, Object> request) {
        if (next != null) next.handle(request);
        else System.out.println("Request unhandled");
    }
}

// Concrete handlers
public class AuthHandler extends Handler {
    public void handle(Map<String, Object> request) {
        if (request.get("token") == null) System.out.println("Rejected: No auth token");
        else passToNext(request);
    }
}

public class RateLimitHandler extends Handler {
    public void handle(Map<String, Object> request) {
        if (Boolean.TRUE.equals(request.get("rateExceeded"))) System.out.println("Rejected: Rate limit exceeded");
        else passToNext(request);
    }
}

public class BusinessHandler extends Handler {
    public void handle(Map<String, Object> request) {
        System.out.println("Processing request: " + request);
    }
}

// Build the chain
AuthHandler      auth = new AuthHandler();
RateLimitHandler rate = new RateLimitHandler();
BusinessHandler  biz  = new BusinessHandler();
auth.setNext(rate).setNext(biz);  // wait — setNext returns next, so: auth->rate->biz

auth.handle(Map.of("token", "abc", "data", "payload"));           // reaches business
auth.handle(Map.of("token", "abc", "rateExceeded", true));        // blocked at rate
auth.handle(Map.of("data", "payload"));                           // blocked at auth
```

**Pitfalls:**
- No guarantee a request gets handled — add a fallback at the end of the chain.

---

### 18. Template Method

**Use it when:** Multiple classes share the same algorithm skeleton but differ in specific steps.

**Core idea:** Define the algorithm in a `final` base class method; let subclasses override specific abstract steps (hooks).

```java
import java.util.List;

// Abstract class with template method
public abstract class DataImporter {

    // Template method — sealed algorithm
    public final void importData(String source) {
        String raw       = read(source);
        List<String> parsed    = parse(raw);
        List<String> validated = validate(parsed);
        save(validated);
    }

    protected abstract String       read(String source);
    protected abstract List<String> parse(String raw);

    // Hook — default implementation, subclasses may override
    protected List<String> validate(List<String> data) {
        return data.stream().filter(row -> !row.isEmpty()).toList();
    }

    protected abstract void save(List<String> data);
}

// Concrete importer: CSV
public class CsvImporter extends DataImporter {
    protected String       read(String source)  { return "a,1\nb,2"; }
    protected List<String> parse(String raw)    { return List.of(raw.split("\n")); }
    protected void         save(List<String> d) { System.out.println("Saved " + d.size() + " CSV rows"); }
}

// Concrete importer: JSON
public class JsonImporter extends DataImporter {
    protected String       read(String source)  { return "[{\"a\":1},{\"b\":2}]"; }
    protected List<String> parse(String raw)    { return List.of(raw.replace("[","").replace("]","").split(",")); }
    protected void         save(List<String> d) { System.out.println("Saved " + d.size() + " JSON records"); }
}

// Usage
new CsvImporter().importData("data.csv");
new JsonImporter().importData("data.json");
```

**Pitfalls:**
- Mark the template method `final` so subclasses can't break the algorithm skeleton.
- Prefer composition (Strategy) if the *whole* algorithm varies, not just steps.

---

### 19. Iterator

**Use it when:** You want to traverse a collection without exposing its internal structure.

**Core idea:** Implement `java.util.Iterator<T>` with `hasNext()` and `next()`.

```java
import java.util.*;

public class TreeNode {
    int value;
    List<TreeNode> children;

    public TreeNode(int value, TreeNode... children) {
        this.value    = value;
        this.children = Arrays.asList(children);
    }
}

// BFS Iterator
public class BfsIterator implements Iterator<Integer> {
    private final Queue<TreeNode> queue = new LinkedList<>();

    public BfsIterator(TreeNode root) {
        if (root != null) queue.add(root);
    }

    @Override
    public boolean hasNext() { return !queue.isEmpty(); }

    @Override
    public Integer next() {
        if (!hasNext()) throw new NoSuchElementException();
        TreeNode node = queue.poll();
        queue.addAll(node.children);
        return node.value;
    }
}

// Usage
TreeNode root = new TreeNode(1,
    new TreeNode(2, new TreeNode(4), new TreeNode(5)),
    new TreeNode(3, new TreeNode(6))
);

Iterator<Integer> it = new BfsIterator(root);
while (it.hasNext()) System.out.print(it.next() + " "); // 1 2 3 4 5 6
```

---

### 20. Mediator

**Use it when:** Many objects communicate with each other directly, creating tight coupling. Centralize communication through a mediator (chat rooms, air traffic control, UI form coordination).

**Core idea:** Objects don't reference each other — they only talk to the mediator.

```java
import java.util.ArrayList;
import java.util.List;

// Mediator
public class ChatMediator {
    private final List<User> users = new ArrayList<>();

    public void register(User user) { users.add(user); }

    public void send(String message, User sender) {
        users.stream()
             .filter(u -> u != sender)
             .forEach(u -> u.receive("[" + sender.getName() + "]: " + message));
    }
}

// Colleague
public class User {
    private final String name;
    private final ChatMediator mediator;

    public User(String name, ChatMediator mediator) {
        this.name     = name;
        this.mediator = mediator;
        mediator.register(this);
    }

    public String getName() { return name; }

    public void send(String message) { mediator.send(message, this); }

    public void receive(String message) {
        System.out.println(name + " received — " + message);
    }
}

// Usage
ChatMediator room = new ChatMediator();
User alice   = new User("Alice",   room);
User bob     = new User("Bob",     room);
User charlie = new User("Charlie", room);

alice.send("Hey everyone!");
// Bob received — [Alice]: Hey everyone!
// Charlie received — [Alice]: Hey everyone!
```

**Pitfalls:**
- Mediator can become a God Object if it grows too large — split by domain if needed.

---

## LLD Interview Cheat Sheet

| Problem clue | Pattern |
|---|---|
| "Only one instance" | Singleton |
| "Create object without specifying class" | Factory Method |
| "Family of related objects" | Abstract Factory |
| "Complex object, many optional params" | Builder |
| "Clone expensive object" | Prototype |
| "Incompatible interfaces" | Adapter |
| "Add behavior without subclassing" | Decorator |
| "Simplify complex subsystem" | Facade |
| "Control access / lazy load" | Proxy |
| "Tree structure, treat uniformly" | Composite |
| "Two independent hierarchies" | Bridge |
| "Millions of similar objects" | Flyweight |
| "Swap algorithms at runtime" | Strategy |
| "Notify many on state change" | Observer |
| "Undo/redo / queue requests" | Command |
| "Behavior changes with state" | State |
| "Pass request along a pipeline" | Chain of Responsibility |
| "Same steps, different details" | Template Method |
| "Traverse without exposing structure" | Iterator |
| "Decouple many-to-many comms" | Mediator |

---

### Quick rule of thumb

- **Creational** → "How do I create this?"
- **Structural** → "How do I connect/wrap this?"
- **Behavioral** → "How do objects talk and share work?"

---

*Last updated: June 2026*
