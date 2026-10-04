---
title: "Module 5 (Part 1): SOLID Principles & Design Patterns"
description: "Learn the five SOLID principles and the key creational design patterns in Java 17+, with bad code, refactors and interview questions for low-level design rounds."
author: ["name": "Rajendra Pancholi", "email": "rpancholi522@gmail.com"]
thumbnail: "/images/oop-solid-design-patterns.png"
tags: [OOP, Java, SOLID, Design-Patterns, Low-Level-Design]
keywords: ["SOLID principles in Java", "Singleton Factory Builder pattern", "Low level design interview", "Dependency inversion and open closed principle", "Design patterns for interviews"]
---

# OOP Mastery for MAANG Interviews: Module 5 (Part 1 of 2) — SOLID Principles & Creational Patterns

![SOLID Principles & Creational Patterns](/images/oop-solid-design-patterns.png)

**Language:** Java 17+ (records, `var`, `Optional`, `java.util.concurrent`). Your template left the language blank, and Java is what most low-level design (LLD) interviews use. If you want this module in C++ instead, tell me and I will port it using RAII, `std::call_once` and smart pointers.

All code is written inline. Money is always stored as `long` cents or `BigDecimal`, never `double`, because interviewers notice.

**This module is split in two messages because of its size:**

- **Part 1 (this file):** SOLID principles (5.1) and creational patterns (5.2: Singleton, Factory Method, Abstract Factory, Builder).
- **Part 2 (next):** behavioral and structural patterns (5.3: Strategy, Observer, Decorator, and the integrated Notification System), State and Chain of Responsibility (5.4), principles beyond SOLID (5.5), the LLD interview playbook (5.6), the mock interview bank (5.7), the final checklist, and the "SOLID in simple words" recap.

### How this module connects to Modules 1–4

In Modules 1–4 you learned how objects work: memory, protection, inheritance, and virtual calls. Module 5 asks a different question: **how do you arrange classes so that code stays easy to change?** Real programs change constantly. A design that is easy to change survives. A design that is hard to change breaks every time someone adds a feature.

Almost everything here is **polymorphism used on purpose** (Module 4). When you see an `interface` and several classes that implement it, you are looking at the same idea as a vtable, only now it is a tool for design.

### What is a "design principle" and what is a "design pattern"?

- A **principle** is a guideline for judging a design ("a class should have only one reason to change"). It tells you _what good looks like_.
- A **pattern** is a named, reusable solution to a problem that shows up again and again ("Singleton," "Strategy"). It gives you _a recipe_.

Principles tell you when your design is going wrong. Patterns are common fixes. Interviewers expect you to use both: name the principle that is being broken, then reach for the pattern that repairs it, and explain the trade-off.

### Words you will meet in this module

| Word                          | Simple meaning                                                                                              |
| ----------------------------- | ----------------------------------------------------------------------------------------------------------- |
| **Interface**                 | A list of methods a class promises to provide, with no body. Many classes can implement the same interface. |
| **Abstraction**               | Something general that hides details (an interface, or an abstract class).                                  |
| **Coupling**                  | How much one piece of code depends on another. Less is better.                                              |
| **Cohesion**                  | How closely the parts of a class belong together. More is better.                                           |
| **Dependency**                | Something a class needs in order to work (another class, a database, an email service).                     |
| **Dependency Injection (DI)** | Handing a class its dependencies from outside instead of letting it create them.                            |
| **Composition root**          | The single place near the start of the program that creates all objects and connects them.                  |
| **Immutable**                 | Cannot change after it is built (Module 2).                                                                 |
| **Thread-safe**               | Works correctly when many threads use it at once.                                                           |
| **Refactor**                  | Reshape code without changing what it does.                                                                 |

### What interviewers expect, by level

| Level | Expectation                                                                                                                                        |
| ----- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| SDE-1 | Name the principle and pattern, recognise violations, write a clean class hierarchy                                                                |
| SDE-2 | Choose a pattern for a requirement, justify it, handle concurrency basics, write testable code                                                     |
| SDE-3 | Argue trade-offs (when _not_ to use SOLID or a pattern), predict future change, discuss the memory model, extensibility, and distributed analogues |

---

## 5.1 SOLID Principles

### The idea in plain words

SOLID is five guidelines that all aim at one thing: making code **cheap to change**. The name is just the first letter of each rule:

| Letter | Rule                  | In one sentence                                           |
| ------ | --------------------- | --------------------------------------------------------- |
| **S**  | Single Responsibility | A class has one reason to change.                         |
| **O**  | Open/Closed           | Add features by adding new code, not by editing old code. |
| **L**  | Liskov Substitution   | A child class must work anywhere its parent works.        |
| **I**  | Interface Segregation | Keep interfaces small and focused.                        |
| **D**  | Dependency Inversion  | Depend on interfaces, and have dependencies handed in.    |

In interviews, knowing the definitions is the entry ticket. What shows seniority is knowing each principle's **cost and limits**: when following it too hard makes things worse.

For every principle below, the same pattern repeats: a plain-words explanation, **bad code**, a **refactor**, and then interview questions with answers.

---

### S: Single Responsibility Principle (SRP)

#### The idea in plain words

Imagine one employee who is the accountant, the electrician and the receptionist. If the tax law changes, the electrician's work is disturbed. If the phone system changes, the accounting is disturbed. Everything is tangled together.

**SRP says: give each class one job.** The precise version (from Robert Martin) is: **a module should have one reason to change**, meaning it answers to one _actor_ (one group of stakeholders). It does **not** mean "does only one thing."

If Finance (tax rules), DBAs (database schema) and UX (print layout) can each force edits to the same class, that class has three responsibilities and will be edited for three unrelated reasons.

#### Bad code

```java
// ❌ Three actors, three reasons to change, one class.
class Invoice {
    private final List<LineItem> items = new ArrayList<>();

    double calculateTotal() {                 // Actor: Finance (tax/discount rules)
        double total = 0;
        for (LineItem i : items) total += i.price * i.qty;
        return total * 1.18;                  // hard-coded tax
    }

    void saveToDatabase() {                   // Actor: DBA / platform team
        // JDBC code, SQL strings, connection handling...
    }

    void print() {                            // Actor: UX / Marketing
        System.out.println("INVOICE total=" + calculateTotal());
    }
}
```

Problems:

- Changing the print layout forces re-testing the tax logic.
- You cannot unit-test `calculateTotal()` without a database on the classpath.
- Two developers editing for different reasons collide in the same file.
- `double` for money is a bug waiting to happen.

#### Refactored (MAANG-ready)

```java
import java.math.BigDecimal;
import java.util.List;
import java.util.Objects;
import java.util.Optional;

/** Value object: immutable, self-validating. */
record LineItem(String sku, int quantity, BigDecimal unitPrice) {
    LineItem {                                   // compact canonical constructor = validation
        Objects.requireNonNull(sku);
        Objects.requireNonNull(unitPrice);
        if (quantity <= 0) throw new IllegalArgumentException("quantity must be > 0");
        if (unitPrice.signum() < 0) throw new IllegalArgumentException("price must be >= 0");
    }
    BigDecimal subtotal() { return unitPrice.multiply(BigDecimal.valueOf(quantity)); }
}

/** Domain entity: ONLY business data and invariants. Actor: domain/finance. */
final class Invoice {
    private final String id;
    private final List<LineItem> items;

    Invoice(String id, List<LineItem> items) {
        this.id = Objects.requireNonNull(id);
        this.items = List.copyOf(items);         // defensive copy -> true immutability
    }
    String id()            { return id; }
    List<LineItem> items() { return items; }     // safe: List.copyOf is unmodifiable

    BigDecimal subtotal() {
        return items.stream().map(LineItem::subtotal).reduce(BigDecimal.ZERO, BigDecimal::add);
    }
}

/** Tax is a separate axis of change (Actor: tax/compliance team). */
interface TaxPolicy { BigDecimal taxFor(BigDecimal subtotal); }

/** Persistence port (Actor: platform/DBA). The domain doesn't know JDBC exists. */
interface InvoiceRepository {
    void save(Invoice invoice);
    Optional<Invoice> findById(String id);
}

/** Presentation (Actor: UX). Add JsonInvoiceFormatter later without touching Invoice. */
interface InvoiceFormatter { String format(Invoice invoice, BigDecimal tax); }

final class TextInvoiceFormatter implements InvoiceFormatter {
    @Override public String format(Invoice inv, BigDecimal tax) {
        return "INVOICE %s | subtotal=%s | tax=%s".formatted(inv.id(), inv.subtotal(), tax);
    }
}

/** Application service: orchestrates, contains no business rules itself. */
final class InvoiceService {
    private final InvoiceRepository repo;
    private final TaxPolicy taxPolicy;
    private final InvoiceFormatter formatter;

    InvoiceService(InvoiceRepository repo, TaxPolicy taxPolicy, InvoiceFormatter formatter) {
        this.repo = repo; this.taxPolicy = taxPolicy; this.formatter = formatter;
    }

    String issue(Invoice invoice) {
        repo.save(invoice);
        return formatter.format(invoice, taxPolicy.taxFor(invoice.subtotal()));
    }
}
```

(A **record** is Java's short syntax for an immutable data class. The **compact canonical constructor** is the block named after the record, with no parameter list, which runs your validation before the fields are set.)

How to read the refactor: each class now answers to exactly one actor. `Invoice` holds data and business rules. `TaxPolicy` belongs to the tax team. `InvoiceRepository` belongs to the platform team. `InvoiceFormatter` belongs to UX. `InvoiceService` only connects them.

#### Interview Q&A

**Q1. (SDE-1) Is SRP "a class should have only one method"?** No. A class can have 20 methods if they all serve the same actor and change for the same reason. `String` has many methods and one responsibility: representing text.

**Q2. (SDE-2) How do you decide where to split?** Ask: "Who would request a change here, and would that change force unrelated code to be re-tested or redeployed?" Another test is to look for method groups that use disjoint subsets of the fields. Disjoint field usage means low cohesion, which suggests several classes hiding inside one.

**Q3. (SDE-3) What is the downside of SRP taken too far?** Class explosion and "ravioli code": hundreds of tiny classes, where following one use case means jumping through 15 files. SRP is a _response to observed change pressure_. Split when a second reason to change actually appears, or is highly predictable. Do not split speculatively.

**Q4. Does `Invoice` violate SRP by having `subtotal()`?** No. Computing a subtotal from its own items is core domain behavior (the Information Expert principle: put behavior with the data it needs). Tax is separate because its rules come from a different actor and vary by region.

---

### O: Open/Closed Principle (OCP)

#### The idea in plain words

Think of a power strip. When you buy a new gadget, you plug it in. You do not open the wall and rewire the house.

**OCP says: software should be open for extension, closed for modification.** New behavior is added by _writing new code_, not by editing code that is already tested and working. Every time you edit old, working code, you risk breaking something that used to be fine.

#### Bad code

```java
// ❌ Every new customer tier edits this method (risk of regressions in old tiers).
class DiscountCalculator {
    double apply(String customerType, double amount) {
        if (customerType.equals("REGULAR")) return amount * 0.95;
        else if (customerType.equals("PREMIUM")) return amount * 0.90;
        else if (customerType.equals("VIP")) return amount * 0.80;
        // Marketing wants "STUDENT" next sprint -> edit again, retest everything
        return amount;
    }
}
```

#### Refactored

```java
import java.math.BigDecimal;
import java.math.RoundingMode;
import java.util.List;

/** The abstraction is the "closed" seam. */
interface DiscountPolicy {
    boolean supports(String customerTier);
    BigDecimal apply(BigDecimal amount);
}

/** Reusable base: a flat percentage discount. */
abstract class PercentageDiscount implements DiscountPolicy {
    private final BigDecimal rate;                   // e.g. 0.10 = 10% off
    protected PercentageDiscount(String rate) { this.rate = new BigDecimal(rate); }

    @Override public BigDecimal apply(BigDecimal amount) {
        return amount.subtract(amount.multiply(rate)).setScale(2, RoundingMode.HALF_UP);
    }
}

final class RegularDiscount extends PercentageDiscount {
    RegularDiscount() { super("0.05"); }
    @Override public boolean supports(String tier) { return "REGULAR".equals(tier); }
}
final class PremiumDiscount extends PercentageDiscount {
    PremiumDiscount() { super("0.10"); }
    @Override public boolean supports(String tier) { return "PREMIUM".equals(tier); }
}
// 👉 Adding STUDENT = ONE new class. DiscountService is NOT touched.
final class StudentDiscount extends PercentageDiscount {
    StudentDiscount() { super("0.15"); }
    @Override public boolean supports(String tier) { return "STUDENT".equals(tier); }
}

final class DiscountService {
    private final List<DiscountPolicy> policies;     // injected (DI framework can auto-collect all beans)

    DiscountService(List<DiscountPolicy> policies) { this.policies = List.copyOf(policies); }

    BigDecimal applyDiscount(String tier, BigDecimal amount) {
        return policies.stream()
                .filter(p -> p.supports(tier))
                .findFirst()
                .map(p -> p.apply(amount))
                .orElse(amount);                     // unknown tier = no discount (explicit default)
    }
}
```

The key: `DiscountService` never mentions `REGULAR`, `PREMIUM` or `STUDENT`. It just asks each policy "do you support this tier?" To add `STUDENT`, you write one new class and add it to the list. The tested `DiscountService` is untouched.

#### Interview Q&A

**Q1. (SDE-1) Does OCP mean you can never edit a class?** No. Bug fixes are fine. OCP is about _adding features_ without modifying stable, tested code.

**Q2. (SDE-2) You still have to edit the place where policies are registered. Isn't that a modification?** Yes, and that is expected. OCP can never be satisfied completely. We push the modification to the edge (the **composition root** or the DI configuration), where it is trivial and low-risk, instead of the center (the business logic). With Spring, `List<DiscountPolicy>` is filled in automatically, so even that edit disappears.

**Q3. (SDE-3) OCP protects against change along one axis. What if you extend the wrong axis?** Good OCP depends on correctly predicting _which_ dimension varies. If the dimension that changes is "number of parameters," the abstraction leaks and every implementation changes anyway. This is why seniors practise the **Rule of Three**: tolerate the `if/else` the first time, and abstract once the pattern repeats (see YAGNI in 5.5).

Also note the **expression problem**: with polymorphism, new _types_ are easy but new _operations_ are hard. With a Visitor, or with `sealed` types plus pattern-matching `switch`, new _operations_ are easy but new _types_ are hard. OCP choices are always along one axis.

**Q4. What is the relation between OCP and the Strategy pattern?** Strategy is OCP's most common mechanical implementation: the varying algorithm is hidden behind an interface and swapped in.

---

### L: Liskov Substitution Principle (LSP)

#### The idea in plain words

If you order "a vehicle" from a rental company and they send you a bicycle, you can ride it, and that is fine. But if you ordered "a vehicle to carry six people" and they send a bicycle, something has broken, even though a bicycle is technically a vehicle.

**LSP says: if `S` is a subtype of `T`, you must be able to replace `T` objects with `S` objects without breaking the correctness of the program.**

LSP is about **behavior**, not just types. The compiler checks that signatures match. LSP checks that the _promises_ match. A child class can compile perfectly and still break LSP by behaving differently from what callers of the parent rely on.

#### The contract rules (Design by Contract)

| Rule               | A subtype must...                                                                           |
| ------------------ | ------------------------------------------------------------------------------------------- |
| Preconditions      | Not be **stronger** (it cannot demand more from callers)                                    |
| Postconditions     | Not be **weaker** (it cannot promise less to callers)                                       |
| Invariants         | Be preserved                                                                                |
| Exceptions         | Not throw new checked or unexpected exception types the parent did not declare              |
| History constraint | Not allow state changes the parent forbids (for example, making an immutable thing mutable) |
| Parameter types    | Be contravariant (accept the same or broader); return types covariant (same or narrower)    |

(A **precondition** is what must be true _before_ a method is called. A **postcondition** is what the method promises will be true _after_.)

#### Bad code (the classic)

```java
class Rectangle {
    protected int width, height;
    void setWidth(int w)  { this.width = w; }
    void setHeight(int h) { this.height = h; }
    int area() { return width * height; }
}

// Mathematically a square IS-A rectangle, but behaviourally it is NOT.
class Square extends Rectangle {
    @Override void setWidth(int w)  { this.width = w; this.height = w; }   // side effect!
    @Override void setHeight(int h) { this.width = h; this.height = h; }
}

class Client {
    static void resize(Rectangle r) {
        r.setWidth(5);
        r.setHeight(4);
        // Rectangle's contract: width and height are independent -> area == 20
        assert r.area() == 20 : "LSP violated, got " + r.area(); // Square gives 16 💥
    }
}
```

`Square` strengthens the invariant (`width == height`) and weakens the postcondition of `setWidth` (the height now changes too).

#### Refactored

```java
/** Model by BEHAVIOUR, not by real-world taxonomy. */
sealed interface Shape permits Rectangle, Square {
    int area();
}

record Rectangle(int width, int height) implements Shape {
    public int area() { return width * height; }
}

record Square(int side) implements Shape {
    public int area() { return side * side; }
}
// Immutability removes the entire class of setter-coupling bugs.
// Need a resize? Return a new object: rect.withWidth(5)
```

(A **`sealed`** interface lists exactly which classes may implement it, so the compiler knows the full set of types.)

#### A second, more realistic violation

```java
// ❌ ReadOnly "is-a" List that throws -> clients written against List break.
List<String> names = Collections.unmodifiableList(new ArrayList<>(List.of("a")));
names.add("b");   // UnsupportedOperationException: the JDK itself bends LSP here
```

Better design: split the capabilities (see ISP below) into `ReadableCollection` and `MutableCollection extends ReadableCollection`.

#### Interview Q&A

**Q1. (SDE-1) Why does Square extending Rectangle break LSP even though a square is a rectangle?** Inheritance is about _behavioral substitutability_, not real-world "is-a." With mutable setters, Rectangle promises independent width and height, and Square cannot keep that promise.

**Q2. (SDE-2) Name code smells that signal an LSP violation.**

- A subclass overrides a method to throw `UnsupportedOperationException` or to do nothing.
- Client code does `instanceof` or casts to check which subtype it received.
- An override ignores its parameters or silently does something different.
- Overrides that need a comment like "don't call this on subtype X".

**Q3. (SDE-2) Does this violate LSP?**

```java
class Bird { void fly() {} }
class Penguin extends Bird { @Override void fly() { throw new UnsupportedOperationException(); } }
```

Yes. Fix: `Bird` has no `fly()`. Introduce `interface Flyable { void fly(); }` and implement it in `Sparrow`, not in `Penguin`.

**Q4. (SDE-3) How does LSP relate to generics and variance?** Java arrays are covariant (`Object[] a = new String[1]; a[0] = 1;` compiles and throws `ArrayStoreException`), which is an LSP-style hole at run time. Generics are invariant for the same reason, and PECS ("Producer Extends, Consumer Super") restores safe substitutability: `List<? extends Shape>` for reading, `List<? super Square>` for writing.

**Q5. (SDE-3) How do you enforce LSP in a large codebase?** Write **abstract contract tests**: an abstract test class that exercises the interface's contract, which every implementation's test class must extend and pass. Also use immutability, sealed hierarchies, and prefer composition over inheritance (5.5).

---

### I: Interface Segregation Principle (ISP)

#### The idea in plain words

Imagine a TV remote with 80 buttons when you only ever use five. You do not need a remote that complex, and if the manufacturer changes one obscure button, you still have to deal with the new remote.

**ISP says: no client should be forced to depend on methods it does not use.** Prefer several small interfaces, each for one role, over one big "fat" interface.

#### Bad code

```java
// ❌ Fat interface
interface MultiFunctionDevice {
    void print(Document d);
    void scan(Document d);
    void fax(Document d);
}

class BasicPrinter implements MultiFunctionDevice {
    public void print(Document d) { /* works */ }
    public void scan(Document d)  { throw new UnsupportedOperationException(); } // LSP smell too
    public void fax(Document d)   { throw new UnsupportedOperationException(); }
}
```

Changing `fax()`'s signature now recompiles and re-releases `BasicPrinter`, which never faxed.

#### Refactored

```java
interface Printer { void print(Document d); }
interface Scanner { void scan(Document d); }
interface Fax     { void fax(Document d); }

/** Compose roles only when a real device needs several. */
interface Copier extends Printer, Scanner { }

final class BasicPrinter implements Printer {
    @Override public void print(Document d) { /* ... */ }
}
final class OfficeMachine implements Printer, Scanner, Fax {
    @Override public void print(Document d) { /* ... */ }
    @Override public void scan(Document d)  { /* ... */ }
    @Override public void fax(Document d)   { /* ... */ }
}

/** Client depends on the narrowest role it needs -> trivially mockable. */
final class ReportService {
    private final Printer printer;
    ReportService(Printer printer) { this.printer = printer; }
    void publish(Document d) { printer.print(d); }
}
```

**Real-world ISP in Java:** `Runnable`, `Comparable`, `Closeable` and `Iterable` are tiny single-method role interfaces. Repository design commonly splits `ReadRepository<T>` from `WriteRepository<T>` (a CQRS-flavored split, so read replicas and caches only implement the read side).

#### Interview Q&A

**Q1. (SDE-1) ISP vs SRP?** SRP is about a _class's_ reasons to change. ISP is about the _client's view_ of an abstraction. A class with one responsibility can still expose a bloated interface, and ISP trims it per client.

**Q2. (SDE-2) What is the cost of over-segregation?** Interface proliferation, and harder discovery. Group by **client role**, not by single method. Splitting `List` into 30 one-method interfaces would be absurd.

**Q3. (SDE-3) How does ISP apply beyond classes (APIs, microservices)?** Fat REST or gRPC endpoints force every consumer to deal with fields and operations they do not need, so a change for one consumer ripples to all. Remedies are the **Backend-for-Frontend** pattern, GraphQL field selection, and versioned, narrow service contracts. It is the same idea at a larger scale.

**Q4. Java `default` methods: a fix or a cheat?** They allow interface evolution without breaking implementors (backward compatibility), but they do not fix fat interfaces. They hide the coupling instead of removing it.

---

### D: Dependency Inversion Principle (DIP)

#### The idea in plain words

A lamp has a plug. The plug fits a standard wall socket. The lamp does _not_ have the wire soldered directly into the power station. Because both the lamp and the power station agree on the **socket standard**, you can move the lamp to any house and swap the power station without touching the lamp.

**DIP says two things:**

1. High-level modules (your business rules) should not depend on low-level modules (databases, email services). **Both should depend on abstractions** (interfaces).
2. Abstractions should not depend on details. **Details should depend on abstractions.**

The "inversion" is that the low-level code now has to fit the interface that the high-level code defined, instead of the high-level code bending to fit the low-level code.

#### Three often-confused terms (a classic interview trap)

| Term                           | Meaning                                                                                                                                     |
| ------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------- |
| **DIP**                        | The _principle_: depend on abstractions, and the abstraction is _owned by the high-level module_                                            |
| **DI** (Dependency Injection)  | A _technique_: pass dependencies in (constructor, setter or method) instead of using `new` inside the class                                 |
| **IoC** (Inversion of Control) | A _broader idea_: a framework or caller controls the flow (the Hollywood Principle: "don't call us, we'll call you"). DI is one form of IoC |

#### Bad code

```java
// ❌ High-level policy (OrderService) welded to low-level details.
class OrderService {
    private final MySqlOrderRepository repo = new MySqlOrderRepository();
    private final SmtpEmailSender email = new SmtpEmailSender("smtp.corp.com");

    void placeOrder(Order o) {
        repo.insert(o);
        email.send(o.customerEmail(), "Thanks!");
    }
}
// Can't unit-test without MySQL + SMTP. Moving to Postgres/SES rewrites the business class.
```

#### Refactored

```java
import java.math.BigDecimal;

record Order(String id, String customerEmail, BigDecimal total) { }

// Abstractions are defined in terms of what the BUSINESS needs (owned by the high-level layer).
interface OrderRepository { void save(Order order); }
interface CustomerNotifier { void orderPlaced(Order order); }

/** High-level policy: zero knowledge of MySQL, SMTP, Kafka, etc. */
final class OrderService {
    private final OrderRepository repository;
    private final CustomerNotifier notifier;

    // Constructor injection: dependencies are explicit, final, and never null.
    OrderService(OrderRepository repository, CustomerNotifier notifier) {
        this.repository = java.util.Objects.requireNonNull(repository);
        this.notifier   = java.util.Objects.requireNonNull(notifier);
    }

    Order place(String id, String email, BigDecimal total) {
        if (total.signum() <= 0) throw new IllegalArgumentException("total must be positive");
        Order order = new Order(id, email, total);
        repository.save(order);
        notifier.orderPlaced(order);
        return order;
    }
}

// Low-level details now depend inward on the abstractions:
final class MySqlOrderRepository implements OrderRepository {
    @Override public void save(Order order) { /* JDBC */ }
}
final class SesCustomerNotifier implements CustomerNotifier {
    @Override public void orderPlaced(Order order) { /* AWS SES call */ }
}

/** COMPOSITION ROOT: the only place that knows concrete classes. */
public class Main {
    public static void main(String[] args) {
        var service = new OrderService(new MySqlOrderRepository(), new SesCustomerNotifier());
        service.place("o-1", "a@b.com", new BigDecimal("49.99"));
    }
}
```

**The payoff: a unit test with zero infrastructure.**

```java
class OrderServiceTest {
    @org.junit.jupiter.api.Test
    void savesAndNotifies() {
        var saved = new java.util.ArrayList<Order>();
        var notified = new java.util.ArrayList<Order>();

        // Single-method interfaces -> lambdas as fakes. No Mockito required.
        var service = new OrderService(saved::add, notified::add);

        service.place("o-1", "a@b.com", new BigDecimal("10"));

        org.junit.jupiter.api.Assertions.assertEquals(1, saved.size());
        org.junit.jupiter.api.Assertions.assertEquals(1, notified.size());
    }
}
```

Look at the test: `saved::add` is a plain method reference that acts as a fake `OrderRepository`. There is no database and no email server. That is what the inversion buys you.

#### Interview Q&A

**Q1. (SDE-1) Is DIP just dependency injection?** No. DI is a mechanism. DIP is about _who owns the abstraction_ and the direction of the source-code dependency. You can inject a concrete class with DI and still violate DIP.

**Q2. (SDE-2) Constructor vs setter vs field injection?** Constructor injection is preferred: dependencies are mandatory, the object is never half-built, fields can be `final` (thread-safe publication), and the dependencies are visible in the API. Setter injection suits optional dependencies. Field injection (`@Autowired` on a field) hides dependencies, blocks `final`, and complicates tests.

**Q3. (SDE-3) Where should the interface live: in the module of the implementation, or of the consumer?** In the **consumer's** package. This is the "inversion": the high-level module defines the port (`OrderRepository`), and the infrastructure adapter depends on it. This is the heart of Hexagonal and Clean Architecture. If the interface lives next to `MySqlOrderRepository`, the dependency direction is unchanged.

**Q4. (SDE-3) Downsides of DI containers?** Hidden wiring, startup-time failures instead of compile-time failures, reflection and proxy complexity, and "magic" that hinders debugging. For small services, manual wiring in a composition root is simpler and compile-safe.

**Q5. How does DIP relate to the Service Locator anti-pattern?** A Service Locator (`Locator.get(Repo.class)`) _looks_ like DIP but hides dependencies inside method bodies, so you cannot see what a class needs from its constructor, and tests need global setup. Prefer explicit injection.

---

### SOLID: Cross-Cutting Questions

**Q. Which SOLID principle is the "most important"?** There is no universal answer, but a good one: **DIP + SRP** give the most leverage (testability and isolation of change). OCP and LSP are _consequences_ of using abstractions correctly. ISP keeps those abstractions small.

**Q. Can SOLID principles conflict?** Yes, and discussing this signals seniority:

- **SRP vs. simplicity and cohesion:** over-splitting scatters logic.
- **OCP vs. YAGNI** ("you aren't gonna need it"): abstracting for extension too early adds complexity for change that never comes.
- **ISP vs. discoverability:** many tiny interfaces are harder to navigate.
- **DIP vs. performance:** interface dispatch plus indirection has a (usually negligible) cost, and hot loops in latency-critical code may deliberately use concrete types.

**Q. How would you detect SOLID violations in a code review?**

| Smell                                                                   | Likely violation |
| ----------------------------------------------------------------------- | ---------------- |
| Class name ending in `Manager`, `Util`, `Helper` with 1000 lines        | SRP              |
| Growing `switch` or `if-else` on a type code                            | OCP              |
| `instanceof` checks, `UnsupportedOperationException` overrides          | LSP              |
| Empty or throwing method implementations                                | ISP (+ LSP)      |
| `new ConcreteClass()` inside business logic; static calls to singletons | DIP              |

---

## 5.2 Creational Patterns

### The idea in plain words

SOLID told you _how to judge_ a design. Now we start the patterns, the named recipes. **Creational patterns** are about one question: **how are objects created?**

The problem: whenever your code writes `new SomeConcreteClass()`, it becomes glued to that exact class. If you later need a different class, you must edit every place that wrote `new`. Creational patterns move the creation step somewhere controlled, so the rest of the code is not coupled to concrete classes or construction details.

This part covers four of them: **Singleton** (exactly one instance), **Factory Method** (let a subclass decide what to create), **Abstract Factory** (create whole families of matching objects), and **Builder** (construct complex objects step by step).

---

### Singleton

#### The idea in plain words

A country has one government. A computer has one clock. Sometimes you really need **exactly one instance** of a class, with one agreed way to reach it.

**Intent:** ensure a class has exactly one instance and provide a global access point.

It sounds simple, and that is the trap. The interview questions are all about doing it **safely**: thread safety, lazy vs. eager creation, serialization, reflection, class loaders, and testability.

#### Evolution of implementations

Each version below fixes a weakness of the one before it.

```java
// V1: EAGER initialization. Simplest, inherently thread-safe.
// The JVM guarantees class initialization runs once, under a class-init lock.
// Downside: instance created even if never used (matters if construction is expensive).
final class EagerSingleton {
    private static final EagerSingleton INSTANCE = new EagerSingleton();
    private EagerSingleton() { }
    static EagerSingleton getInstance() { return INSTANCE; }
}

// V2: Lazy + synchronized method. Correct but SLOW:
// every call (not just the first) acquires the monitor.
final class SyncSingleton {
    private static SyncSingleton instance;
    private SyncSingleton() { }
    static synchronized SyncSingleton getInstance() {
        if (instance == null) instance = new SyncSingleton();
        return instance;
    }
}

// V3: Double-Checked Locking (DCL). Lazy, fast path is lock-free.
// `volatile` is MANDATORY, see explanation below.
final class ConfigManager {
    private static volatile ConfigManager instance;
    private final java.util.Map<String, String> props;

    private ConfigManager() {
        // Weak reflection guard (cannot fully stop reflection; enum can, see V5).
        if (instance != null) throw new IllegalStateException("Use getInstance()");
        this.props = java.util.Map.copyOf(loadFromDisk());   // final field + immutable map
    }

    static ConfigManager getInstance() {
        ConfigManager local = instance;            // single volatile read on the fast path
        if (local == null) {                       // 1st check: no lock
            synchronized (ConfigManager.class) {
                local = instance;                  // 2nd check: under lock
                if (local == null) {
                    instance = local = new ConfigManager();
                }
            }
        }
        return local;
    }

    String get(String key) { return props.get(key); }
    private static java.util.Map<String, String> loadFromDisk() { return java.util.Map.of("env", "prod"); }
}

// V4: Initialization-on-demand holder (Bill Pugh).
// Lazy + thread-safe + NO explicit synchronization.
// Holder class loads (and INSTANCE is built) only when getInstance() first runs;
// the JVM class-initialization lock provides the safety.
final class HolderSingleton {
    private HolderSingleton() { }
    private static final class Holder { static final HolderSingleton INSTANCE = new HolderSingleton(); }
    static HolderSingleton getInstance() { return Holder.INSTANCE; }
}

// V5: Enum Singleton (Effective Java, Item 3).
// Immune to reflection attacks and serialization duplication; JVM-guaranteed single instance.
// Limit: cannot extend another class, and eager (loads with the enum class).
enum MetricsRegistry {
    INSTANCE;
    private final java.util.concurrent.atomic.LongAdder requests = new java.util.concurrent.atomic.LongAdder();
    void recordRequest() { requests.increment(); }     // LongAdder: low-contention counter
    long total() { return requests.sum(); }
}
```

A quick tour of the five versions:

- **V1 (eager):** the instance is built when the class loads. Simple and safe, but wasteful if you never use it.
- **V2 (synchronized method):** built only when first needed, but _every_ call takes a lock, even long after the instance exists. Slow on hot paths.
- **V3 (double-checked locking):** checks without a lock first (fast path), and only locks if the instance might not exist yet. Fast, but needs `volatile` to be correct (explained next).
- **V4 (holder idiom):** lazy and thread-safe with no locks in your code at all. The JVM's own class-loading rules do the work.
- **V5 (enum):** the simplest safe version. The JVM guarantees one instance, even against reflection and serialization tricks.

#### Why `volatile` in DCL? (the low-level mechanics)

`instance = new ConfigManager()` is **not atomic**. Conceptually it is three steps:

```
1. allocate memory
2. run constructor (initialise fields)
3. assign reference to `instance`
```

The compiler or CPU may **reorder steps 2 and 3**. Thread A could publish the reference _before_ the constructor finishes. Thread B then sees `instance != null`, skips the lock, and uses a **partially constructed object**.

`volatile` (Java 5+ memory model) puts a **happens-before** edge on the write and read, and prevents that reordering. (Happens-before is the Java memory model's guarantee that one action's results are visible to another.) `final` fields in the constructor also get special freeze semantics (Module 2), but you should not rely on them alone here.

#### Breaking singletons (and defences)

```java
// 1) Reflection: Constructor<?> c = X.class.getDeclaredConstructor(); c.setAccessible(true); c.newInstance();
//    Defence: throw from constructor if instance exists, or use an enum.

// 2) Serialization: deserializing creates a NEW object.
//    Defence: implement readResolve(), or use an enum.
private Object readResolve() { return getInstance(); }

// 3) Cloning: don't implement Cloneable; or override clone() to throw.

// 4) Multiple ClassLoaders (app servers, OSGi): one "singleton" per loader.
//    Singleton means singleton PER CLASSLOADER.

// 5) Multiple JVMs / microservice replicas: a "global" singleton is per-process.
//    Distributed uniqueness needs a lock/leader election (ZooKeeper, etcd, Redis lock).
```

#### Quick concurrency proof-of-correctness test

```java
import java.util.Set;
import java.util.concurrent.*;

public class SingletonStressTest {
    public static void main(String[] args) throws Exception {
        int threads = 200;
        ExecutorService pool = Executors.newFixedThreadPool(threads);
        CountDownLatch startGun = new CountDownLatch(1);          // release all threads simultaneously
        Set<Integer> identities = ConcurrentHashMap.newKeySet();

        for (int i = 0; i < threads; i++) {
            pool.submit(() -> {
                try { startGun.await(); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
                identities.add(System.identityHashCode(ConfigManager.getInstance()));
            });
        }
        startGun.countDown();
        pool.shutdown();
        pool.awaitTermination(5, TimeUnit.SECONDS);
        System.out.println("Distinct instances: " + identities.size());   // must be 1
    }
}
```

(Caveat: a passing test does not _prove_ thread safety, because races are probabilistic. Reason about the memory model, and use jcstress for rigorous testing.)

#### Interview Q&A

**Q1. (SDE-1) What is a Singleton, and one real use?** A class with a single instance and global access. Real uses: a config registry, a connection or thread pool manager, a logger, a metrics registry.

**Q2. (SDE-2) Compare the 5 implementations. Which would you choose?** Default to the **enum** (safest) or the **holder idiom** (lazy, simplest when you cannot use an enum). Use DCL only when you need lazy loading with custom initialization and you understand `volatile`. Avoid V2 on hot paths.

**Q3. (SDE-2) Why is Singleton considered an anti-pattern by many?**

- It is global mutable state, which creates hidden coupling and order-of-initialization bugs.
- It violates SRP (it manages its own lifecycle as well as its business logic).
- It hinders testing, since you cannot swap it for a fake, and tests leak state into each other.
- It hides dependencies, because the constructor does not reveal that a class needs it.

**Q4. (SDE-3) What do you use instead?** Make the class a normal class and let the **DI container manage its scope** (`@Singleton` or `@Scope("singleton")`), or wire a single instance in the composition root. You get one instance without global access. That is "singleton lifetime without the Singleton pattern."

**Q5. (SDE-3) Design a thread-safe Singleton that takes constructor parameters.** The holder idiom cannot take arguments. Options: (a) DCL with an `init(config)` that fails on a second call, (b) an `AtomicReference` with `compareAndSet`, or (c) wire it in the composition root and inject it.

**Q6. Is a `static` utility class the same as a Singleton?** No. A static class cannot implement an interface, cannot be passed as a dependency or mocked easily, and has no instance lifecycle. A Singleton is a real object that can implement interfaces and be swapped.

---

### Factory Method

#### The idea in plain words

You walk into a restaurant and say "I'll have the soup of the day." You do not go into the kitchen and pick the ingredients. Whoever is running that kitchen decides _which_ soup you get. You just receive a bowl of soup and eat it.

**Intent:** define an interface for creating an object, but let **subclasses decide which class to instantiate**. The creator uses the product only through its abstraction, so it never has to say `new EmailChannel()`.

#### Scenario: notification delivery

```java
/** Product abstraction. */
interface Channel {
    void deliver(String recipient, String message) throws DeliveryException;
}

class DeliveryException extends Exception {
    DeliveryException(String msg, Throwable cause) { super(msg, cause); }
}

/**
 * CREATOR. Contains the stable workflow (template) and delegates the one
 * varying step (creation) to the abstract factory method.
 */
abstract class NotificationSender {

    /** FACTORY METHOD: subclasses choose the concrete Channel. */
    protected abstract Channel createChannel();

    /** Stable algorithm: validation + audit never change when new channels appear. */
    public final void send(String recipient, String message) throws DeliveryException {
        if (recipient == null || recipient.isBlank()) throw new IllegalArgumentException("recipient");
        Channel channel = createChannel();
        long start = System.nanoTime();
        try {
            channel.deliver(recipient, message);
        } finally {
            System.out.printf("[audit] %s delivered in %d µs%n",
                    channel.getClass().getSimpleName(), (System.nanoTime() - start) / 1_000);
        }
    }
}

final class EmailChannel implements Channel {
    @Override public void deliver(String to, String msg) { System.out.println("EMAIL -> " + to + ": " + msg); }
}
final class SmsChannel implements Channel {
    @Override public void deliver(String to, String msg) { System.out.println("SMS -> " + to + ": " + msg); }
}

final class EmailSender extends NotificationSender {
    @Override protected Channel createChannel() { return new EmailChannel(); }
}
final class SmsSender extends NotificationSender {
    @Override protected Channel createChannel() { return new SmsChannel(); }
}
```

In `NotificationSender.send`, the validation and the audit log are the _stable_ part that never changes. The one varying step, "which channel?", is a single abstract method `createChannel()`. Each subclass fills in just that one blank.

#### Practical variant: registry-based factory (OCP-friendly, thread-safe)

Interviewers often prefer this in modern codebases, because it avoids a subclass per product and avoids a growing `switch`.

```java
import java.util.Map;
import java.util.Optional;
import java.util.concurrent.ConcurrentHashMap;
import java.util.function.Supplier;

final class ChannelFactory {
    // ConcurrentHashMap -> safe concurrent registration and lookup, no external locking.
    private final Map<String, Supplier<Channel>> registry = new ConcurrentHashMap<>();

    /** Plugins/modules register themselves at startup; the factory code never changes. */
    ChannelFactory register(String type, Supplier<Channel> supplier) {
        if (registry.putIfAbsent(type.toUpperCase(), supplier) != null) {
            throw new IllegalStateException("Duplicate channel type: " + type);
        }
        return this;
    }

    Channel create(String type) {
        return Optional.ofNullable(registry.get(type.toUpperCase()))
                .map(Supplier::get)                  // Supplier => a fresh instance per call (or return a cached one)
                .orElseThrow(() -> new IllegalArgumentException("Unknown channel: " + type));
    }
}

// Usage:
// var factory = new ChannelFactory().register("email", EmailChannel::new).register("sms", SmsChannel::new);
// factory.create("sms").deliver("+49...", "hi");
```

(A **`Supplier<Channel>`** is just "a thing that can give me a new Channel when asked." `EmailChannel::new` is shorthand for "call the constructor.")

#### Terminology (very commonly confused)

| Name                                | What it is                                                                                      |
| ----------------------------------- | ----------------------------------------------------------------------------------------------- |
| **Simple Factory / Static Factory** | A single class with a `create(type)` method. Not a GoF pattern, but the most common in practice |
| **Factory Method (GoF)**            | Inheritance-based: a subclass overrides the creation method                                     |
| **Abstract Factory (GoF)**          | Object-composition based: a factory object that creates _families_ of products                  |

(**GoF** stands for "Gang of Four," the authors of the classic book that named these patterns.)

#### Interview Q&A

**Q1. (SDE-1) Factory Method vs `new`?** `new` hardcodes the concrete class (it violates DIP and OCP). Factory Method centralizes creation so the client depends on an interface.

**Q2. (SDE-2) Factory Method vs Simple Factory?** A Simple Factory is one class with a conditional that you edit for each new product (it violates OCP). Factory Method defers the choice to subclasses. The registry variant above achieves OCP without subclassing.

**Q3. (SDE-2) When would you avoid factories?** When there is only one implementation with no foreseeable variation, a factory is dead weight. Also avoid them if a lambda or constructor reference (`EmailChannel::new`) is enough.

**Q4. (SDE-3) Thread safety of factories?** Stateless factories are inherently thread-safe. The risk is _shared mutable state_: a registry (use `ConcurrentHashMap`), cached products (must be immutable or thread-safe), or counters. Decide explicitly whether the factory returns a **new instance per call** (prototype scope) or a **shared instance** (flyweight or singleton scope), and document it.

**Q5. How does a DI container relate to factories?** A container is essentially a giant, configurable factory (`BeanFactory`) with lifecycle and scope management.

---

### Abstract Factory

#### The idea in plain words

Furniture comes in matching sets. A "Modern" set has a modern chair and a modern sofa. A "Victorian" set has a Victorian chair and a Victorian sofa. You never want a modern chair with a Victorian sofa.

**Intent:** provide an interface for creating **families of related objects** without specifying their concrete classes. The key guarantee: products that come from one factory are **compatible with each other**.

#### Scenario: a data pipeline that must run on AWS _or_ GCP

Mixing an S3 storage with a Pub/Sub queue would be an inconsistent configuration. Abstract Factory makes that mix impossible to write.

```java
import java.util.EnumMap;
import java.util.Map;

// Product families (abstract)
interface BlobStorage  { void put(String key, byte[] data); }
interface MessageQueue { void publish(String topic, String message); }

// ABSTRACT FACTORY
interface CloudFactory {
    BlobStorage createStorage();
    MessageQueue createQueue();
}

// Family 1: AWS
final class S3Storage implements BlobStorage {
    @Override public void put(String key, byte[] data) { System.out.println("S3 PUT " + key); }
}
final class SqsQueue implements MessageQueue {
    @Override public void publish(String t, String m) { System.out.println("SQS -> " + t + ": " + m); }
}
final class AwsFactory implements CloudFactory {
    @Override public BlobStorage createStorage() { return new S3Storage(); }
    @Override public MessageQueue createQueue()  { return new SqsQueue(); }
}

// Family 2: GCP
final class GcsStorage implements BlobStorage {
    @Override public void put(String key, byte[] data) { System.out.println("GCS PUT " + key); }
}
final class PubSubQueue implements MessageQueue {
    @Override public void publish(String t, String m) { System.out.println("PubSub -> " + t + ": " + m); }
}
final class GcpFactory implements CloudFactory {
    @Override public BlobStorage createStorage() { return new GcsStorage(); }
    @Override public MessageQueue createQueue()  { return new PubSubQueue(); }
}

// Provider selection: built once, immutable, thread-safe
enum CloudProvider { AWS, GCP }

final class CloudFactories {
    private static final Map<CloudProvider, CloudFactory> FACTORIES;
    static {
        EnumMap<CloudProvider, CloudFactory> m = new EnumMap<>(CloudProvider.class);
        m.put(CloudProvider.AWS, new AwsFactory());
        m.put(CloudProvider.GCP, new GcpFactory());
        // Published safely via static initialiser (class-init happens-before first use) + unmodifiable view.
        FACTORIES = java.util.Collections.unmodifiableMap(m);
    }
    private CloudFactories() { }
    static CloudFactory of(CloudProvider p) { return FACTORIES.get(p); }
}

// Client: knows ONLY abstractions
final class DataPipeline {
    private final BlobStorage storage;
    private final MessageQueue queue;

    DataPipeline(CloudFactory factory) {          // one factory => consistent family guaranteed
        this.storage = factory.createStorage();
        this.queue   = factory.createQueue();
    }

    void ingest(String key, byte[] payload) {
        storage.put(key, payload);
        queue.publish("ingested", key);
    }
}
// new DataPipeline(CloudFactories.of(CloudProvider.GCP)).ingest("a.csv", new byte[0]);
```

Notice that `DataPipeline` receives _one_ `CloudFactory` and asks it for both the storage and the queue. Because both come from the same factory, they are guaranteed to be a matching pair.

#### Interview Q&A

**Q1. (SDE-1) Abstract Factory vs Factory Method?** Factory Method creates **one** product via inheritance. Abstract Factory creates **a family** of related products via composition (the factory is an object passed in).

**Q2. (SDE-2) What is the biggest drawback?** Adding a **new product type** (say `createCache()`) forces changes to the factory interface and to _all_ concrete factories. It is OCP-friendly for adding new **families**, and OCP-hostile for adding new **products**. Mitigation: group products meaningfully, or use default methods or a registry for optional products.

**Q3. (SDE-2) Give three real examples.**

- `javax.xml.parsers.DocumentBuilderFactory`
- JDBC's `Connection` (it creates `Statement` and `PreparedStatement`, all consistent with the driver)
- Cross-platform UI toolkits (a Windows or Mac button and checkbox)

**Q4. (SDE-3) How would you make the provider choice configuration-driven and still thread-safe?** Read the config at startup, build an immutable `Map<Provider, CloudFactory>` (as above), and inject the chosen factory. Do not let factories hold mutable state. If factories must be lazy, use `ConcurrentHashMap.computeIfAbsent`, remembering that its mapping function must not modify the same map.

**Q5. (Tricky) Does Abstract Factory violate SRP, since it creates many things?** No. The _reason to change_ is one: "which family are we targeting?" The products are cohesive by family.

---

### Builder (added)

#### The idea in plain words

Ordering a custom sandwich: bread, then cheese, then sauce, then toppings. You say each choice, and at the end the counter assembles it and hands it over. You do not shout twelve unlabeled options in a single sentence.

**Intent:** construct a complex object **step by step**, with validation, producing an **immutable** result. It solves two classic problems:

- The **telescoping constructor** (`new X(1, null, false, 3)`, where nobody can tell what the numbers mean).
- **Half-initialized JavaBeans** (objects built with many setters, which can be left in an invalid state).

```java
import java.time.Duration;
import java.util.Map;
import java.util.Objects;
import java.util.TreeMap;

final class HttpRequestSpec {
    // All fields final -> immutable -> safely shareable across threads without locks.
    private final String method;
    private final String url;
    private final Map<String, String> headers;
    private final byte[] body;
    private final Duration timeout;

    private HttpRequestSpec(Builder b) {                // private: only the Builder can construct
        this.method  = b.method;
        this.url     = b.url;
        this.headers = Map.copyOf(b.headers);           // defensive copy
        this.body    = b.body == null ? new byte[0] : b.body.clone();
        this.timeout = b.timeout;
    }

    String method() { return method; }
    String url() { return url; }
    Map<String, String> headers() { return headers; }
    byte[] body() { return body.clone(); }              // defensive copy on read
    Duration timeout() { return timeout; }

    static Builder builder(String url) { return new Builder(url); }

    static final class Builder {
        private final String url;                       // required -> constructor param
        private String method = "GET";                  // optional with default
        private final Map<String, String> headers = new TreeMap<>();
        private byte[] body;
        private Duration timeout = Duration.ofSeconds(30);

        private Builder(String url) { this.url = Objects.requireNonNull(url, "url"); }

        Builder method(String m)        { this.method = m; return this; }
        Builder header(String k, String v) { headers.put(k, v); return this; }
        Builder body(byte[] b)          { this.body = b; return this; }
        Builder timeout(Duration d)     { this.timeout = d; return this; }

        HttpRequestSpec build() {
            // Cross-field validation happens ONCE, here, so an invalid object can never exist.
            if (body != null && method.equals("GET")) {
                throw new IllegalStateException("GET must not have a body");
            }
            if (timeout.isNegative() || timeout.isZero()) {
                throw new IllegalStateException("timeout must be positive");
            }
            return new HttpRequestSpec(this);
        }
    }
}
// var req = HttpRequestSpec.builder("https://api.x.com").method("POST").header("A","1").body(data).build();
```

Notice the three ideas from Module 2 working together here: the product's constructor is private (only the Builder can create one), all fields are `final` (immutable), and `build()` validates everything once, so an invalid `HttpRequestSpec` can never exist. Each setter returns `this`, which is the chaining trick from Module 1, Topic 1.5.

#### Interview Q&A

**Q1. Builder vs telescoping constructors vs JavaBean setters?** Telescoping constructors are unreadable (`new X(1, null, false, 3)`). Setters allow an inconsistent half-built state and prevent immutability. A Builder gives readable calls, validation at `build()`, and an immutable product.

**Q2. (SDE-3) Is a Builder thread-safe?** Typically no, and it does not need to be: it is a short-lived, thread-confined helper. The _product_ is immutable and therefore thread-safe, which is the point.

**Q3. Alternatives?** Java records with a compact constructor for small cases, static factory methods with named parameters, or Lombok's `@Builder`.

---

### Part 1 recap

| Topic            | Remember it as                                                                                         |
| ---------------- | ------------------------------------------------------------------------------------------------------ |
| SRP              | One class, one reason to change (one actor). Split when a second reason really appears.                |
| OCP              | Add by adding classes. Push the unavoidable edit to the composition root.                              |
| LSP              | Substitutability is about behavior and contracts, not types. Smells: `instanceof`, throwing overrides. |
| ISP              | Small role-based interfaces. Fat interfaces force useless dependencies.                                |
| DIP              | Depend on abstractions owned by the high-level module. DI is the technique, IoC the broad idea.        |
| Singleton        | Prefer enum or holder. DCL needs `volatile`. In real systems prefer DI-managed scope.                  |
| Factory Method   | Subclass decides what to create. The registry variant avoids a subclass per product.                   |
| Abstract Factory | A family of matching products from one factory. Easy to add families, hard to add product types.       |
| Builder          | Step-by-step, validated, immutable result. The Builder itself is thread-confined.                      |
