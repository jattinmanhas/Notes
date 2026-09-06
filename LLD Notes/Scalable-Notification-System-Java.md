# Scalable Notification System in Java: complete LLD walkthrough

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
notification-java/
  src/notification/Channel.java
  src/notification/Status.java
  src/notification/Notification.java
  src/notification/CreateRequest.java
  src/notification/NotificationSender.java
  src/notification/EmailSender.java
  src/notification/SmsSender.java
  src/notification/PushSender.java
  src/notification/ProviderException.java
  src/notification/SenderFactory.java
  src/notification/NotificationRepository.java
  src/notification/SaveResult.java
  src/notification/InMemoryNotificationRepository.java
  src/notification/MessageQueue.java
  src/notification/InMemoryMessageQueue.java
  src/notification/PreferenceService.java
  src/notification/AllowAllPreferences.java
  src/notification/NotificationService.java
  src/notification/RetryPolicy.java
  src/notification/Dispatcher.java
  src/notification/Main.java
  src/notification/NotificationTest.java
```

Use JDK 17 or newer. From the notification-java directory:

```bash
javac --release 17 -d out src/notification/*.java
java -cp out notification.Main
java -cp out notification.NotificationTest
```

Each type has its own file in the notification package. Main and NotificationTest are public entry points; collaborators are package-private to keep the exposed API small. There are no nested application classes. A larger project can split model/service/infrastructure into packages and deliberately expose the necessary types and constructors. Maven or Gradle is optional for this dependency-free exercise.

Expected output has a different UUID each run:

```text
EMAIL to=learner@example.com subject=Order shipped body=Your order is on its way.
FINAL id=<generated UUID> status=SENT attempts=1
```

## 4. Every file, code and explanation

### 4.1. `src/notification/Channel.java` — Channel enum

```java
package notification;


enum Channel { EMAIL, SMS, PUSH }
```

**How this file works:** The enum limits the channel vocabulary and avoids misspelled string comparisons. Add a new value when introducing a new strategy, then register that strategy.

### 4.2. `src/notification/Status.java` — Delivery state enum

```java
package notification;


enum Status { PENDING, PROCESSING, SENT, FAILED }
```

**How this file works:** The lifecycle is explicit. An enum represents states; this is not the GoF State pattern, which would delegate behavior to state objects.

### 4.3. `src/notification/Notification.java` — Encapsulated entity

```java
package notification;


final class Notification {
    private final String id, idempotencyKey, userId, destination, subject, body;
    private final Channel channel;
    private Status status = Status.PENDING;
    private int attempts;
    private String lastError = "";

    Notification(String id, String key, String userId, Channel channel,
                 String destination, String subject, String body) {
        this.id = id; this.idempotencyKey = key; this.userId = userId;
        this.channel = channel; this.destination = destination;
        this.subject = subject; this.body = body;
    }

    synchronized boolean claim() {
        if (status != Status.PENDING) return false;
        status = Status.PROCESSING; attempts++; return true;
    }
    synchronized void markSent() { status = Status.SENT; lastError = ""; }
    synchronized void markRetry(String error) { status = Status.PENDING; lastError = error; }
    synchronized void markFailed(String error) { status = Status.FAILED; lastError = error; }
    String id() { return id; }
    String idempotencyKey() { return idempotencyKey; }
    Channel channel() { return channel; }
    String destination() { return destination; }
    String subject() { return subject; }
    String body() { return body; }
    synchronized Status status() { return status; }
    synchronized int attempts() { return attempts; }
}
```

**How this file works:** Routing and content fields are final. Mutable delivery fields are accessed through synchronized methods so state transitions and visibility are protected. claim atomically refuses non-pending work and increments attempts. Repository returns a live entity reference, so this synchronization matters. Completion methods assume a successful claim and are package-private; a database-backed design should move persistence transitions into conditional repository operations with ownership tokens. Do not treat this live-reference behavior as a SQL persistence contract.

### 4.4. `src/notification/CreateRequest.java` — Immutable input record

```java
package notification;


record CreateRequest(String idempotencyKey, String userId, Channel channel,
                     String destination, String subject, String body) {}
```

**How this file works:** A record generates field accessors, equality and construction boilerplate. Input deliberately excludes status, attempts and internal ID; the service controls those. Validation happens in the service so the rule is applied for every caller.

### 4.5. `src/notification/NotificationSender.java` — Strategy contract

```java
package notification;


interface NotificationSender { void send(Notification n) throws ProviderException; }
```

**How this file works:** Every sender implements send with the same domain input. A checked ProviderException forces the worker to handle expected provider failures. Vendor adapters should configure network timeouts and map vendor failures into retryable or permanent categories.

### 4.6. `src/notification/EmailSender.java` — Email strategy

```java
package notification;


final class EmailSender implements NotificationSender {
    public void send(Notification n) {
        System.out.printf("EMAIL to=%s subject=%s body=%s%n", n.destination(), n.subject(), n.body());
    }
}
```

**How this file works:** This implementation shows the channel-specific subject and body output. Replace its internals with an email provider SDK; Dispatcher continues to depend only on NotificationSender.

### 4.7. `src/notification/SmsSender.java` — SMS strategy

```java
package notification;


final class SmsSender implements NotificationSender {
    public void send(Notification n) {
        System.out.printf("SMS to=%s body=%s%n", n.destination(), n.body());
    }
}
```

**How this file works:** SMS uses destination and body; the interface permits a different implementation without changing dispatch. Real message length, encoding and pricing rules belong in explicit channel validation or the adapter.

### 4.8. `src/notification/PushSender.java` — Push strategy

```java
package notification;


final class PushSender implements NotificationSender {
    public void send(Notification n) {
        System.out.printf("PUSH token=%s body=%s%n", n.destination(), n.body());
    }
}
```

**How this file works:** Push interprets destination as a device token. This keeps the example small; a typed destination model is preferable once recipient validation differs substantially by channel.

### 4.9. `src/notification/ProviderException.java` — Typed delivery failure

```java
package notification;


final class ProviderException extends Exception {
    private final boolean retryable;
    ProviderException(String message, boolean retryable) {
        super(message);
        this.retryable = retryable;
    }
    boolean retryable() { return retryable; }
}
```

**How this file works:** retryable is an explicit boolean so the worker can distinguish temporary unavailability from invalid destinations. Classification belongs in the provider adapter. Unexpected RuntimeException values fail terminally rather than being retried blindly.

### 4.10. `src/notification/SenderFactory.java` — Immutable strategy registry

```java
package notification;

import java.util.Map;

final class SenderFactory {
    private final Map<Channel, NotificationSender> senders;
    SenderFactory(Map<Channel, NotificationSender> senders) { this.senders = Map.copyOf(senders); }
    NotificationSender get(Channel channel) {
        var sender = senders.get(channel);
        if (sender == null) throw new IllegalArgumentException("Unsupported channel: " + channel);
        return sender;
    }
}
```

**How this file works:** Map.copyOf freezes registration and rejects null entries. get looks up an existing sender; it does not construct an object. The conventional name Factory is used here, but the precise pattern is Registry. No inheritance or switch-heavy dispatcher is required.

### 4.11. `src/notification/NotificationRepository.java` — Repository port

```java
package notification;

import java.util.Optional;

interface NotificationRepository {
    SaveResult createIfAbsent(Notification n);
    Optional<Notification> findById(String id);
}
```

**How this file works:** Creation returns the stored notification and whether it was new. findById returns Optional to make absence explicit. This minimal port suits an in-memory domain model. SQL adapters require explicit state updates; mutating the returned Java object alone would not update a database.

### 4.12. `src/notification/SaveResult.java` — Atomic creation result

```java
package notification;


record SaveResult(Notification notification, boolean created) {}
```

**How this file works:** This record avoids overloading null or exceptions to indicate duplicates. created controls whether the service schedules a job. The result and the idempotency check must come from the same atomic repository operation.

### 4.13. `src/notification/InMemoryNotificationRepository.java` — Thread-safe adapter

```java
package notification;

import java.util.Optional;
import java.util.concurrent.ConcurrentMap;
import java.util.concurrent.ConcurrentHashMap;

final class InMemoryNotificationRepository implements NotificationRepository {
    private final ConcurrentMap<String, Notification> items = new ConcurrentHashMap<>();
    private final ConcurrentMap<String, String> byKey = new ConcurrentHashMap<>();

    public synchronized SaveResult createIfAbsent(Notification n) {
        String existingId = byKey.get(n.idempotencyKey());
        if (existingId != null) return new SaveResult(items.get(existingId), false);
        items.put(n.id(), n); byKey.put(n.idempotencyKey(), n.id());
        return new SaveResult(n, true);
    }
    public Optional<Notification> findById(String id) { return Optional.ofNullable(items.get(id)); }
}
```

**How this file works:** The synchronized createIfAbsent method coordinates both the item and key maps. ConcurrentHashMap alone would not make a multi-map check-and-insert atomic. findById safely reads from its concurrent map, and the returned entity synchronizes its mutable state. Locks here only protect this JVM.

### 4.14. `src/notification/MessageQueue.java` — Queue abstraction

```java
package notification;

import java.time.Duration;

interface MessageQueue {
    void publish(String notificationId, Duration delay);
    String take() throws InterruptedException;
}
```

**How this file works:** publish accepts a delay; take blocks until an ID is ready. This is a demo scheduling contract, without durable acceptance or broker acknowledgements. A production broker interface needs delivery handles and ack/nack behavior.

### 4.15. `src/notification/InMemoryMessageQueue.java` — Scheduled in-memory queue

```java
package notification;

import java.time.Duration;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.LinkedBlockingQueue;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

final class InMemoryMessageQueue implements MessageQueue, AutoCloseable {
    private final BlockingQueue<String> messages = new LinkedBlockingQueue<>();
    private final ScheduledExecutorService scheduler = Executors.newSingleThreadScheduledExecutor();
    public void publish(String id, Duration delay) {
        scheduler.schedule(() -> messages.offer(id), delay.toMillis(), TimeUnit.MILLISECONDS);
    }
    public String take() throws InterruptedException { return messages.take(); }
    public void close() { scheduler.shutdownNow(); }
}
```

**How this file works:** The scheduler makes delayed IDs available to workers without making worker threads sleep through backoff. take blocks efficiently on BlockingQueue. Both the queue and scheduled backlog are unbounded here; a fixed worker count does not limit memory consumption. close interrupts scheduling. Real traffic needs explicit admission limits and a durable delay/retry mechanism.

### 4.16. `src/notification/PreferenceService.java` — Preferences interface

```java
package notification;


interface PreferenceService { boolean allowed(String userId, Channel channel); }
```

**How this file works:** The service knows the question it needs answered: is this user allowed to receive on this channel? It does not need to know the preference storage schema.

### 4.17. `src/notification/AllowAllPreferences.java` — Demo preference adapter

```java
package notification;


final class AllowAllPreferences implements PreferenceService {
    public boolean allowed(String userId, Channel channel) { return true; }
}
```

**How this file works:** Always allowing delivery is useful for a deterministic demonstration. Supply another implementation through the constructor to simulate opt-out or query user settings.

### 4.18. `src/notification/NotificationService.java` — Application service

```java
package notification;

import java.util.UUID;
import java.time.Duration;

final class NotificationService {
    private final NotificationRepository repository;
    private final MessageQueue queue;
    private final PreferenceService preferences;

    NotificationService(NotificationRepository repository, MessageQueue queue,
                        PreferenceService preferences) {
        this.repository = repository; this.queue = queue; this.preferences = preferences;
    }

    Notification create(CreateRequest req) {
        if (req.idempotencyKey() == null || req.idempotencyKey().isBlank()
                || req.userId() == null || req.userId().isBlank()
                || req.channel() == null || req.destination() == null || req.destination().isBlank()
                || req.body() == null || req.body().isBlank()) {
            throw new IllegalArgumentException("Missing required field");
        }
        if (!preferences.allowed(req.userId(), req.channel())) {
            throw new IllegalStateException("User opted out");
        }
        var n = new Notification(UUID.randomUUID().toString(), req.idempotencyKey(),
                req.userId(), req.channel(), req.destination(), req.subject(), req.body());
        SaveResult saved = repository.createIfAbsent(n);
        if (saved.created()) {
            try { queue.publish(saved.notification().id(), Duration.ZERO); }
            catch (RuntimeException e) { saved.notification().markFailed(e.getMessage()); throw e; }
        }
        return saved.notification();
    }
}
```

**How this file works:** create validates, checks preferences, creates a UUID, deduplicates atomically and publishes only when newly inserted. Scheduling rejection marks the record failed and propagates the exception. UUID generation is kept directly in this small service; inject a Supplier if deterministic IDs become valuable. The separate save/publish operations are not transactional; see the outbox discussion.

### 4.19. `src/notification/RetryPolicy.java` — Bounded backoff policy

```java
package notification;

import java.time.Duration;
import java.util.Optional;

final class RetryPolicy {
    private final int maxAttempts;
    private final Duration baseDelay;
    RetryPolicy(int maxAttempts, Duration baseDelay) {
        if (maxAttempts < 1 || baseDelay.isNegative() || baseDelay.isZero()) {
            throw new IllegalArgumentException("Positive attempts and delay required");
        }
        this.maxAttempts = maxAttempts; this.baseDelay = baseDelay;
    }
    Optional<Duration> nextDelay(int attempt) {
        if (attempt < 1 || attempt >= maxAttempts) return Optional.empty();
        Duration cap = Duration.ofSeconds(30);
        Duration delay = baseDelay.compareTo(cap) > 0 ? cap : baseDelay;
        for (int i = 1; i < attempt && delay.compareTo(cap) < 0; i++) {
            delay = delay.multipliedBy(2);
            if (delay.compareTo(cap) > 0) delay = cap;
        }
        return Optional.of(delay);
    }
}
```

**How this file works:** Constructor checks the configuration. nextDelay returns Optional.empty when attempts are exhausted, otherwise doubles delay up to 30 seconds. The count includes the original send. This policy is a concrete class because there is only one algorithm in this exercise.

### 4.20. `src/notification/Dispatcher.java` — Consumer and worker-pool owner

```java
package notification;

import java.time.Duration;
import java.util.Optional;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

final class Dispatcher implements AutoCloseable {
    private final NotificationRepository repository;
    private final MessageQueue queue;
    private final SenderFactory factory;
    private final RetryPolicy retryPolicy;
    private final ExecutorService workers;
    private final int workerCount;
    private volatile boolean running = true;

    Dispatcher(NotificationRepository repository, MessageQueue queue,
               SenderFactory factory, RetryPolicy retryPolicy, int workerCount) {
        this.repository = repository; this.queue = queue;
        this.factory = factory; this.retryPolicy = retryPolicy;
        if (workerCount < 1) throw new IllegalArgumentException("workers must be positive");
        this.workerCount = workerCount;
        this.workers = Executors.newFixedThreadPool(workerCount);
    }

    void start() {
        for (int i = 0; i < workerCount; i++) {
            workers.submit(() -> {
                while (running && !Thread.currentThread().isInterrupted()) {
                    try { process(queue.take()); }
                    catch (InterruptedException e) { Thread.currentThread().interrupt(); }
                }
            });
        }
    }

    private void process(String id) {
        Notification n = repository.findById(id).orElse(null);
        if (n == null || !n.claim()) return;
        try {
            factory.get(n.channel()).send(n);
            n.markSent();
        } catch (ProviderException e) {
            Optional<Duration> delay = e.retryable() ? retryPolicy.nextDelay(n.attempts()) : Optional.empty();
            if (delay.isPresent()) {
                n.markRetry(e.getMessage());
                try { queue.publish(id, delay.get()); }
                catch (RuntimeException failure) { n.markFailed(failure.getMessage()); }
            }
            else n.markFailed(e.getMessage());
        } catch (RuntimeException e) {
            n.markFailed(e.getMessage());
        }
    }

    public void close() {
        running = false;
        workers.shutdownNow();
        try {
            if (!workers.awaitTermination(3, TimeUnit.SECONDS)) {
                throw new IllegalStateException("Workers did not stop; provider must honor timeouts/interruption");
            }
        } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
    }
}
```

**How this file works:** start submits exactly workerCount long-lived consumer loops once. Each loop takes an ID and process claims the entity before selecting a strategy. ProviderException is retried only when flagged retryable; unexpected runtime errors terminate the notification. close interrupts consumers and waits for termination. This is a one-start lifecycle object; repeatedly calling start is unsupported. Real network clients must use timeouts and honor cancellation so shutdown cannot hang.

### 4.21. `src/notification/Main.java` — Composition root

```java
package notification;

import java.time.Duration;
import java.util.Map;

public class Main {
    public static void main(String[] args) throws Exception {
        var repository = new InMemoryNotificationRepository();
        try (var queue = new InMemoryMessageQueue()) {
            var factory = new SenderFactory(Map.of(
                    Channel.EMAIL, new EmailSender(), Channel.SMS, new SmsSender(), Channel.PUSH, new PushSender()));
            var service = new NotificationService(repository, queue, new AllowAllPreferences());
            try (var dispatcher = new Dispatcher(repository, queue, factory,
                    new RetryPolicy(3, Duration.ofMillis(100)), 4)) {
                dispatcher.start();
                Notification n = service.create(new CreateRequest("order-42-shipped", "user-7",
                        Channel.EMAIL, "learner@example.com", "Order shipped", "Your order is on its way."));
                long deadline = System.nanoTime() + Duration.ofSeconds(3).toNanos();
                while (n.status() != Status.SENT && n.status() != Status.FAILED) {
                    if (System.nanoTime() >= deadline) throw new IllegalStateException("Delivery timed out");
                    Thread.sleep(1);
                }
                System.out.printf("FINAL id=%s status=%s attempts=%d%n", n.id(), n.status(), n.attempts());
            }
        }
    }
}
```

**How this file works:** Main wires adapters, the registry, retry configuration and four workers. try-with-resources closes the dispatcher before closing the queue. It waits for sent/failed with a deadline and prints the result; this makes the demo meaningful on slower machines. The application uses a package, so java must run notification.Main with the compiled output directory on the classpath.

### 4.22. `src/notification/NotificationTest.java` — Executable tests without a framework

```java
package notification;


public class NotificationTest {
    static final class RecordingQueue implements MessageQueue {
        int published;
        public void publish(String id, java.time.Duration delay) { published++; }
        public String take() { throw new UnsupportedOperationException(); }
    }
    static void check(boolean condition, String message) {
        if (!condition) throw new AssertionError(message);
    }
    public static void main(String[] args) throws Exception {
        var repository = new InMemoryNotificationRepository();
        var queue = new RecordingQueue();
        var service = new NotificationService(repository, queue, new AllowAllPreferences());
        var request = new CreateRequest("k","u",Channel.EMAIL,"a@b","subject","body");
        var first = service.create(request);
        var duplicate = service.create(request);
        check(first.id().equals(duplicate.id()) && queue.published == 1,"idempotency failed");
        check(first.claim() && !first.claim(),"claim must be exclusive");
        first.markSent();
        check(!first.claim(),"sent notification claimed again");
        var attempts = new java.util.concurrent.atomic.AtomicInteger();
        NotificationSender flaky = n -> {
            if (attempts.incrementAndGet() == 1) throw new ProviderException("try again",true);
        };
        try (var realQueue = new InMemoryMessageQueue();
             var dispatcher = new Dispatcher(repository,realQueue,
                 new SenderFactory(java.util.Map.of(Channel.EMAIL,flaky)),
                 new RetryPolicy(3,java.time.Duration.ofMillis(1)),1)) {
            dispatcher.start();
            var svc = new NotificationService(repository,realQueue,new AllowAllPreferences());
            var n = svc.create(new CreateRequest("retry","u",Channel.EMAIL,"a@b","s","b"));
            long deadline = System.nanoTime()+java.time.Duration.ofSeconds(3).toNanos();
            while(n.status()!=Status.SENT && System.nanoTime()<deadline) Thread.sleep(1);
            check(n.status()==Status.SENT && n.attempts()==2,"retry failed");
        }
        System.out.println("All tests passed");
    }
}
```

**How this file works:** The test main throws AssertionError explicitly, so it does not require the JVM -ea option. It checks duplicate creation, exclusive claiming, terminal-state deduplication and a real asynchronous retry that succeeds on attempt two. A production repository would normally use JUnit, but no external libraries are required to run these checks.

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

## Java OOP and its Go equivalent

| Concept | Java in this guide | Go in the companion guide |
|---|---|---|
| Encapsulation | private fields, package-private methods | repository-controlled mutation and package visibility |
| Polymorphism | NotificationSender with implements | Sender satisfied implicitly |
| Construction | constructors and new | New... functions and struct literals |
| Immutable input | CreateRequest record | request struct passed by value |
| Concurrency | executor worker loops, synchronized entity | goroutines, mutex-protected repository |
| Cancellation | interruption and close | context cancellation and WaitGroup |

You do not need inheritance to demonstrate OOP. The important relationship is “EmailSender is a NotificationSender,” expressed through an interface; Service **has** a repository, queue and preference service. Abstract base classes become useful only when shared behavior and a stable subtype relationship justify them. Adding an abstract class solely to display inheritance usually weakens this design.
