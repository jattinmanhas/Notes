# Logging Framework in Go — complete LLD walkthrough

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
logging-framework-go/
  go.mod
  internal/lld/event.go
  internal/lld/formatter.go
  internal/lld/sink.go
  internal/lld/logger.go
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
module example.com/logging-framework

go 1.22
```

**How it works and why it belongs here:** The module path matches main’s import. Go 1.22 or newer is sufficient; the implementation uses only the standard library.

### 4.2. `internal/lld/event.go`

```go
package lld

import (
	"time"
)

type Level int

const (
	Debug Level = iota
	Info
	Warn
	Error
)

func (l Level) String() string {
	switch l {
	case Debug:
		return "DEBUG"
	case Info:
		return "INFO"
	case Warn:
		return "WARN"
	case Error:
		return "ERROR"
	}
	return "UNKNOWN"
}

type Event struct {
	At      time.Time
	Level   Level
	Message string
}
```

**How it works and why it belongs here:** Level ordering enables threshold comparison. Event is a value passed to the formatter; no mutable global context is involved.

### 4.3. `internal/lld/formatter.go`

```go
package lld

import (
	"fmt"
	"time"
)

type Formatter interface{ Format(Event) string }
type TextFormatter struct{}

func (TextFormatter) Format(e Event) string {
	return fmt.Sprintf("%s [%s] %s", e.At.UTC().Format(time.RFC3339), e.Level, e.Message)
}
```

**How it works and why it belongs here:** Formatter is a Strategy with one responsibility: representation. UTC timestamps keep demo output consistent.

### 4.4. `internal/lld/sink.go`

```go
package lld

import (
	"fmt"
	"io"
)

type Sink interface{ Write(string) error }
type WriterSink struct{ Writer io.Writer }

func (s WriterSink) Write(line string) error { _, err := fmt.Fprintln(s.Writer, line); return err }
```

**How it works and why it belongs here:** WriterSink adapts any io.Writer, including stdout or bytes.Buffer. It propagates write errors; ownership and close/flush remain with the caller.

### 4.5. `internal/lld/logger.go`

```go
package lld

import (
	"fmt"
	"sync"
	"time"
)

type Logger struct {
	mu        sync.Mutex
	threshold Level
	formatter Formatter
	sinks     []Sink
	now       func() time.Time
}

func NewLogger(threshold Level, f Formatter, sinks []Sink, now func() time.Time) *Logger {
	if threshold < Debug || threshold > Error || f == nil || now == nil {
		panic("invalid configuration")
	}
	for _, s := range sinks {
		if s == nil {
			panic("nil sink")
		}
	}
	return &Logger{threshold: threshold, formatter: f, sinks: append([]Sink(nil), sinks...), now: now}
}
func (l *Logger) Log(level Level, message string) []error {
	if level < Debug || level > Error {
		return []error{fmt.Errorf("invalid level")}
	}
	if level < l.threshold {
		return nil
	}
	l.mu.Lock()
	defer l.mu.Unlock()
	line := l.formatter.Format(Event{l.now(), level, message})
	var failures []error
	for i, s := range l.sinks {
		if err := s.Write(line); err != nil {
			failures = append(failures, fmt.Errorf("sink %d: %w", i, err))
		}
	}
	return failures
}
```

**How it works and why it belongs here:** The logger copies its sink slice, filters before creating an event, formats once and accumulates failures. Threshold and registrations are immutable after construction, making reads outside the lock safe. Callback panics are programming failures, not recovered log-delivery errors.

### 4.6. `cmd/demo/main.go`

```go
package main

import (
	"example.com/logging-framework/internal/lld"
	"os"
	"time"
)

func main() {
	l := lld.NewLogger(lld.Info, lld.TextFormatter{}, []lld.Sink{lld.WriterSink{Writer: os.Stdout}}, func() time.Time { return time.Unix(0, 0) })
	l.Log(lld.Debug, "hidden")
	if errs := l.Log(lld.Info, "order accepted"); len(errs) > 0 {
		panic(errs[0])
	}
}
```

**How it works and why it belongs here:** The composition root constructs dependencies, runs a concrete scenario, and prints observable results. Error checks make a rejected operation visible rather than silently treating it as success.

### 4.7. `internal/lld/service_test.go`

```go
package lld

import (
	"bytes"
	"fmt"
	"strings"
	"testing"
	"time"
)

type badSink struct{}

func (badSink) Write(string) error { return fmt.Errorf("failure") }
func TestLogging(t *testing.T) {
	var b bytes.Buffer
	l := NewLogger(Info, TextFormatter{}, []Sink{badSink{}, WriterSink{&b}}, func() time.Time { return time.Unix(0, 0) })
	if len(l.Log(Debug, "hidden")) != 0 || b.Len() != 0 {
		t.Fatal("filter")
	}
	if len(l.Log(Info, "hello")) != 1 {
		t.Fatal("missing error")
	}
	if !strings.Contains(b.String(), "[INFO] hello") {
		t.Fatal("second sink skipped")
	}
}
```

**How it works and why it belongs here:** A failing first sink and buffer second sink prove failure isolation. Filtering is checked before delivery.

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
?   	example.com/logging-framework/cmd/demo	[no test files]
ok  	example.com/logging-framework/internal/lld	(cached)

1970-01-01T00:00:00Z [INFO] order accepted
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
