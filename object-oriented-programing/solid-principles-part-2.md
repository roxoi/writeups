---
title: "Module 5 (Part 2): Design Patterns, Principles & the LLD Interview Playbook"
description: "Learn Strategy, Observer, Decorator, State and Chain of Responsibility in Java 17+, build a complete notification system, and get a 45-minute low-level design interview playbook with a mock question bank."
author: ["name": "Rajendra Pancholi", "email": "rpancholi522@gmail.com"]
thumbnail: "/images/oop-design-patterns-lld.png"
tags: [OOP, Java, Design-Patterns, Low-Level-Design, Interview-Preparation]
keywords: ["Strategy Observer Decorator pattern Java", "State and Chain of Responsibility pattern", "LLD interview playbook", "Low level design mock interview questions", "DRY KISS YAGNI Law of Demeter"]
---

# OOP Mastery for MAANG Interviews: Module 5 (Part 2 of 2) — Behavioral Patterns, Principles & the LLD Playbook

![Module 5 Part 2: Design Patterns and LLD Playbook](/images/oop-design-patterns-lld.png)

**Language:** Java 17+ (records, `var`, lambdas, `sealed`). Same rules as Part 1: money is stored as `long` cents or `BigDecimal`, never `double`.

**This is Part 2 of 2.** Part 1 ([SOLID & Creational Patterns](./solid-principles.md)) covered SOLID, Singleton, Factory Method, Abstract Factory and Builder. This part covers:

- **5.3** Strategy, Observer, Decorator, and a complete **Notification System** that combines them
- **5.4** State, Chain of Responsibility, and a **pattern cheat sheet**
- **5.5** Principles beyond SOLID (composition over inheritance, Law of Demeter, DRY, KISS, YAGNI)
- **5.6** The **45-minute LLD interview playbook**
- **5.7** The **mock interview bank** and the **final checklist**
- A closing **"SOLID in simple words"** recap

### How this part connects to Part 1

In Part 1 you learned how to judge a design (SOLID) and how to create objects (creational patterns). Now the question is: **how do objects talk to each other, and how do you add behavior without editing old code?** Every pattern in this part is polymorphism (Module 4) used on purpose: an interface, several implementations, and a caller that does not care which one it gets.

### Words you will meet in this part

| Word                    | Simple meaning                                                                                 |
| ----------------------- | ---------------------------------------------------------------------------------------------- |
| **Behavioral pattern**  | A pattern about how objects cooperate and share work.                                          |
| **Structural pattern**  | A pattern about how objects are assembled into bigger structures (Decorator is one).           |
| **Strategy**            | A swappable algorithm behind an interface.                                                     |
| **Observer**            | Subscribers get told when something happens. The publisher does not know who they are.         |
| **Decorator**           | A wrapper that adds behavior and has the same interface as the thing it wraps.                 |
| **State**               | An object changes its behavior when its internal state changes.                                |
| **Chain**               | A request passes through a line of handlers. Each one handles it or passes it on.              |
| **Event**               | An immutable record of "something happened".                                                   |
| **Idempotent**          | Doing it twice has the same effect as doing it once (important for retries).                   |
| **Backoff**             | Waiting longer between each retry so you do not hammer a failing service.                      |
| **LLD**                 | Low-Level Design: designing classes, interfaces and their relations for one feature or system. |

## 5.3 Strategy, Observer & Decorator

### Strategy

#### The idea in plain words

You open a maps app. You can choose "fastest", "shortest" or "avoid tolls". The app does not rewrite itself for each choice. It has one "get me there" screen and **swaps the route algorithm** behind it.

**Intent:** define a family of algorithms, put each behind a common interface, and make them interchangeable. The caller picks one and uses it through the interface.

#### Bad code

```java
// ❌ Every new shipping method edits this method. Old methods can break.
class ShippingCalculator {
    long cost(String method, int weightGrams) {
        switch (method) {
            case "STANDARD": return 500 + weightGrams / 100;
            case "EXPRESS":  return 1500 + weightGrams / 50;
            case "PICKUP":   return 0;
            default: throw new IllegalArgumentException(method);
        }
    }
}
```

This violates OCP (Part 1): a new method such as `"DRONE"` means editing and re-testing the whole method.

#### Refactored

```java
import java.util.Map;

record Parcel(int weightGrams, String country) { }

/** One method = a functional interface, so lambdas work as strategies. */
@FunctionalInterface
interface ShippingStrategy {
    long costCents(Parcel parcel);
}

final class ShippingStrategies {
    static final ShippingStrategy STANDARD = p -> 500 + p.weightGrams() / 100;
    static final ShippingStrategy EXPRESS  = p -> 1500 + p.weightGrams() / 50;
    static final ShippingStrategy PICKUP   = p -> 0;
    private ShippingStrategies() { }
}

/** Context: holds a strategy and uses it. Knows nothing about the formulas. */
final class Checkout {
    private final Map<String, ShippingStrategy> byName;

    Checkout(Map<String, ShippingStrategy> byName) { this.byName = Map.copyOf(byName); }

    long shippingFor(String method, Parcel parcel) {
        ShippingStrategy s = byName.get(method);
        if (s == null) throw new IllegalArgumentException("Unknown shipping: " + method);
        return s.costCents(parcel);
    }
}
// new Checkout(Map.of("STANDARD", STANDARD, "EXPRESS", EXPRESS, "PICKUP", PICKUP))
// Adding DRONE = one new lambda and one new map entry in the composition root.
```

How to read it: `Checkout` is the stable part. The formulas are the varying part, and each one lives alone behind `ShippingStrategy`. Because the interface has one method, a strategy can be as small as a lambda.

#### Interview Q&A

**Q1. (SDE-1) Strategy vs a big `if/else`?** `if/else` puts all algorithms in one place that must be edited for each new one (OCP violation). Strategy puts each algorithm in its own unit and lets the caller choose.

**Q2. (SDE-2) Strategy vs State?** Structurally they look the same (a context delegating to an interface). The difference is **who changes the object**. In Strategy, the *client* picks the algorithm and it usually stays put. In State, the object *changes its own state* as events happen, and the behavior follows. See 5.4.

**Q3. (SDE-2) Does every Strategy need its own class?** No. In Java 8+, if the interface has one method, use a lambda or method reference. Use a class when the strategy has state, configuration or needs a name for logging.

**Q4. (SDE-3) Where do real systems use Strategy?** `Comparator` passed to `sort`, retry and backoff policies, load-balancing algorithms (round robin, least connections), pricing and discount engines, compression choice, and cache-eviction policies (LRU, LFU).

**Q5. (SDE-3) How do you choose the strategy at run time?** Through a registry map (as above), a factory (Part 1), or configuration. Keep strategies **stateless and immutable** where you can, so one instance can be shared by many threads.

### Observer

#### The idea in plain words

You subscribe to a newsletter. The publisher does not phone you. It sends one email to the list. New subscribers can join and old ones can leave without the publisher changing anything.

**Intent:** define a one-to-many dependency, so that when one object (the **subject** or publisher) changes or something happens, all its **observers** (subscribers) are notified automatically.

#### Bad code

```java
// ❌ OrderService knows every consumer of "order placed". Each new need edits it.
class OrderService {
    private final EmailSender email = new EmailSender();
    private final InventoryService inventory = new InventoryService();
    private final AnalyticsClient analytics = new AnalyticsClient();

    void place(Order o) {
        // ... save order ...
        email.sendConfirmation(o);
        inventory.reserve(o);
        analytics.track("order", o);     // loyalty points next sprint -> edit again
    }
}
```

Problems: tight coupling (DIP and OCP violations), a slow email blocks the order, and one crash in analytics can break the order flow.

#### Refactored

```java
import java.util.List;
import java.util.concurrent.CopyOnWriteArrayList;
import java.util.function.Consumer;

/** Event = immutable fact. Never pass mutable objects to observers. */
record OrderPlaced(String orderId, String email, long totalCents) { }

/** Subject (a tiny event bus). Thread-safe subscribe/unsubscribe/publish. */
final class OrderEvents {
    // CopyOnWriteArrayList: iteration never fails if someone (un)subscribes meanwhile.
    // Good when reads (publish) are far more common than writes (subscribe).
    private final List<Consumer<OrderPlaced>> listeners = new CopyOnWriteArrayList<>();

    /** Returns an "unsubscribe" handle. Callers MUST call it to avoid leaks. */
    Runnable subscribe(Consumer<OrderPlaced> listener) {
        listeners.add(listener);
        return () -> listeners.remove(listener);
    }

    void publish(OrderPlaced event) {
        for (Consumer<OrderPlaced> l : listeners) {
            try {
                l.accept(event);
            } catch (RuntimeException ex) {
                // ISOLATE failures: one bad observer must not stop the others.
                System.err.println("listener failed: " + ex);
            }
        }
    }
}

final class OrderService {
    private final OrderEvents events;
    OrderService(OrderEvents events) { this.events = events; }

    void place(String id, String email, long total) {
        // ... save order ...
        events.publish(new OrderPlaced(id, email, total));   // that is ALL it knows
    }
}

// Composition root:
// var events = new OrderEvents();
// events.subscribe(e -> emailService.sendConfirmation(e.email()));
// events.subscribe(e -> inventory.reserve(e.orderId()));
// events.subscribe(e -> loyalty.addPoints(e.email(), e.totalCents() / 100));  // new feature: zero edits to OrderService
```

How to read it: `OrderService` only says "an order was placed." Who cares about that is decided elsewhere. Adding loyalty points is one `subscribe` line.

#### The pitfalls (this is what interviewers probe)

| Pitfall                          | What goes wrong                                                          | Fix                                                                         |
| -------------------------------- | ------------------------------------------------------------------------ | --------------------------------------------------------------------------- |
| **Lapsed listener (memory leak)**| A subscriber is never removed, so the subject keeps it alive forever.    | Return an unsubscribe handle, use `WeakReference`, or tie to a lifecycle.   |
| **Exception in one observer**    | Later observers never run.                                               | Catch and log per observer (as above).                                      |
| **Slow observer**                | Synchronous call blocks the publisher.                                   | Dispatch on an `ExecutorService`, or use a queue (Kafka, SQS).              |
| **Modification while iterating** | `ConcurrentModificationException` on a plain `ArrayList`.                | `CopyOnWriteArrayList`, or copy the list before iterating.                  |
| **Re-entrancy**                  | An observer publishes another event while handling one, causing loops.   | Queue events, or forbid publishing from listeners.                          |
| **Ordering**                     | Observers are expected to run in a fixed order, but nothing guarantees it.| Do not depend on order. If you must, you need a different design.           |
| **Mutable events**               | An observer changes the event and others see the change.                 | Use records (immutable).                                                    |

#### Interview Q&A

**Q1. (SDE-1) Observer vs Pub/Sub?** In classic Observer, the subject holds direct references to its observers (same process, they know each other through an interface). In Pub/Sub, a **broker** sits between publishers and subscribers, so they do not know each other, and it can span processes (Kafka, RabbitMQ). Pub/Sub is Observer scaled up with a middleman.

**Q2. (SDE-2) Synchronous or asynchronous notification?** Synchronous is simple and ordered, but a slow observer slows the publisher and a failure can propagate. Asynchronous (executor or queue) isolates the publisher but loses ordering and makes errors harder to see. Choose based on whether the publisher needs the observers' work finished before it continues.

**Q3. (SDE-2) Push vs pull model?** In **push**, the subject sends the data in the event (`OrderPlaced` carries the fields). In **pull**, the subject only says "I changed" and the observer asks for what it needs. Push is simpler and keeps observers decoupled from the subject's internals. Pull avoids sending data nobody needs.

**Q4. (SDE-3) How do you guarantee at-least-once delivery to observers across services?** In-process Observer cannot. Use a durable queue, and design observers to be **idempotent** (use the event id to ignore duplicates). For "save to DB and publish event atomically" use the **transactional outbox** pattern: write the event to an outbox table in the same transaction, and a relay publishes it.

**Q5. Java built-ins?** `java.util.Observable` is deprecated since Java 9 (not type-safe, not thread-safe by design). Prefer `PropertyChangeListener`, `Flow` (reactive streams), or your own tiny bus as above.

### Decorator

#### The idea in plain words

You order a coffee. Then you add milk. Then caramel. Each topping wraps what you already have and adds something, and it is still "a coffee" at the end. You did not need a class for "milk-caramel-coffee".

**Intent:** attach extra behavior to an object **dynamically** by wrapping it in another object that implements the **same interface**.

#### Bad code

```java
// ❌ Subclass explosion: every combination needs its own class.
class Store { }
class LoggingStore extends Store { }
class CachingStore extends Store { }
class LoggingCachingStore extends Store { }
class LoggingCachingEncryptedStore extends Store { }   // 2^n classes for n features
```

#### Refactored

```java
import java.util.concurrent.ConcurrentHashMap;
import java.util.Map;

interface DataStore {
    void write(String key, byte[] data);
    byte[] read(String key);
}

/** The real thing. */
final class InMemoryStore implements DataStore {
    private final Map<String, byte[]> map = new ConcurrentHashMap<>();
    @Override public void write(String key, byte[] data) { map.put(key, data.clone()); }
    @Override public byte[] read(String key) { byte[] v = map.get(key); return v == null ? null : v.clone(); }
}

/** Decorator 1: same interface, wraps another DataStore. */
final class LoggingStore implements DataStore {
    private final DataStore inner;
    LoggingStore(DataStore inner) { this.inner = inner; }

    @Override public void write(String key, byte[] data) {
        System.out.println("write " + key + " (" + data.length + " bytes)");
        inner.write(key, data);                       // delegate!
    }
    @Override public byte[] read(String key) {
        System.out.println("read " + key);
        return inner.read(key);
    }
}

/** Decorator 2: a read-through cache in front of any DataStore. */
final class CachingStore implements DataStore {
    private final DataStore inner;
    private final Map<String, byte[]> cache = new ConcurrentHashMap<>();
    CachingStore(DataStore inner) { this.inner = inner; }

    @Override public void write(String key, byte[] data) {
        inner.write(key, data);
        cache.put(key, data.clone());
    }
    @Override public byte[] read(String key) {
        byte[] hit = cache.get(key);
        if (hit != null) return hit.clone();
        byte[] value = inner.read(key);
        if (value != null) cache.put(key, value.clone());
        return value;
    }
}

// Stack features at run time, in any order, with ZERO new classes per combination:
// DataStore store = new LoggingStore(new CachingStore(new InMemoryStore()));
```

How to read it: every decorator **is a** `DataStore` (so the caller cannot tell) and **has a** `DataStore` (the thing it wraps). Order matters: here logging is outermost, so every call is logged, even cache hits. Swap the order and cache hits would not be logged.

**Java's own example:** `new BufferedInputStream(new GZIPInputStream(new FileInputStream("a.gz")))`. Each layer wraps the next and is still an `InputStream`.

#### Interview Q&A

**Q1. (SDE-1) Decorator vs inheritance?** Inheritance fixes the extra behavior at compile time, and combinations explode. Decorator adds behavior at run time and combinations are just wrapping order. It is "composition over inheritance" (5.5) in action.

**Q2. (SDE-2) Decorator vs Proxy vs Adapter?**

| Pattern       | Interface of wrapper vs wrapped | Purpose                                                              |
| ------------- | ------------------------------- | -------------------------------------------------------------------- |
| **Decorator** | Same                            | Add behavior (caller often builds the stack)                         |
| **Proxy**     | Same                            | Control access (lazy load, security, remote call). Often hidden from the caller |
| **Adapter**   | Different                       | Convert one interface into another                                   |

**Q3. (SDE-2) What are the downsides?** Many small objects, long stack traces, and order sensitivity (encrypt-then-compress is not the same as compress-then-encrypt). Identity checks (`==`, `instanceof`) stop working because the object you hold is a wrapper.

**Q4. (SDE-3) Where do you see Decorators in frameworks?** Servlet filters and middleware, `Collections.unmodifiableList` and `Collections.synchronizedList`, Spring AOP proxies (cross-cutting concerns like transactions and logging), and retry or circuit-breaker wrappers around clients.

**Q5. LSP angle?** A decorator must honor the wrapped interface's **contract**. A caching decorator that returns stale data when the contract promises fresh data breaks LSP.

### The Integrated Notification System

Now we combine what you have learned. This is a classic LLD interview problem.

**Requirements (what you would confirm with the interviewer):**

- Send notifications over Email, SMS and Push.
- Each user has preferred channels.
- High-priority messages go to **all** channels.
- Delivery can fail temporarily, so **retry with backoff**.
- Every attempt is **logged**.
- Other parts of the system want to **react** to delivery results (metrics, alerts).
- New channels must be addable without editing existing code.

**Pattern map:**

| Need                                    | Pattern                  |
| --------------------------------------- | ------------------------ |
| Many channels behind one interface      | Polymorphism / Strategy  |
| Which channels to use for a message     | Strategy                 |
| Retry and logging around any channel    | Decorator                |
| Create channels by name                 | Factory (registry)       |
| React to delivery results               | Observer                 |
| Building a message safely               | Immutable record         |

```java
import java.util.*;
import java.util.concurrent.CopyOnWriteArrayList;

// Message (immutable)
record Notification(String userId, String subject, String body, Priority priority) {
    enum Priority { NORMAL, HIGH }
    Notification {
        Objects.requireNonNull(userId); Objects.requireNonNull(subject);
        Objects.requireNonNull(body);   Objects.requireNonNull(priority);
    }
}

// Channel abstraction (Strategy for "how to deliver")
interface NotificationChannel {
    String name();
    void send(Notification n) throws DeliveryException;     // DeliveryException from 5.2
}

final class EmailNotificationChannel implements NotificationChannel {
    @Override public String name() { return "EMAIL"; }
    @Override public void send(Notification n) { System.out.println("EMAIL to " + n.userId() + ": " + n.subject()); }
}
final class SmsNotificationChannel implements NotificationChannel {
    @Override public String name() { return "SMS"; }
    @Override public void send(Notification n) { System.out.println("SMS to " + n.userId() + ": " + n.subject()); }
}
final class PushNotificationChannel implements NotificationChannel {
    @Override public String name() { return "PUSH"; }
    @Override public void send(Notification n) { System.out.println("PUSH to " + n.userId() + ": " + n.subject()); }
}

// Decorator: retry with exponential backoff
final class RetryingChannel implements NotificationChannel {
    private final NotificationChannel inner;
    private final int maxAttempts;
    private final long baseDelayMillis;

    RetryingChannel(NotificationChannel inner, int maxAttempts, long baseDelayMillis) {
        this.inner = inner; this.maxAttempts = maxAttempts; this.baseDelayMillis = baseDelayMillis;
    }
    @Override public String name() { return inner.name(); }

    @Override public void send(Notification n) throws DeliveryException {
        DeliveryException last = null;
        for (int attempt = 1; attempt <= maxAttempts; attempt++) {
            try {
                inner.send(n);
                return;
            } catch (DeliveryException e) {
                last = e;
                if (attempt == maxAttempts) break;
                try {
                    Thread.sleep(baseDelayMillis << (attempt - 1));   // 100, 200, 400 ...
                } catch (InterruptedException ie) {
                    Thread.currentThread().interrupt();               // restore the flag
                    throw new DeliveryException("interrupted while retrying", ie);
                }
            }
        }
        throw last;
    }
}

// Decorator: logging
final class LoggingChannel implements NotificationChannel {
    private final NotificationChannel inner;
    LoggingChannel(NotificationChannel inner) { this.inner = inner; }
    @Override public String name() { return inner.name(); }

    @Override public void send(Notification n) throws DeliveryException {
        System.out.println("[log] " + name() + " attempt for " + n.userId());
        try {
            inner.send(n);
            System.out.println("[log] " + name() + " OK");
        } catch (DeliveryException e) {
            System.out.println("[log] " + name() + " FAILED: " + e.getMessage());
            throw e;
        }
    }
}

// Strategy: which channels for this message?
interface ChannelSelectionStrategy {
    Set<String> select(Notification n, Set<String> userPreferences, Set<String> available);
}

/** Use only what the user prefers (and what exists). */
final class PreferredChannels implements ChannelSelectionStrategy {
    @Override public Set<String> select(Notification n, Set<String> prefs, Set<String> available) {
        Set<String> result = new LinkedHashSet<>(prefs);
        result.retainAll(available);
        return result;
    }
}

/** HIGH priority ignores preferences and uses everything. */
final class PriorityAwareSelection implements ChannelSelectionStrategy {
    private final ChannelSelectionStrategy normal = new PreferredChannels();
    @Override public Set<String> select(Notification n, Set<String> prefs, Set<String> available) {
        return n.priority() == Notification.Priority.HIGH
                ? new LinkedHashSet<>(available)
                : normal.select(n, prefs, available);
    }
}

// Ports
interface PreferenceStore { Set<String> channelsFor(String userId); }

// Observer: who wants to know about results?
record DeliveryResult(Notification notification, String channel, boolean success) { }
interface DeliveryListener { void onResult(DeliveryResult result); }

// The service: orchestrates, holds no business rules
final class NotificationService {
    private final Map<String, NotificationChannel> channels;     // already decorated
    private final PreferenceStore preferences;
    private final ChannelSelectionStrategy selection;
    private final List<DeliveryListener> listeners = new CopyOnWriteArrayList<>();

    NotificationService(Map<String, NotificationChannel> channels,
                        PreferenceStore preferences,
                        ChannelSelectionStrategy selection) {
        this.channels = Map.copyOf(channels);
        this.preferences = preferences;
        this.selection = selection;
    }

    void addListener(DeliveryListener l) { listeners.add(l); }

    void notify(Notification n) {
        Set<String> chosen = selection.select(n, preferences.channelsFor(n.userId()), channels.keySet());
        for (String name : chosen) {
            boolean ok = true;
            try {
                channels.get(name).send(n);
            } catch (DeliveryException e) {
                ok = false;                                  // one failing channel must not stop the others
            }
            DeliveryResult result = new DeliveryResult(n, name, ok);
            for (DeliveryListener l : listeners) {
                try { l.onResult(result); } catch (RuntimeException ignored) { }
            }
        }
    }
}

// ── Composition root: the ONLY place that knows concrete classes
final class NotificationApp {
    public static void main(String[] args) {
        Map<String, NotificationChannel> channels = new LinkedHashMap<>();
        for (NotificationChannel raw : List.of(
                new EmailNotificationChannel(), new SmsNotificationChannel(), new PushNotificationChannel())) {
            channels.put(raw.name(), new LoggingChannel(new RetryingChannel(raw, 3, 100)));
        }
        PreferenceStore prefs = userId -> Set.of("EMAIL", "PUSH");

        var service = new NotificationService(channels, prefs, new PriorityAwareSelection());
        service.addListener(r -> System.out.println("[metrics] " + r.channel() + " success=" + r.success()));

        service.notify(new Notification("u1", "Welcome", "Hello!", Notification.Priority.NORMAL));   // EMAIL + PUSH
        service.notify(new Notification("u1", "Security alert", "New login", Notification.Priority.HIGH)); // all three
    }
}
```

**Adding a new channel (WhatsApp):** write `WhatsAppChannel implements NotificationChannel` and add it to the list in `main`. `NotificationService`, the strategies and the decorators are not touched. That is OCP.

**What to say in the interview:** "Channel is the abstraction. Retry and logging are decorators, so they work for any channel. Selection is a strategy, so rules can change without touching the service. Results are published to observers so metrics do not clutter delivery. The service gets everything through its constructor, so tests can use fakes."

#### Interview Q&A (follow-ups the interviewer will ask)

**Q1. (SDE-2) Delivery is slow. How do you stop one slow channel from blocking the others?** Send each channel on an executor (`ExecutorService`) and collect the results with `CompletableFuture`. Give each channel its own thread pool or bulkhead, so a slow SMS provider cannot starve email.

**Q2. (SDE-3) How do you avoid sending a duplicate notification when a retry happens after a timeout?** Make delivery **idempotent**: attach a unique notification id, and have the provider or your own store ignore a repeated id.

**Q3. (SDE-3) What if the process crashes after saving the request but before sending?** Persist notifications in a queue or table with a status (`PENDING`, `SENT`, `FAILED`) and have a worker process them. This is a durable queue, not an in-memory loop. Retries then survive restarts.

**Q4. (SDE-3) How do you rate limit per user or per channel?** Add another Decorator (`RateLimitedChannel`) with a token bucket per channel. It fits into the same stack with no changes elsewhere.

**Q5. (SDE-2) How do you test `RetryingChannel`?** Inject a fake inner channel that fails twice and then succeeds, and assert it was called three times. Make the sleeper injectable (a `Sleeper` interface) so the test does not really wait.

## 5.4 State, Chain of Responsibility & the Cheat Sheet

### State

#### The idea in plain words

A vending machine acts differently depending on whether you have inserted a coin. Press "dispense" with no coin and nothing happens. With a coin, it gives the item. The machine is the same machine, but its **state** changes what each button does.

**Intent:** let an object change its behavior when its internal state changes, so it appears to change its class.

#### Bad code

```java
// ❌ Every method repeats the same status checks. Adding a state touches all of them.
class Order {
    String status = "NEW";

    void pay()    { if (status.equals("NEW")) status = "PAID"; else throw new IllegalStateException(); }
    void ship()   { if (status.equals("PAID")) status = "SHIPPED"; else throw new IllegalStateException(); }
    void cancel() {
        if (status.equals("NEW") || status.equals("PAID")) status = "CANCELLED";
        else throw new IllegalStateException();   // "RETURNED" state next sprint -> edit every method
    }
}
```

#### Refactored

```java
/** Each state decides what every action does. Invalid actions fail by default. */
enum OrderStatus {
    NEW {
        @Override OrderStatus pay()    { return PAID; }
        @Override OrderStatus cancel() { return CANCELLED; }
    },
    PAID {
        @Override OrderStatus ship()   { return SHIPPED; }
        @Override OrderStatus cancel() { return CANCELLED; }     // would also trigger a refund
    },
    SHIPPED {
        @Override OrderStatus deliver() { return DELIVERED; }
    },
    DELIVERED,
    CANCELLED;

    // Defaults: an action is illegal unless the state overrides it.
    OrderStatus pay()     { throw illegal("pay"); }
    OrderStatus ship()    { throw illegal("ship"); }
    OrderStatus deliver() { throw illegal("deliver"); }
    OrderStatus cancel()  { throw illegal("cancel"); }

    private IllegalStateException illegal(String action) {
        return new IllegalStateException("Cannot " + action + " when order is " + this);
    }
}

final class Order {
    private final String id;
    private OrderStatus status = OrderStatus.NEW;

    Order(String id) { this.id = id; }

    synchronized void pay()     { status = status.pay(); }
    synchronized void ship()    { status = status.ship(); }
    synchronized void deliver() { status = status.deliver(); }
    synchronized void cancel()  { status = status.cancel(); }
    synchronized OrderStatus status() { return status; }
}
```

How to read it: the rules of "what can happen from here" live **inside each state**, not scattered across `if` chains. Adding `RETURNED` is one new constant plus one override in `DELIVERED`.

**State machine view:**

```
NEW --pay--> PAID --ship--> SHIPPED --deliver--> DELIVERED
 |            |
 +--cancel----+--cancel--> CANCELLED
```

#### Interview Q&A

**Q1. (SDE-1) State vs Strategy?** Strategy: the client chooses the algorithm, usually once. State: the object moves between states by itself as events happen, and each state defines behavior and often decides the next state.

**Q2. (SDE-2) Who should own the transitions: the context or the states?** Either. States owning transitions (as above) keeps the rules local and avoids a central table, but the full state diagram is spread across many places. A **table-driven** state machine (`Map<State, Map<Event, State>>`) keeps the whole diagram in one place and is easier to validate and to load from configuration. Choose the table when states are many and rules are data.

**Q3. (SDE-2) How do you persist the state?** Store the enum **name** in the database and rebuild with `OrderStatus.valueOf`. Do not store ordinals, because reordering the enum silently corrupts data.

**Q4. (SDE-3) Concurrency?** Two threads calling `pay()` and `cancel()` at once can interleave badly. Make transitions atomic (a `synchronized` method as above, an `AtomicReference` with `compareAndSet`, or a database row version for optimistic locking across servers).

**Q5. When is State overkill?** With two or three states and little behavior, a simple enum plus a few checks is clearer. Use the pattern when the same `switch (status)` appears in many methods.

### Chain of Responsibility

#### The idea in plain words

You complain about a bill. The front desk tries to fix it. If it cannot, it passes you to a supervisor. If the supervisor cannot, it goes to a manager. Each person either handles the problem or passes it along, and you do not need to know who will end up solving it.

**Intent:** pass a request along a chain of handlers. Each handler decides to handle it, modify it, or pass it on, so the sender is decoupled from the receivers.

#### Scenario: a request pipeline (the idea behind servlet filters and web middleware)

```java
import java.util.List;

record Request(String user, String path) { }
record Response(int status, String body) { }

interface Handler { Response handle(Request request); }

/** A middleware gets the request AND the next handler in the chain. */
interface Middleware { Response handle(Request request, Handler next); }

final class AuthMiddleware implements Middleware {
    @Override public Response handle(Request r, Handler next) {
        if (r.user() == null) return new Response(401, "login required");   // STOP the chain
        return next.handle(r);                                              // or pass it on
    }
}

final class RateLimitMiddleware implements Middleware {
    private final java.util.concurrent.ConcurrentHashMap<String, java.util.concurrent.atomic.AtomicInteger> counts =
            new java.util.concurrent.ConcurrentHashMap<>();
    private final int limit;
    RateLimitMiddleware(int limit) { this.limit = limit; }

    @Override public Response handle(Request r, Handler next) {
        int n = counts.computeIfAbsent(r.user(), u -> new java.util.concurrent.atomic.AtomicInteger()).incrementAndGet();
        if (n > limit) return new Response(429, "too many requests");
        return next.handle(r);
    }
}

final class LoggingMiddleware implements Middleware {
    @Override public Response handle(Request r, Handler next) {
        long start = System.nanoTime();
        Response resp = next.handle(r);                                     // work AFTER the rest ran too
        System.out.printf("%s %s -> %d (%d µs)%n", r.user(), r.path(), resp.status(), (System.nanoTime() - start) / 1000);
        return resp;
    }
}

/** Builds the chain by folding from the end, so the first middleware is outermost. */
final class Pipeline {
    private final Handler head;

    Pipeline(List<Middleware> middlewares, Handler terminal) {
        Handler h = terminal;
        for (int i = middlewares.size() - 1; i >= 0; i--) {
            Middleware m = middlewares.get(i);
            Handler next = h;
            h = request -> m.handle(request, next);
        }
        this.head = h;
    }

    Response handle(Request r) { return head.handle(r); }
}

// var pipeline = new Pipeline(
//         List.of(new LoggingMiddleware(), new AuthMiddleware(), new RateLimitMiddleware(100)),
//         req -> new Response(200, "hello " + req.user()));
```

How to read it: each middleware is small and does one job. Order is the first thing to get right: logging is outermost so it records everything (including rejected requests), auth runs before rate limiting so anonymous requests are stopped early.

#### Interview Q&A

**Q1. (SDE-1) Chain of Responsibility vs Decorator?** The structure is similar (wrapping and delegating). A **Decorator** always delegates and *adds* to the result. A **Chain** handler may **stop** the chain (as `AuthMiddleware` does). Intent differs: Decorator adds behavior, Chain routes or filters a request.

**Q2. (SDE-2) What if no handler handles the request?** Decide explicitly. Use a terminal handler that returns a default (404), or throw. A silent fall-through is a bug waiting to happen.

**Q3. (SDE-2) Where does this appear in real systems?** Servlet filters, Spring Security's filter chain, Express or Koa middleware, logging frameworks (a log record passes through handlers), approval workflows (manager → director → VP), and ATM cash dispensers (₹2000 notes, then ₹500, then ₹100).

**Q4. (SDE-3) Downsides?** Hard to debug (you must trace a request through many handlers), there is no guarantee the request is handled, and long chains add latency. Add logging or tracing at each hop.

**Q5. (SDE-3) Can a chain be changed at run time?** Yes, by rebuilding the list (build a new `Pipeline` and swap the reference atomically; the old one keeps serving in-flight requests). Keep the chain itself immutable after construction.

### Pattern Cheat Sheet

| Pattern              | Intent in one line                               | Reach for it when...                                         | Java example                          | Principle it serves |
| -------------------- | ------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------- | ------------------- |
| **Singleton**        | Exactly one instance                             | One shared resource (prefer DI scope instead)                | `Runtime.getRuntime()`                | (use with care)     |
| **Factory Method**   | Subclass decides what to create                  | Creation varies, the workflow around it is stable            | `Calendar.getInstance()`              | OCP, DIP            |
| **Abstract Factory** | Create families of matching objects              | Products must be compatible (AWS vs GCP)                     | `DocumentBuilderFactory`              | DIP                 |
| **Builder**          | Build complex objects step by step               | Many optional parameters, immutability, validation           | `StringBuilder`, `HttpRequest.newBuilder()` | SRP           |
| **Strategy**         | Swappable algorithm                              | Many `if/else` on "which way to do it"                       | `Comparator`                          | OCP                 |
| **Observer**         | One-to-many notification                         | Many parties react to one event                              | `PropertyChangeListener`              | OCP, DIP            |
| **Decorator**        | Add behavior by wrapping                         | Optional, stackable features                                 | `BufferedInputStream`                 | OCP                 |
| **State**            | Behavior changes with state                      | The same `switch (status)` repeated in many methods          | `Thread.State` + lifecycle logic      | OCP                 |
| **Chain**            | Pass along until someone handles it              | Pipelines, filters, approvals                                | Servlet `Filter`                      | SRP, OCP            |

**How to choose quickly:**

| The requirement sounds like...                                  | Think...                      |
| --------------------------------------------------------------- | ----------------------------- |
| "Support multiple ways to do X, choose at run time"             | Strategy                      |
| "When X happens, several things must happen"                    | Observer                      |
| "Optionally add logging, caching, retry, encryption"            | Decorator                     |
| "Behavior depends on the current status and it changes"         | State                         |
| "Request goes through checks, any of which can reject"          | Chain of Responsibility       |
| "Create the right object by type or config"                     | Factory                       |
| "Objects must match as a set"                                   | Abstract Factory              |
| "Object has many optional fields"                               | Builder                       |

## 5.5 Principles Beyond SOLID

SOLID is not the only set of guidelines interviewers care about. These are shorter, and each one fits in a sentence.

### Composition over inheritance

**Plain words:** build behavior by *having* parts, not by *being* a subtype. Inheritance couples a child to its parent's internals (the fragile base class problem from Module 3). Composition couples you only to an interface.

```java
// ❌ Inheritance just to reuse code: a Stack IS NOT a Vector, but Java's Stack extends Vector,
// so callers can insert in the middle and break stack behavior.

// ✅ Composition: hide the list, expose only stack operations.
final class Stack<T> {
    private final java.util.ArrayDeque<T> items = new java.util.ArrayDeque<>();
    void push(T t) { items.push(t); }
    T pop()        { return items.pop(); }
    boolean isEmpty() { return items.isEmpty(); }
}
```

**Rule of thumb:** use inheritance for a true behavioral "is-a" (it passes LSP) and for a deliberate extension point. Everything else, compose. Strategy, Decorator and State are all composition.

### Law of Demeter (principle of least knowledge)

**Plain words:** talk to your friends, not to strangers. A method should only call methods on: itself, its parameters, objects it creates, and its own fields. Long chains like `a.getB().getC().doIt()` mean you know too much about the structure.

```java
// ❌ Train wreck: Checkout knows Customer has a Wallet that has a Card.
customer.getWallet().getCard().charge(amount);

// ✅ Tell the immediate friend what you want.
customer.pay(amount);       // Customer decides how (wallet, card, voucher)
```

Note: fluent builders (`builder.a().b().c()`) and streams do **not** violate it. They return the *same* kind of object, so you are not digging through someone else's internals.

### Tell, Don't Ask

**Plain words:** do not pull an object's data out to make a decision for it. Tell it what to do.

```java
// ❌ Ask, then decide outside
if (account.getBalance() >= amount) account.setBalance(account.getBalance() - amount);

// ✅ Tell: the rule lives with the data (and is atomic and testable in one place)
account.withdraw(amount);       // throws if insufficient
```

### DRY: Don't Repeat Yourself

**Plain words:** every piece of *knowledge* should have one home. The key word is *knowledge*, not *text*. Two methods that look the same but change for different reasons are **not** duplication, and merging them couples unrelated things (SRP warns against this).

**Test:** "If this rule changes, how many places must I edit?" If more than one, you have real duplication.

### KISS: Keep It Simple

**Plain words:** pick the simplest design that works. If you reach for three patterns to solve a problem that one `if` solves, you are over-engineering.

### YAGNI: You Aren't Gonna Need It

**Plain words:** do not build for a future that may not arrive. Do not add an interface "just in case" when there is one implementation and no sign of a second.

**Rule of Three:** the first time, just write it. The second time, wince and copy. The third time, **refactor into an abstraction**. By then you actually know what varies.

### GRASP: Information Expert and friends

**Information Expert:** give a responsibility to the class that already has the data it needs (`Invoice.subtotal()` belongs to `Invoice`, not to a separate calculator that reads every field).

**Low coupling, high cohesion:** the two underlying goals behind almost every principle on this page.

### Interview Q&A

**Q1. (SDE-1) Give an example of DRY going wrong.** Two validation rules that look identical today (name length for users and for products) get merged into one helper. Later, products need a longer limit and every change must work around the shared helper. The duplication was incidental, not real.

**Q2. (SDE-2) YAGNI vs OCP, how do you decide?** Abstract when the variation is **real or very likely** (a second payment provider is on the roadmap). Do not abstract for a variation you invented. If you are unsure, write the simple version, keep it clean, and refactor on the second occurrence.

**Q3. (SDE-2) Does `a.getB().getC()` always violate the Law of Demeter?** No. If `getB()` returns a plain data structure (a record or DTO), reading through it is fine. It matters when you are reaching into the *behavior and structure* of another object.

**Q4. (SDE-3) Where do principles conflict?** DRY vs coupling (sharing code between two modules couples them), KISS vs OCP (extension points add complexity), and DIP vs performance in hot paths. Strong answers name the trade-off, choose, and say what would change the decision.

## 5.6 The 45-Minute LLD Interview Playbook

### What the interviewer is actually judging

It is not "can you draw UML." It is:

1. Can you turn a vague problem into **clear requirements**?
2. Can you find the right **entities and responsibilities**?
3. Is your code **extensible** and **testable** (SOLID, patterns used with reason)?
4. Do you handle **edge cases and concurrency** sensibly?
5. Can you **communicate** your reasoning and accept feedback?

### The timeline

| Time (min) | Step                       | What to do                                                                                             |
| ---------- | -------------------------- | ------------------------------------------------------------------------------------------------------ |
| 0–5        | **Clarify requirements**   | Ask questions. Write functional requirements, non-functional needs, and what is **out of scope**.      |
| 5–10       | **Entities and use cases** | List nouns (candidate classes) and verbs (candidate methods). Name the main use cases.                 |
| 10–15      | **Design the skeleton**    | Interfaces, key classes, relations. Say which patterns you will use and why.                           |
| 15–35      | **Code the core**          | Write the main flow cleanly. Get one use case fully working before adding more.                        |
| 35–42      | **Extend and harden**      | Add one extension (new type, new rule). Discuss concurrency, failures, edge cases.                     |
| 42–45      | **Test and wrap up**       | How you would test it, what you would do with more time, trade-offs you chose.                         |

### Step 1: Questions to ask (use as a checklist)

- Who are the users and what are the **main use cases**? What is explicitly out of scope?
- What are the **entities** and their relations (one-to-many, many-to-many)?
- Single machine or **distributed**? Do I need persistence, or in-memory is fine?
- **Concurrency**: can many users act at once?
- Which parts will **change most**? (This tells you where to put abstractions.)
- Any **scale or latency** needs? Expected sizes?
- What should happen on **failure** (full lot, invalid input, payment failed)?

### Step 2: The noun-verb trick

Read the requirements and underline **nouns** (likely classes: `Vehicle`, `Ticket`, `Slot`) and **verbs** (likely methods: `park`, `unpark`, `pay`). Then check each noun: does it have its own data and behavior, or is it just a field? Then ask for each verb: *which class has the data needed to do this?* (Information Expert).

### Step 3: Pick patterns from the *requirements*, not from a wish list

| Problem prompt               | Likely design choices                                                                                     |
| ---------------------------- | --------------------------------------------------------------------------------------------------------- |
| **Parking lot**              | Strategy (pricing, slot allocation), Factory (vehicle/slot by type), immutable `Ticket`                   |
| **Elevator system**          | State (idle, moving up, moving down, doors open), Strategy (scheduling algorithm)                         |
| **Vending machine / ATM**    | State (idle, has coin, dispensing), Chain of Responsibility (cash denominations)                          |
| **Splitwise**                | Strategy (equal, exact, percent split), `long` cents for money                                            |
| **Logger framework**         | Chain of Responsibility (levels), Observer or Strategy (appenders), Singleton or DI-scoped instance        |
| **Notification system**      | Strategy, Decorator (retry), Observer (results), Factory (channels), as built in 5.3                      |
| **Rate limiter**             | Strategy (token bucket, sliding window), thread-safety with atomics                                       |
| **Chess / board game**       | Polymorphism for pieces, Strategy for move rules, Command for undo                                        |
| **Library management**       | Entities and relations first, Observer for due-date alerts, State for book status                         |

### Worked mini-example: Parking Lot (core only)

**Requirements I confirmed:** multiple vehicle types (bike, car, truck), each with its own slots. On entry, give a ticket with an assigned slot. On exit, compute the fee from the duration. Fee rules must be changeable. Many gates operate at the same time. Out of scope: payment gateway, multiple floors (extension later).

```java
import java.time.*;
import java.util.*;

enum VehicleType { BIKE, CAR, TRUCK }

/** Immutable: a ticket never changes after issue. */
record Ticket(String id, String slotId, VehicleType type, Instant entryTime) { }

/** Strategy: pricing rules can change without touching ParkingLot. */
interface PricingStrategy {
    long priceCents(VehicleType type, Duration parked);
}

final class HourlyPricing implements PricingStrategy {
    private final Map<VehicleType, Long> centsPerHour;
    HourlyPricing(Map<VehicleType, Long> centsPerHour) { this.centsPerHour = Map.copyOf(centsPerHour); }

    @Override public long priceCents(VehicleType type, Duration parked) {
        long hours = Math.max(1, (parked.toMinutes() + 59) / 60);   // round up, minimum 1 hour
        return hours * centsPerHour.get(type);
    }
}

final class ParkingLot {
    private final Map<VehicleType, Deque<String>> freeSlots = new EnumMap<>(VehicleType.class);
    private final Map<String, Ticket> active = new HashMap<>();
    private final PricingStrategy pricing;
    private final Clock clock;                              // injected so tests control time

    ParkingLot(Map<VehicleType, List<String>> slotsByType, PricingStrategy pricing, Clock clock) {
        slotsByType.forEach((t, ids) -> freeSlots.put(t, new ArrayDeque<>(ids)));
        this.pricing = pricing;
        this.clock = clock;
    }

    /** synchronized = one lock for the whole lot. Correct and simple; see Q&A for finer locking. */
    synchronized Ticket enter(VehicleType type) {
        Deque<String> free = freeSlots.get(type);
        if (free == null || free.isEmpty()) throw new IllegalStateException("No free slot for " + type);
        String slotId = free.pollFirst();
        Ticket ticket = new Ticket(UUID.randomUUID().toString(), slotId, type, clock.instant());
        active.put(ticket.id(), ticket);
        return ticket;
    }

    synchronized long exit(String ticketId) {
        Ticket ticket = active.remove(ticketId);            // remove first: a second exit with the same ticket fails
        if (ticket == null) throw new IllegalArgumentException("Unknown or already used ticket");
        freeSlots.get(ticket.type()).addFirst(ticket.slotId());
        return pricing.priceCents(ticket.type(), Duration.between(ticket.entryTime(), clock.instant()));
    }
}
```

**Things to say out loud:** "Money is `long` cents. `Clock` is injected so I can test overnight fees without waiting. `Ticket` is an immutable record. Pricing is a Strategy because it is the part most likely to change. Removing the ticket first makes double-exit fail safely."

**Extension question you should expect:** "Add multiple floors." → introduce a `Floor` holding its own slots, make `ParkingLot` hold floors, and add a `SlotAllocationStrategy` (nearest to entrance, fill floor by floor). The pricing code stays untouched.

### Common mistakes (and how to avoid them)

| Mistake                                                  | Better                                                              |
| -------------------------------------------------------- | ------------------------------------------------------------------- |
| Jumping to code without asking questions                 | Spend the first 5 minutes on requirements.                          |
| Using five patterns to look smart                        | Use a pattern only when you can name the requirement it serves.     |
| A `Manager` or `Util` class that does everything         | Split by responsibility (SRP).                                      |
| Public mutable fields and setters everywhere             | Immutable records, constructor validation.                          |
| `double` for money                                       | `long` cents or `BigDecimal`.                                       |
| `new` for dependencies inside business classes           | Inject them (DIP), so you can test with fakes.                      |
| Ignoring concurrency                                     | Say what is shared, and protect it (and say what you would refine). |
| Silent failure                                           | Decide: throw, return `Optional`, or return a result type.          |
| Going silent while thinking                              | Narrate your reasoning. The interviewer grades the thinking.        |
| Defending a design when the interviewer hints at a flaw  | Treat hints as help. Say "good point" and adapt.                    |

### Concurrency talking points (ready to use)

1. **Identify shared mutable state first.** If nothing is shared and mutable, you need no locks.
2. Prefer **immutability** and **thread confinement** over locking.
3. For counters use `AtomicLong` or `LongAdder`. For maps use `ConcurrentHashMap` with atomic methods (`computeIfAbsent`, `merge`).
4. Make **check-then-act** atomic (check slot free and take it under one lock or one `compareAndSet`).
5. Start with a coarse lock (correct), then discuss **finer-grained locks** (per slot type, per floor) or lock-free structures for hot paths.
6. Mention **deadlock** if you take more than one lock (always acquire in a fixed order).

## 5.7 Mock Interview Bank & Final Checklist

Practise answering each question **out loud in under two minutes**.

### Conceptual questions

**1. (SDE-1) What is the difference between abstraction and encapsulation?** Abstraction is about showing *what* and hiding *how* (an interface). Encapsulation is about protecting an object's *state and rules* by controlling access. Abstraction is a design view, encapsulation is an enforcement mechanism.

**2. (SDE-1) Name the SOLID principles and give one violation smell for each.** S: a 1000-line `Manager`. O: a growing `switch` on a type code. L: an override that throws `UnsupportedOperationException`. I: empty method implementations. D: `new ConcreteClass()` inside business logic.

**3. (SDE-2) Composition vs inheritance: when do you still use inheritance?** For a true behavioral is-a that passes LSP, and for deliberate extension points (template method, framework base classes).

**4. (SDE-2) How do DIP, DI and IoC differ?** DIP: depend on abstractions owned by the high-level module. DI: a technique (pass dependencies in). IoC: a framework or caller controls the flow. DI is one form of IoC.

**5. (SDE-3) When would you deliberately violate a SOLID principle?** In a small script or a throwaway prototype, in a hot path where interface dispatch matters, or when an abstraction would cost more than the change it protects. The key is to make the choice consciously and document it.

### Pattern-choice questions

**6. (SDE-1) Your `OrderService` calls email, inventory and analytics directly. How do you improve it?** Publish an `OrderPlaced` event and let each consumer subscribe (Observer). Mention isolating failures and idempotent consumers.

**7. (SDE-2) You have a `switch` on `status` in six methods. What do you do?** State pattern: move behavior per state into the state. For many states with rules as data, a table-driven state machine.

**8. (SDE-2) How do you add retry, logging and caching to a client without editing it?** Decorators around the client interface, with attention to wrapping order.

**9. (SDE-2) How do you build a request pipeline with auth, rate limiting and logging?** Chain of Responsibility (middleware), with order: logging, auth, rate limit, handler.

**10. (SDE-3) Singleton or DI-managed single instance?** Prefer a DI-managed single instance: you get one object without global access, and tests can inject fakes.

### Code-reading questions

**11. (SDE-2) What is wrong here?**

```java
class ReportService {
    void run() {
        Database db = new Database("prod-db");
        List<Row> rows = db.query("SELECT * FROM sales");
        System.out.println(format(rows));
    }
}
```

Answer: DIP violation (`new Database` inside business logic, untestable), SRP (querying and formatting and printing in one place), and the output goes straight to `System.out`. Fix: inject a `SalesRepository` and a `ReportFormatter`/`Writer`.

**12. (SDE-2) What is wrong here?**

```java
class Shape { }
class Circle extends Shape { double r; }
class AreaCalculator {
    double area(Shape s) {
        if (s instanceof Circle c) return Math.PI * c.r * c.r;
        if (s instanceof Square q) return q.side * q.side;
        throw new IllegalArgumentException();
    }
}
```

Answer: OCP violation (`instanceof` chain, each new shape edits it). Fix: `interface Shape { double area(); }` with each shape computing its own, or a `sealed` interface with pattern-matching `switch` if operations grow faster than types.

**13. (SDE-3) What is wrong here?**

```java
class Cache {
    private static Cache instance;
    static Cache getInstance() {
        if (instance == null) instance = new Cache();
        return instance;
    }
}
```

Answer: not thread-safe (two threads can create two instances), and no `volatile` for a safe publication. Fix: holder idiom or enum. Also mention global state hurting tests.

### Design prompts (practise with the playbook)

**14.** Design a **parking lot** with multiple floors and dynamic pricing.
**15.** Design an **elevator system** with several elevators and a scheduling policy.
**16.** Design a **rate limiter** supporting multiple algorithms.
**17.** Design **Splitwise** (groups, expenses, equal/exact/percent splits, balance simplification).
**18.** Design a **logging library** with levels, multiple appenders and async writing.
**19.** Design a **vending machine** or **ATM**.
**20.** Design a **notification system** (use 5.3 as your base and be ready to extend it).

For each, rehearse: requirements (5 min), entities and patterns (10 min), core code (20 min), extension and concurrency (10 min).

### Final checklist before the interview

**Concepts**
- [ ] I can explain each SOLID principle in one sentence and show a violation.
- [ ] I can implement Singleton (holder or enum) and explain why DCL needs `volatile`.
- [ ] I can explain Factory Method vs Abstract Factory vs Builder.
- [ ] I can implement Strategy, Observer, Decorator, State and Chain from memory.
- [ ] I can say how State vs Strategy and Decorator vs Proxy differ.

**LLD technique**
- [ ] I start with clarifying questions and write down requirements and out-of-scope items.
- [ ] I use the noun-verb trick to find classes and methods.
- [ ] I name the **requirement** each pattern serves.
- [ ] I use immutable records, constructor validation and `long` cents for money.
- [ ] I inject dependencies and a `Clock` so my code is testable.
- [ ] I can state the shared mutable state and how I protect it.

**Communication**
- [ ] I narrate my thinking and state trade-offs.
- [ ] I treat hints as help.
- [ ] I finish with: how I would test it, what I would improve, and what I chose not to build.

### SOLID in simple words (final recap)

| Principle | Simple words                                  | Everyday picture                                       | Quick smell                          |
| --------- | --------------------------------------------- | ------------------------------------------------------ | ------------------------------------ |
| **S**     | One class, one job (one reason to change)     | Accountant, electrician and receptionist are 3 people  | A giant `Manager` class              |
| **O**     | Add new code instead of editing old code      | A power strip: plug in, do not rewire the house        | A growing `switch`                   |
| **L**     | A child must work wherever its parent works   | Ordering "a vehicle for six people" and getting a bike | Overrides that throw                 |
| **I**     | Small interfaces, only what the client needs  | A remote with five buttons, not eighty                 | Empty method implementations         |
| **D**     | Depend on interfaces, get them handed in      | A lamp plug and a standard socket                      | `new ConcreteClass()` in the logic   |

### Part 2 recap

| Topic                 | Remember it as                                                                                           |
| --------------------- | -------------------------------------------------------------------------------------------------------- |
| Strategy              | Swappable algorithm behind an interface. A lambda is often enough. Client chooses.                       |
| Observer              | Publisher does not know subscribers. Isolate failures, avoid leaks, make consumers idempotent.           |
| Decorator             | Same interface, wraps another. Stack features at run time. Order matters.                                |
| Notification system   | Channel interface + retry/logging decorators + selection strategy + result observers + injected deps.    |
| State                 | Behavior lives in the state. Persist the enum name. Make transitions atomic.                             |
| Chain                 | Each handler handles, modifies, or passes on. Can stop the chain. Decide what happens if nobody handles. |
| Beyond SOLID          | Compose over inherit, Demeter, Tell-Don't-Ask, DRY (knowledge, not text), KISS, YAGNI, Rule of Three.    |
| LLD playbook          | Clarify, noun-verb, patterns from requirements, core first, extend, concurrency, test, narrate.          |

### Where this leads next

You have now covered the full path: how an object lives in memory (Module 1), how to protect it (Module 2), how classes relate (Module 3), how one call runs different code (Module 4), and how to arrange it all so it stays easy to change (Module 5). The next step is practice: take the design prompts in 5.7, set a 45-minute timer, and talk through each one out loud.

Back to the [OOP Mastery hub](./index.md) · Previous: [Module 5, Part 1: SOLID & Creational Patterns](./solid-principles.md)