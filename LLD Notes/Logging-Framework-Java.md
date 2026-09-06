# Logging Framework in Java — complete LLD walkthrough

Study order: requirements → model and flow → project tree → each complete file and explanation → patterns → walkthrough → limitations and exercises. This is a runnable single-process interview reference, with no external services or omitted source files.

## 1. Requirements and scope

Build a logger with severity filtering, a replaceable formatter and multiple output sinks. Suppressed messages must not reach any sink. Format each accepted event once, attempt every sink even if one fails, and report collected failures to the caller. Use an injected clock for repeatable output. Configure the logger once; dynamic configuration, async buffering, file rotation and structured context inheritance are extensions.

## 2. Design and responsibilities

```text
Main -> Logger.Log(level,message)
       -> threshold check
       -> Event(timestamp,level,message)
       -> Formatter -> one rendered line
       -> Sink 1
       -> Sink 2
       -> collected errors
```

## 3. Project structure and running

Create each file at the shown relative path. All imports and package declarations are included.

```text
logging-framework-java/
  src/study/Level.java
  src/study/Event.java
  src/study/Formatter.java
  src/study/TextFormatter.java
  src/study/Sink.java
  src/study/WriterSink.java
  src/study/Logger.java
  src/study/Main.java
  src/study/BehaviorTest.java
```

JDK 17 or newer; no Maven, Gradle or external libraries required. From the project root:

```bash
javac --release 17 -d out src/study/*.java
java -cp out study.Main
java -cp out study.BehaviorTest
```

Each top-level type has its own file. `Main` and `BehaviorTest` are public entry points. Other types are package-private in `study`; private fields protect state, while package methods expose the intended operations. A larger repository can split packages and deliberately make selected APIs public. Tests throw AssertionError explicitly and do not need `-ea`.

## 4. Every file, its code, and explanation

### 4.1. `src/study/Level.java`

```java
package study;

enum Level {
    DEBUG, INFO, WARN, ERROR
}
```

**How it works and why it belongs here:** Declaration order defines severity ordering.

### 4.2. `src/study/Event.java`

```java
package study;

import java.time.Instant;

record Event(Instant at, Level level, String message) {
}
```

**How it works and why it belongs here:** Immutable event transports data between orchestration and formatting.

### 4.3. `src/study/Formatter.java`

```java
package study;

interface Formatter {
    String format(Event event);
}
```

**How it works and why it belongs here:** Strategy contract isolates representation.

### 4.4. `src/study/TextFormatter.java`

```java
package study;

final class TextFormatter implements Formatter {

    public String format(Event e) {
        return e.at()+" ["+e.level()+"] "+e.message();
    }
}
```

**How it works and why it belongs here:** Instant renders UTC. More elaborate formats can replace this class without changing Logger.

### 4.5. `src/study/Sink.java`

```java
package study;

import java.io.IOException;

interface Sink {
    void write(String line) throws IOException;
}
```

**How it works and why it belongs here:** Expected output failures use a checked exception so orchestration handles them explicitly.

### 4.6. `src/study/WriterSink.java`

```java
package study;

import java.io.PrintStream;
import java.io.IOException;
import java.util.Objects;

final class WriterSink implements Sink {
    private final PrintStream out;
    WriterSink(PrintStream out) {
        this.out = Objects.requireNonNull(out);
    }

    public void write(String line) throws IOException {
        out.println(line);
        if (out.checkError()) throw new IOException("stream write failed");
    }
}
```

**How it works and why it belongs here:** PrintStream suppresses IOExceptions internally, so the adapter checks its error flag. It does not close a caller-owned stream.

### 4.7. `src/study/Logger.java`

```java
package study;

import java.util.List;
import java.util.ArrayList;
import java.util.Objects;
import java.time.Instant;
import java.util.function.Supplier;

final class Logger {
    private final Level threshold;
    private final Formatter formatter;
    private final List<Sink> sinks;
    private final Supplier<Instant> clock;
    Logger(Level threshold, Formatter formatter, List<Sink> sinks, Supplier<Instant> clock) {
        this.threshold = Objects.requireNonNull(threshold);
        this.formatter = Objects.requireNonNull(formatter);
        this.sinks = List.copyOf(sinks);
        this.clock = Objects.requireNonNull(clock);
    }

    synchronized List<Exception> log(Level level, String message) {
        Objects.requireNonNull(level);
        if (level.ordinal()<threshold.ordinal()) return List.of();
        String line = formatter.format(new Event(clock.get(), level, message));
        var failures = new ArrayList<Exception>();
        for (var sink:sinks)try {
            sink.write(line);
        }
        catch (Exception e) {
            failures.add(e);
        }
        return List.copyOf(failures);
    }
}
```

**How it works and why it belongs here:** A synchronized call serializes formatting and sink delivery. A failed sink does not suppress subsequent sinks. Formatter exceptions propagate because no valid rendered event exists to fan out. Serious JVM Errors are not treated as ordinary sink failures.

### 4.8. `src/study/Main.java`

```java
package study;

public class Main {

    public static void main(String[]args) {
        var l = new Logger(Level.INFO, new TextFormatter(), java.util.List.of(new WriterSink(System.out)), () -> java.time.Instant.EPOCH);
        l.log(Level.DEBUG, "hidden");
        if (!l.log(Level.INFO, "order accepted").isEmpty()) throw new IllegalStateException("logging failed");
    }
}
```

**How it works and why it belongs here:** The composition root creates the collaborators explicitly and runs the example. The public entry point lives in its own file; domain collaborators remain package-private within study.

### 4.9. `src/study/BehaviorTest.java`

```java
package study;

public class BehaviorTest {

    static void check(boolean value, String message) {
        if (!value) throw new AssertionError(message);
    }

    public static void main(String[] args) throws Exception {
        var lines = new java.util.ArrayList<String>();
        Sink broken = line -> {
            throw new java.io.IOException("failed");
        }
        ;
        Sink capture = lines::add;
        var l = new Logger(Level.INFO, new TextFormatter(), java.util.List.of(broken, capture), () -> java.time.Instant.EPOCH);
        check(l.log(Level.DEBUG, "hidden").isEmpty() && lines.isEmpty(), "filter");
        check(l.log(Level.INFO, "hello").size() == 1, "failure report");
        check(lines.size() == 1 && lines.get(0).contains("[INFO] hello"), "fanout continues");
        System.out.println("All behavior tests passed");
    }
}
```

**How it works and why it belongs here:** Tests severity suppression and later-sink delivery after an earlier sink fails.

## 5. Patterns and principles: where and why

| Pattern or principle | Concrete location | Reason |
|---|---|---|
| Strategy | Formatter / TextFormatter | Output representation changes independently of event routing. |
| Adapter | WriterSink wrapping io.Writer / PrintStream | Adapts a standard output API to the logger sink contract. |
| Fan-out composition | Logger sink list | One accepted event is offered to every configured sink. |
| Dependency injection | formatter, sinks and clock in constructor | Tests can capture output and simulate failures without touching files. |

A mutex or synchronized method is a concurrency mechanism, not a GoF design pattern. An enum is a state representation, not automatically the State pattern. Interfaces are justified by interchangeable behavior or a useful boundary; inheritance is not required to demonstrate OOP.

### Applying SOLID without unnecessary abstractions

**Single responsibility:** the main entry point assembles dependencies; the domain service owns state invariants; policy collaborators own the behavior named in the pattern table. The tests exercise behavior through the public operations rather than depending on implementation maps.

**Open/closed:** inspect `Formatter / TextFormatter` as the primary variation point. Where a policy interface exists, supply a new implementation without changing the state-transition algorithm. Where this example implements a specific data structure, do not claim its algorithm is interchangeable until you deliberately extract that boundary.

**Liskov substitution:** a replacement collaborator must preserve the documented contract, including invalid-input behavior, clock units, ownership rules and callback failure behavior. Merely matching a method signature is not enough.

**Interface segregation and dependency inversion:** interfaces expose the small question the caller needs answered. Constructors receive policies and clocks where tests or alternative behavior benefit. Simple records, enum values and internal containers remain concrete. A dedicated repository interface is useful when persistence is in scope; these samples do not pretend in-memory mutations automatically translate to database transactions.

## 6. Follow the example and the invariants

Logger threshold is INFO. A DEBUG call returns with no output. An INFO call creates one timestamped event, formats it, then passes the same line to each sink. If sink 1 fails, sink 2 is still attempted. The result reports failures rather than throwing away an entire event after the first error. The demo fixes the timestamp at the Unix epoch and prints one INFO line. Sink success means the sink accepted the write, not that it durably reached disk.

### Verified execution

The complete project above was compiled and its behavior tests executed. The following is actual validation output (paths, timing and identifiers can vary):

```text
1970-01-01T00:00:00Z [INFO] order accepted

All behavior tests passed
```

## 7. Best practices, edge cases and production extensions

The logger serializes accepted calls to keep sink output coherent in this instance; order is lock acquisition order, not a universal chronological order. Sink and formatter callbacks must not recursively call the same logger. Slow I/O delays callers and other logging threads. Separate logger instances sharing a sink need a sink that is independently safe. The console adapter does not own stdout and never closes it.

Asynchronous logging needs a bounded queue and an explicit overload policy: block, drop low-severity entries or fail. Shutdown must drain or explicitly discard pending entries. Rotation and durable flushing belong to a file sink. Structured JSON formatting should escape fields using a serializer, not hand-built concatenation. Never add credentials or tokens to example log fields. Severity filtering is not the Chain of Responsibility pattern here: all selected sinks receive the event instead of one handler claiming it. No Singleton is needed; global mutable loggers complicate isolation and tests.

## 8. Presenting this in an interview

Start by agreeing on the scope in section 1. Draw the responsibility flow, identify the state that must remain consistent, and name the operation that owns that invariant. Implement the core model and service, then wire the collaborators in main and run a concrete example. Show at least one rejected or boundary case from the tests. Explain the pattern at the point where it solves a problem, rather than starting with a list of pattern names.

For a distributed follow-up, distinguish thread safety inside this process from coordination across replicas. In-memory objects do not survive restarts. Agree on consistency, failure recovery and storage requirements before replacing them with remote infrastructure.

## 9. Practice next

1. Add a JSON formatter using the standard serializer.
2. Add per-sink thresholds without duplicating event construction.
3. Implement a bounded asynchronous wrapper and test shutdown/overflow.
4. Add request context fields as immutable event data.
