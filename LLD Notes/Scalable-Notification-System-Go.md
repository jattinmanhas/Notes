# Scalable Notification System in Go: complete LLD walkthrough

Read this from top to bottom: requirements → file tree → each complete source file and its explanation → patterns → failure scenarios → interview practice. No source files or external dependencies are omitted.

## 1. Requirements and boundaries

One request represents one notification to one destination on one channel. Support Email, SMS, and Push; reject invalid requests; check preferences; deduplicate creation; deliver asynchronously; track attempts and terminal status; retry temporary failures with a limit.

Bulk campaigns, templates, attachments, scheduling for a future date, delivery receipts, and provider failover are extensions. Do not silently turn one small LLD exercise into all of these features.

This is a **single-process teaching implementation** with console senders and in-memory storage. Its interfaces show where scalable infrastructure belongs. It does not implement durable delivery or a transactional outbox. A process crash loses its data and queued work. `SENT` means the sender returned success (provider acceptance for a real adapter), not that a human received or read the message.

## 2. Request flow and state

```text
main (composition root)
  -> NotificationService.create
       -> validate request
       -> PreferenceService
       -> Repository.createIfAbsent (atomic idempotency check + insert)
       -> Queue.publish(notification ID)
            -> Dispatcher worker
                 -> atomic claim
                 -> SenderFactory.get(channel)
                 -> Sender.send(notification)
                 -> success / retry / terminal failure
```

```text
PENDING --claim, increment attempts--> PROCESSING --success--> SENT
                                         |
                                         +--temporary error, attempts remain--> PENDING
                                         |                                     (delayed enqueue)
                                         +--permanent error / exhausted limit--> FAILED
```

Creation returns an ID without waiting for delivery. The returned status can still be pending even if a worker finishes immediately afterward. Query the repository for the latest state. A repeated request with the same key returns the same logical notification and does not enqueue again.

## 3. File structure and running the project

Create the files at the paths shown below. Every source block is complete, including its package and imports. The source files are split by responsibility, and the entry point constructs the dependencies explicitly.

```text
notification-go/
  go.mod
  internal/notification/model.go
  internal/notification/sender.go
  internal/notification/repository.go
  internal/notification/queue.go
  internal/notification/preferences.go
  internal/notification/service.go
  internal/notification/retry.go
  internal/notification/dispatcher.go
  cmd/demo/main.go
  internal/notification/notification_test.go
```

Use Go 1.22 or newer. From the notification-go directory:

```bash
go run ./cmd/demo
go test ./...
go test -race ./...
```

`cmd/demo` contains the executable package. `internal/notification` contains the application's domain, use cases and adapters in one cohesive package; Go's internal rule prevents unrelated external modules from importing it. Files in one package share package-private identifiers, so filenames organize responsibilities without introducing an interface/package for every type.

Expected final output:

```text
EMAIL to=learner@example.com subject="Order shipped" body="Your order is on its way."
FINAL id=notification-1 status=sent attempts=1
```

## 4. Every file, code and explanation

### 4.1. `go.mod` — Module boundary

```text
module example.com/notifications

go 1.22
```

**How this file works:** The module path matches the import in cmd/demo/main.go. Go 1.22 or newer is required because the worker startup uses integer range. No third-party dependencies are needed.

### 4.2. `internal/notification/model.go` — Domain vocabulary

```go
package notification

type Channel string

const (
	Email Channel = "email"
	SMS   Channel = "sms"
	Push  Channel = "push"
)

type Status string

const (
	Pending    Status = "pending"
	Processing Status = "processing"
	Sent       Status = "sent"
	Failed     Status = "failed"
)

type Notification struct {
	ID, IdempotencyKey, UserID string
	Channel                    Channel
	Destination, Subject, Body string
	Status                     Status
	Attempts                   int
	LastError                  string
}

type CreateRequest struct {
	IdempotencyKey, UserID     string
	Channel                    Channel
	Destination, Subject, Body string
}
```

**How this file works:** Channel and Status are named string types. Notification holds delivery state while CreateRequest represents caller input. Keeping them separate prevents the API from accepting an arbitrary sent status or attempt count. These structs are values: the memory repository returns copies. It protects mutations under its own lock. In a larger domain, unexport fields or use transition methods when you need to enforce invariants outside that repository.

### 4.3. `internal/notification/sender.go` — Strategy interface and registry

```go
package notification

import (
	"context"
	"fmt"
)

type Sender interface {
	Send(context.Context, Notification) error
}

type EmailSender struct{}

func (EmailSender) Send(_ context.Context, n Notification) error {
	fmt.Printf("EMAIL to=%s subject=%q body=%q\n", n.Destination, n.Subject, n.Body)
	return nil
}

type SMSSender struct{}

func (SMSSender) Send(_ context.Context, n Notification) error {
	fmt.Printf("SMS to=%s body=%q\n", n.Destination, n.Body)
	return nil
}

type PushSender struct{}

func (PushSender) Send(_ context.Context, n Notification) error {
	fmt.Printf("PUSH token=%s body=%q\n", n.Destination, n.Body)
	return nil
}

type SenderFactory struct{ senders map[Channel]Sender }

func NewSenderFactory(s map[Channel]Sender) *SenderFactory {
	copy := make(map[Channel]Sender, len(s))
	for channel, sender := range s {
		copy[channel] = sender
	}
	return &SenderFactory{senders: copy}
}
func (f *SenderFactory) Get(channel Channel) (Sender, error) {
	sender, ok := f.senders[channel]
	if !ok {
		return nil, fmt.Errorf("unsupported channel %q", channel)
	}
	return sender, nil
}

// DeliveryError distinguishes retryable provider failures from permanent ones.
type DeliveryError struct {
	Message   string
	Temporary bool
}

func (e *DeliveryError) Error() string { return e.Message }
```

**How this file works:** Sender is defined next to its consumers in the application package. EmailSender, SMSSender and PushSender satisfy it implicitly; no implements keyword is necessary. SenderFactory copies registrations so later mutation of the caller map does not change dispatch behavior. Configure it once before starting workers. DeliveryError carries retryability; a vendor adapter should translate provider responses into this error rather than forcing the dispatcher to parse error strings. Console implementations succeed immediately; they do not contact providers.

### 4.4. `internal/notification/repository.go` — Repository and atomic creation/claim

```go
package notification

import (
	"sync"
)

type Repository interface {
	CreateIfAbsent(Notification) (Notification, bool)
	Claim(string) (Notification, bool)
	MarkSent(string)
	MarkRetry(string, string)
	MarkFailed(string, string)
	Get(string) (Notification, bool)
}

type MemoryRepository struct {
	mu    sync.Mutex
	items map[string]Notification
	byKey map[string]string
}

func NewMemoryRepository() *MemoryRepository {
	return &MemoryRepository{items: map[string]Notification{}, byKey: map[string]string{}}
}

func (r *MemoryRepository) CreateIfAbsent(n Notification) (Notification, bool) {
	r.mu.Lock()
	defer r.mu.Unlock()
	if id, exists := r.byKey[n.IdempotencyKey]; exists {
		return r.items[id], false
	}
	r.items[n.ID], r.byKey[n.IdempotencyKey] = n, n.ID
	return n, true
}

func (r *MemoryRepository) Claim(id string) (Notification, bool) {
	r.mu.Lock()
	defer r.mu.Unlock()
	n, exists := r.items[id]
	if !exists || n.Status != Pending {
		return n, false
	}
	n.Status, n.Attempts = Processing, n.Attempts+1
	r.items[id] = n
	return n, true
}

func (r *MemoryRepository) update(id string, status Status, message string) {
	r.mu.Lock()
	defer r.mu.Unlock()
	n := r.items[id]
	n.Status, n.LastError = status, message
	r.items[id] = n
}

func (r *MemoryRepository) MarkSent(id string)            { r.update(id, Sent, "") }
func (r *MemoryRepository) MarkRetry(id, message string)  { r.update(id, Pending, message) }
func (r *MemoryRepository) MarkFailed(id, message string) { r.update(id, Failed, message) }
func (r *MemoryRepository) Get(id string) (Notification, bool) {
	r.mu.Lock()
	defer r.mu.Unlock()
	n, ok := r.items[id]
	return n, ok
}
```

**How this file works:** CreateIfAbsent locks both maps together, so checking the key and saving cannot race. Claim changes pending to processing and increments attempts within one critical section. Returning value copies avoids exposing a mutable map entry. MarkSent/MarkRetry/MarkFailed are internal completion operations invoked after claiming a known ID; this small adapter assumes the ID exists and does not implement lease tokens or legal-transition validation. A production database repository needs conditional updates and explicit errors, not unconditional overwrites.

### 4.5. `internal/notification/queue.go` — Producer–consumer transport

```go
package notification

import (
	"context"
	"time"
)

type Queue interface {
	Publish(context.Context, string, time.Duration) error
	Messages() <-chan string
}

type MemoryQueue struct{ messages chan string }

func NewMemoryQueue(capacity int) *MemoryQueue {
	return &MemoryQueue{messages: make(chan string, capacity)}
}
func (q *MemoryQueue) Messages() <-chan string { return q.messages }
func (q *MemoryQueue) Publish(ctx context.Context, id string, delay time.Duration) error {
	if err := ctx.Err(); err != nil {
		return err
	}
	go func() {
		timer := time.NewTimer(delay)
		defer timer.Stop()
		select {
		case <-ctx.Done():
			return
		case <-timer.C:
		}
		select {
		case <-ctx.Done():
			return
		case q.messages <- id:
		}
	}()
	return nil
}
```

**How this file works:** Only IDs enter the channel, keeping the stored notification authoritative. Publish schedules a timer goroutine and returns before the ID reaches the channel; successful return means local scheduling only. The channel buffer is bounded but the number of waiting publisher goroutines is not, so this is not system-wide backpressure. Use one application-lifetime context here. An HTTP request context ends after the response and would cancel these pending deliveries. A durable publisher must confirm acceptance independently of the request lifetime.

### 4.6. `internal/notification/preferences.go` — Business-rule boundary

```go
package notification

type PreferenceService interface{ Allowed(string, Channel) bool }
type AllowAll struct{}

func (AllowAll) Allowed(string, Channel) bool { return true }
```

**How this file works:** PreferenceService lets request creation check a user/channel rule without knowing where preferences are stored. AllowAll is an explicit demo adapter. Tests can supply a denying implementation. Transactional and marketing messages can have different preference rules; agree on that requirement before adding conditionals.

### 4.7. `internal/notification/service.go` — Application orchestration and idempotency

```go
package notification

import (
	"context"
	"errors"
	"fmt"
	"sync/atomic"
)

type IDGenerator struct{ value atomic.Uint64 }

func (g *IDGenerator) New() string { return fmt.Sprintf("notification-%d", g.value.Add(1)) }

type NotificationService struct {
	repository  Repository
	queue       Queue
	preferences PreferenceService
	ids         *IDGenerator
}

func NewNotificationService(r Repository, q Queue, p PreferenceService, ids *IDGenerator) *NotificationService {
	return &NotificationService{repository: r, queue: q, preferences: p, ids: ids}
}

func (s *NotificationService) Create(ctx context.Context, req CreateRequest) (Notification, error) {
	if req.IdempotencyKey == "" || req.UserID == "" || req.Destination == "" || req.Body == "" {
		return Notification{}, errors.New("missing required field")
	}
	switch req.Channel {
	case Email, SMS, Push:
	default:
		return Notification{}, errors.New("unsupported channel")
	}
	if err := ctx.Err(); err != nil {
		return Notification{}, err
	}
	if !s.preferences.Allowed(req.UserID, req.Channel) {
		return Notification{}, errors.New("user opted out")
	}
	n := Notification{ID: s.ids.New(), IdempotencyKey: req.IdempotencyKey, UserID: req.UserID,
		Channel: req.Channel, Destination: req.Destination, Subject: req.Subject, Body: req.Body, Status: Pending}
	saved, created := s.repository.CreateIfAbsent(n)
	if created {
		if err := s.queue.Publish(ctx, saved.ID, 0); err != nil {
			s.repository.MarkFailed(saved.ID, err.Error())
			return saved, err
		}
	}
	return saved, nil
}
```

**How this file works:** Create validates channel and required fields, checks cancellation and preferences, constructs a pending record, atomically deduplicates, and schedules only newly created records. It returns a queue error rather than silently discarding it. Sequence IDs use an atomic counter and are unique only within this process lifetime; production needs a shared-safe ID scheme. The save/enqueue gap remains: this is where an outbox transaction would replace two independent operations. Constructor injection makes the collaborators visible and testable.

### 4.8. `internal/notification/retry.go` — Retry policy value

```go
package notification

import (
	"time"
)

type RetryPolicy struct {
	MaxAttempts int
	BaseDelay   time.Duration
}

func (p RetryPolicy) Next(attempt int) (time.Duration, bool) {
	if attempt < 1 || attempt >= p.MaxAttempts {
		return 0, false
	}
	delay := p.BaseDelay
	if delay <= 0 {
		delay = 100 * time.Millisecond
	}
	const capDelay = 30 * time.Second
	for i := 1; i < attempt && delay < capDelay; i++ {
		if delay >= capDelay/2 {
			return capDelay, true
		}
		delay *= 2
	}
	if delay > capDelay {
		delay = capDelay
	}
	return delay, true
}
```

**How this file works:** Next returns a delay and whether another attempt is allowed. Attempts counts sends, beginning at one after claim. Backoff doubles up to 30 seconds. This is a concrete policy value, not a Strategy hierarchy; adding an interface later makes sense if different categories require interchangeable algorithms. Only temporary DeliveryError values reach this policy.

### 4.9. `internal/notification/dispatcher.go` — Worker pool and delivery algorithm

```go
package notification

import (
	"context"
	"errors"
	"sync"
	"time"
)

type Dispatcher struct {
	repository Repository
	queue      Queue
	factory    *SenderFactory
	retry      RetryPolicy
	workers    int
}

func (d *Dispatcher) Run(ctx context.Context) {
	var wg sync.WaitGroup
	for range d.workers {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for {
				select {
				case <-ctx.Done():
					return
				case id := <-d.queue.Messages():
					d.process(ctx, id)
				}
			}
		}()
	}
	<-ctx.Done()
	wg.Wait()
}

func (d *Dispatcher) process(ctx context.Context, id string) {
	n, claimed := d.repository.Claim(id)
	if !claimed {
		return
	}
	sender, err := d.factory.Get(n.Channel)
	if err != nil {
		d.repository.MarkFailed(id, err.Error())
		return
	}
	sendCtx, cancel := context.WithTimeout(ctx, 2*time.Second)
	err = sender.Send(sendCtx, n)
	cancel()
	if err == nil {
		d.repository.MarkSent(id)
		return
	}
	var deliveryError *DeliveryError
	if !errors.As(err, &deliveryError) || !deliveryError.Temporary {
		d.repository.MarkFailed(id, err.Error())
		return
	}
	if delay, retry := d.retry.Next(n.Attempts); retry {
		d.repository.MarkRetry(id, err.Error())
		if err := d.queue.Publish(ctx, id, delay); err != nil {
			d.repository.MarkFailed(id, err.Error())
		}
		return
	}
	d.repository.MarkFailed(id, err.Error())
}

func NewDispatcher(r Repository, q Queue, f *SenderFactory, retry RetryPolicy, workers int) *Dispatcher {
	if workers < 1 {
		panic("workers must be positive")
	}
	return &Dispatcher{repository: r, queue: q, factory: f, retry: retry, workers: workers}
}
```

**How this file works:** NewDispatcher rejects zero workers. Run starts the fixed pool and waits for cancellation and worker exit. process claims, selects, sends with a two-second context deadline, classifies errors and records the result. The deadline is cooperative: a sender must honor context and configure its network client correctly. Unknown errors are terminal by default. A temporary error is re-enqueued after changing state to pending; the durable retry and early-duplicate gaps are explained later. The queue is never closed in this adapter; production consumers should handle channel closure explicitly.

### 4.10. `cmd/demo/main.go` — Composition root and executable demonstration

```go
package main

import (
	"context"
	"example.com/notifications/internal/notification"
	"fmt"
	"time"
)

func main() {
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()
	repository := notification.NewMemoryRepository()
	queue := notification.NewMemoryQueue(100)
	factory := notification.NewSenderFactory(map[notification.Channel]notification.Sender{notification.Email: notification.EmailSender{}, notification.SMS: notification.SMSSender{}, notification.Push: notification.PushSender{}})
	service := notification.NewNotificationService(repository, queue, notification.AllowAll{}, &notification.IDGenerator{})
	dispatcher := notification.NewDispatcher(repository, queue, factory, notification.RetryPolicy{MaxAttempts: 3, BaseDelay: 100 * time.Millisecond}, 4)
	done := make(chan struct{})
	go func() { defer close(done); dispatcher.Run(ctx) }()
	defer func() { cancel(); <-done }()

	n, err := service.Create(ctx, notification.CreateRequest{IdempotencyKey: "order-42-shipped", UserID: "user-7",
		Channel: notification.Email, Destination: "learner@example.com", Subject: "Order shipped", Body: "Your order is on its way."})
	if err != nil {
		panic(err)
	}
	deadline := time.Now().Add(3 * time.Second)
	for {
		current, _ := repository.Get(n.ID)
		if current.Status == notification.Sent || current.Status == notification.Failed {
			break
		}
		if time.Now().After(deadline) {
			panic("delivery timed out")
		}
		time.Sleep(time.Millisecond)
	}
	result, _ := repository.Get(n.ID)
	fmt.Printf("FINAL id=%s status=%s attempts=%d\n", result.ID, result.Status, result.Attempts)
}
```

**How this file works:** main constructs concrete adapters and injects them into the service and dispatcher. It starts the workers before submitting one request. Polling waits for a terminal state with a deadline, avoiding an arbitrary sleep followed by a possibly pending result. Shutdown cancels and joins workers. Running the entire command package is necessary because types live in another package. Production HTTP handlers return an ID; they do not poll until sending completes.

### 4.11. `internal/notification/notification_test.go` — Behavior tests through substitutes

```go
package notification

import (
	"context"
	"testing"
	"time"
)

type recordingQueue struct {
	count int
	delay time.Duration
}

func (q *recordingQueue) Publish(_ context.Context, _ string, delay time.Duration) error {
	q.count++
	q.delay = delay
	return nil
}
func (*recordingQueue) Messages() <-chan string { return nil }

type controlledSender struct {
	calls     int
	temporary bool
	fail      bool
}

func (s *controlledSender) Send(context.Context, Notification) error {
	s.calls++
	if s.fail {
		return &DeliveryError{Message: "provider failure", Temporary: s.temporary}
	}
	return nil
}

func TestIdempotencyAndDelivery(t *testing.T) {
	r, q := NewMemoryRepository(), &recordingQueue{}
	svc := NewNotificationService(r, q, AllowAll{}, &IDGenerator{})
	req := CreateRequest{IdempotencyKey: "k", UserID: "u", Channel: Email, Destination: "a@b", Body: "hi"}
	a, err := svc.Create(context.Background(), req)
	if err != nil {
		t.Fatal(err)
	}
	b, err := svc.Create(context.Background(), req)
	if err != nil || a.ID != b.ID || q.count != 1 {
		t.Fatal("duplicate request enqueued twice")
	}
	sender := &controlledSender{}
	d := NewDispatcher(r, q, NewSenderFactory(map[Channel]Sender{Email: sender}), RetryPolicy{MaxAttempts: 3, BaseDelay: time.Millisecond}, 1)
	d.process(context.Background(), a.ID)
	d.process(context.Background(), a.ID)
	n, _ := r.Get(a.ID)
	if sender.calls != 1 || n.Status != Sent {
		t.Fatal("duplicate delivery processed")
	}
}

func TestRetryAndPermanentFailure(t *testing.T) {
	for _, temporary := range []bool{true, false} {
		r, q := NewMemoryRepository(), &recordingQueue{}
		r.CreateIfAbsent(Notification{ID: "n", IdempotencyKey: "k", Channel: Email, Status: Pending})
		sender := &controlledSender{fail: true, temporary: temporary}
		d := NewDispatcher(r, q, NewSenderFactory(map[Channel]Sender{Email: sender}), RetryPolicy{MaxAttempts: 2, BaseDelay: time.Millisecond}, 1)
		d.process(context.Background(), "n")
		n, _ := r.Get("n")
		if temporary {
			if n.Status != Pending || q.count != 1 || q.delay != time.Millisecond {
				t.Fatal("retry not scheduled")
			}
			d.process(context.Background(), "n")
		}
		n, _ = r.Get("n")
		if n.Status != Failed {
			t.Fatal("expected terminal failure")
		}
	}
}
```

**How this file works:** A recording queue and controlled sender replace infrastructure. Tests assert repeated keys enqueue once, duplicate deliveries cannot reprocess sent records, temporary failures schedule retry, and permanent/exhausted failures terminate. Direct processing tests are deterministic and avoid timer races. They do not prove distributed durability, which the adapter does not provide.

## Design patterns: exact locations and reasons

| Pattern or principle | Exact participant | Why it is used |
|---|---|---|
| Strategy | Sender interface and Email/SMS/Push implementations | Dispatcher invokes the same operation while channel behavior varies. |
| Registry | SenderFactory and its channel-to-sender map | Looks up an already constructed strategy. Despite its name, it is not the GoF Factory Method or Abstract Factory pattern. |
| Repository | Repository interface and in-memory implementation | Hides storage details and owns creation/idempotency behavior. |
| Dependency injection | main supplies constructor arguments | Dependencies are explicit and tests can replace them with fakes; no DI framework is required. |
| Producer–consumer | Service publishes, Dispatcher consumes | Separates request latency from provider latency. |
| Worker pool | Dispatcher | Bounds simultaneous provider calls within one process. It does not bound the whole backlog. |
| Idempotent creation | Atomic createIfAbsent | Concurrent repeated keys map to one notification. This is a reliability technique, not a GoF pattern. |
| Composition | Service and Dispatcher contain collaborators | Vary behavior by supplying another implementation. Inheritance is unnecessary. |

An interface alone is not a design pattern. An enum with state transitions is not automatically the GoF State pattern. These examples do not use Observer: workers compete for queued messages; they do not broadcast every event to all observers. Console senders demonstrate Strategy; a real vendor wrapper would additionally be an Adapter.

## How the design follows SOLID

**Single responsibility:** the service orchestrates creation, the repository manages storage, the queue transports IDs, the dispatcher delivers, and the sender handles its channel. Keeping these separate prevents provider changes from affecting persistence logic.

**Open/closed:** adding a channel introduces a sender and registration. You must also extend the channel enum/constants and request validation. The delivery algorithm stays unchanged; “open/closed” does not mean no existing file can ever change.

**Liskov substitution:** every sender must respect its contract: success means acceptance, failures communicate retryability, and real network adapters need bounded execution. An implementation that reports success before it even attempts delivery violates this expectation.

**Interface segregation:** clients depend on small operation groups such as preferences or sending. For a larger application, split the repository into narrower read/write/worker ports when consumers actually benefit; avoid an interface for every tiny class solely to claim SOLID.

**Dependency inversion:** orchestration depends on repository, queue, preferences, and sender abstractions. The composition root chooses in-memory implementations. RetryPolicy remains a concrete value/class here because there is only one policy; introduce a policy interface when you actually need interchangeable policies.

## Trace an example through the files

1. `main` creates shared storage, queue, sender registry, service, and dispatcher, then starts workers.
2. It submits `order-42-shipped`, channel Email, destination `learner@example.com`.
3. Service validates and checks preferences. Repository atomically looks up the key and either returns its existing record or inserts a new pending notification.
4. Only a newly created record is queued. The payload is its ID; the notification remains in the repository.
5. A worker claims the record. Attempts becomes 1 and status becomes processing.
6. Registry selects EmailSender. The demo prints its destination, subject, and body.
7. Dispatcher records sent. `main` polls with a deadline and prints the final state, then stops the workers.

For a temporary provider error with maxAttempts=3 and baseDelay=100ms, attempts occur immediately, after 100ms, then after 200ms. The third failure is terminal. There are three total attempts, not three retries. Permanent errors fail immediately. The backoff has a 30-second cap to prevent arithmetic growth from becoming unbounded; jitter is a production extension.

## Reliability gaps you must be able to explain

| Concrete failure | What this sample does | Production change |
|---|---|---|
| Process dies after save but before enqueue | Record can remain pending without a job; restart loses all in-memory state | In one DB transaction, insert notification and outbox row. Relay publishes committed rows and records publication. |
| Process dies while a worker owns a record | No lease recovery; work is lost | Store lease expiry and an ownership token. Reclaim expired leases; reject writes from stale owners. |
| Provider accepts, worker dies before saving success | A durable retry could send again | Pass notification ID as provider idempotency key if supported; otherwise duplicates remain possible. |
| Retry is stored but publishing retry fails | Delivery may be marked failed; no durable recovery | Transactionally persist retry due time/outbox; use broker ack/nack and redelivery. |
| Incoming rate exceeds delivery rate | Backlog grows in memory | Bound admission and broker backlog; use quotas and overload responses. |
| A duplicate message arrives during retry backoff | Pending record can be claimed early | Persist nextAttemptAt and check it atomically during claim. |
| Same key is reused with different contents | Existing record wins silently | Compare a normalized request fingerprint and return a conflict. Scope key by tenant. |
| User opts out after enqueue | Preference checked only at creation | Recheck appropriate preferences before sending, depending on message category. |

The sample does **not** provide an at-least-once delivery guarantee. A durable broker, acknowledgements, retry persistence, and lease recovery are prerequisites. With those additions the usual target is at least once, with idempotent effects where possible. A mutex or synchronized block only coordinates threads in one process, never multiple application servers.

### Transactional outbox extension, step by step

1. Begin a database transaction; enforce a unique `(tenant_id, idempotency_key)` constraint.
2. Insert the notification and an outbox event containing its ID in that same transaction; commit.
3. A relay claims an outbox batch, publishes to a durable broker, and marks the event published after broker confirmation.
4. A crash between publishing and marking can publish twice. Consumers must therefore tolerate duplicates.
5. Workers use atomic conditional state updates plus leases and acknowledge only after recording the result or a durable retry.

These steps describe an extension, not hidden functionality in the code below. The queue port would need delivery envelopes, acknowledgements and error handling; the repository port would need transactions, persistent leases and storage errors. Replacing an in-memory class with Kafka or SQL without changing these contracts is not enough.

### Scaling decisions

Use separate queues and worker allocations for Email, SMS, and Push when one provider's slowness should not block another. Multiple worker replicas consume durable queues; a shared database coordinates claims. Provider quotas often require a shared rate limiter, because a limit per process multiplies when more replicas start. Add send timeouts, a dead-letter destination, and metrics for backlog age, acceptance latency, attempts, failure categories and provider response codes.

A rough concurrency estimate is target throughput multiplied by average send latency: 200 sends/sec × 0.25 sec = 50 simultaneous sends, before headroom. This is an illustration, not a benchmark. Provider quotas and database/broker capacity constrain the final number. Avoid logging message bodies, addresses and push tokens in production; console output here exists to make the interview demo observable.

## How to present this in an LLD interview

Start by asking whether the interviewer wants an in-memory implementation or durable distributed delivery. State the channels, one-recipient scope and asynchronous semantics. Draw the flow, identify the domain object, and write the sender interface first. Implement one sender, registration, service and repository, then show `main` successfully sending. Add worker ownership, retryability and duplicate handling as time allows. Explain the outbox and broker changes when scalability is discussed.

Do not spend the whole interview writing infrastructure boilerplate. Be able to explain why each boundary exists and demonstrate a passing end-to-end run. These files form a reference to study; a 45-minute interview may require a smaller subset agreed with the interviewer.

## Suggested exercises after reading

1. Add a WhatsApp channel and identify every required edit.
2. Replace allow-all preferences with per-user channel preferences and test opt-out.
3. Make a sender fail twice temporarily, then succeed; assert three attempts and one final success.
4. Send concurrently with the same idempotency key; assert one record and one initial enqueue.
5. Add request fingerprint conflicts for reused keys.
6. Design the durable outbox repository contract and worker lease contract before implementing a database adapter.
