---
title: "Module 2: Encapsulation & Data Hiding"
description: "Understand why private is a compiler rule and not security, and learn to protect class invariants, build truly immutable objects and avoid leaky getters in C++17."
author: ["name": "Rajendra Pancholi", "email": "rpancholi522@gmail.com"]
thumbnail: "/images/oop-encapsulation.png"
tags: [OOP, C++, Encapsulation, Immutability, PIMPL]
keywords: ["Encapsulation in C++", "Class invariants", "Immutable objects const correctness", "PIMPL idiom and ABI stability", "Access modifiers public private protected"]
---

# OOP Mastery for MAANG Interviews: Module 2 — Encapsulation & Data Hiding

![Encapsulation & Data Hiding](/images/oop-encapsulation.png)

### How this module connects to Module 1

Module 1 showed you _how an object sits in memory_ and _how it is born, copied, moved and destroyed_. Module 2 asks the next question: **who is allowed to touch the object, and how do we stop its rules from being broken?**

That is all "encapsulation" means. You put data and the rules for changing it in one box, and you lock the box so that every change must go through the rules.

### Words you will meet in this module

| Word                           | Simple meaning                                                                                            |
| ------------------------------ | --------------------------------------------------------------------------------------------------------- |
| **Access specifier**           | `public`, `protected`, `private`: keywords that say who may use a member.                                 |
| **Invariant**                  | A rule that must always be true for an object (for example "balance is never negative").                  |
| **Mutator**                    | Any function that changes an object's state.                                                              |
| **Static type / dynamic type** | Static = the type written in the code. Dynamic = the real type of the object at run time.                 |
| **Data race**                  | Two threads touch the same memory at the same time and at least one writes. This is UB in C++.            |
| **TOCTOU**                     | "Time of check, time of use": data is checked, then changed by someone else before you use it.            |
| **View**                       | A lightweight pointer-plus-length that looks at data it does not own (`string_view`, `span`).             |
| **ABI**                        | The binary-level agreement (sizes, layouts, call rules) that lets separately compiled code work together. |
| **Snapshot**                   | A frozen copy of data at one moment in time.                                                              |

## Topic 2.1: Access Modifiers (public / protected / private)

### The idea in plain words

Think of a house. The **front door and living room** are open to guests (`public`). The **family rooms** are open only to family (`protected`: the class and its children). The **bedroom** is only for you (`private`: only the class itself).

In code:

- `public`: anyone can use it.
- `protected`: the class, its friends, and classes that inherit from it.
- `private`: the class and its friends only.
- A `class` starts as private by default. A `struct` starts as public (for members and for inheritance).

Here is the most important thing to understand, and many beginners get it wrong: **`private` is a polite rule for the compiler, not a lock on the memory.** It stops you from _writing_ code that names a private member. It does not hide the bytes. This module will show you exactly what that means.

### How it works inside

- **Enforcement is purely compile-time.** The compiler checks access on _names_ while it analyzes your code. The machine code that comes out has no access bits at all. A `private` field is just `[this + 8]`, no different from a public one. There is no run-time check, no memory-protection hardware, and no tag in the object.
- **Access is per class, not per object.** Any member function of `T` can touch the private members of _any_ `T` object (for example `other.cents_`). Java works the same way. Ruby and Smalltalk are per object.
- **Overload resolution runs BEFORE access checking.** The compiler first picks the best matching function among _all_ candidates, including private ones, and only then checks access on the winner. So a private exact match beats a public one that needs a conversion, and you get an error.
- **Access is checked against the static type, never the dynamic type.** You may override a `public virtual` function with a `private` one. Calling `base->run()` still reaches it through the vtable. This is the basis of the **Non-Virtual Interface (NVI)** pattern.
- **`protected` has a hidden rule.** A derived class may use a protected base member only _through an object of the derived type_ (its own type or deeper), never through a `Base&`. Otherwise `Derived1` could damage the state of its sibling `Derived2`.
- **`friend`** gives a named class or function access. It is not inherited and not transitive. Some template tricks can even get around access checks, so `private` is **not a security mechanism**.
- **Layout:** access specifiers do not change layout in practice (members stay in declaration order). Before C++23 the standard left the order _between different access sections_ unspecified. Mixed access also makes a class **non-standard-layout**, which matters for C interop and `offsetof`.
- **Java and JVM comparison:** `javac` checks at compile time, and the **JVM checks again at link time** when it resolves a field or method reference (`IllegalAccessError`). Reflection with `setAccessible(true)` can bypass this, so Java 9+ added **JPMS** (the module system) with _strong encapsulation_: packages that are not `opens` throw `InaccessibleObjectException`, and JDK internals are sealed by default since Java 17. Since Java 11, **nestmates** (JEP 181) let inner and outer classes touch each other's private members directly. Before that, `javac` generated synthetic `access$000` helper methods. Java also has a fourth level, **package-private** (no keyword), and `protected` in Java _includes_ package access.

### Code: five access-control ideas in one file

```cpp
##include <cstring>
##include <iostream>
##include <type_traits>

// ---- 1. Access is per-CLASS, not per-OBJECT ----
class Account {
    long long cents_;                                   // private
public:
    explicit Account(long long c) : cents_(c) {}
    bool richerThan(const Account& other) const {       // legal: same class
        return cents_ > other.cents_;                   // touches ANOTHER object's private field
    }
};

// ---- 2. Overload resolution happens BEFORE the access check ----
class Overloaded {
    void f(int) {}                                      // private: exact match for an int arg
public:
    void f(double) {}                                   // public: needs a conversion
};
// Overloaded o; o.f(1);    // ERROR: overload resolution picks private f(int), THEN access fails
// o.f(1.0);                // OK
// (Same mechanism makes `= delete`d overloads useful for blocking implicit conversions.)

// ---- 3. Non-Virtual Interface: public non-virtual API, private virtual hook ----
class Exporter {
public:
    virtual ~Exporter() = default;
    void exportAll(std::ostream& os) {                  // the CONTRACT: pre/post-conditions,
        os << "BEGIN\n";                                // logging, locking live here, once.
        doExport(os);                                   // customization point
        os << "END\n";
    }
private:
    virtual void doExport(std::ostream& os) = 0;        // private virtual: overridable, not callable
};
class CsvExporter final : public Exporter {
    void doExport(std::ostream& os) override { os << "a,b,c\n"; }  // override is legal despite
};                                                                  // being private

// ---- 4. protected: access only through the derived type ----
class Base { protected: int p_ = 0; };
class Derived : public Base {
public:
    void ok(Derived& d) { d.p_ = 1; p_ = 2; }           // OK: via Derived
    // void bad(Base& b) { b.p_ = 1; }                  // ERROR: could poke a sibling's state
};

// ---- 5. Layout is untouched by access specifiers; standard-layout-ness is not ----
struct A { int x; int y; };
class  B { public: int x; private: int y; };
static_assert(sizeof(A) == sizeof(B), "same bytes in practice");
static_assert(std::is_standard_layout_v<A> && !std::is_standard_layout_v<B>, "mixed access");

int main() {
    Account a(42);
    long long peek;
    std::memcpy(&peek, &a, sizeof peek);   // well-defined (trivially copyable): "private" is NOT secret
    std::cout << peek << '\n';             // 42
    CsvExporter c; Exporter& e = c; e.exportAll(std::cout);
}
```

Read example 3 slowly, because it is a very useful design. The public function `exportAll` is the **contract**: it prints BEGIN and END and calls a hook in the middle. The hook `doExport` is private, so outsiders cannot call it, but subclasses can still _replace_ it. The base class keeps control of the order of steps and the subclass only fills in the variable part.

And look at `main`: it reads the "private" value `42` straight out of memory with `memcpy`. That is the proof that `private` is not secret.

### Real-world picture

Access modifiers are a **service's public API surface versus its internal network**. The API gateway (the compiler) rejects calls to internal endpoints at the edge. But if an attacker is already _inside the VPC_ (the same address space), nothing stops direct calls to the internal port.

So `private` is **API contract enforcement, not a security boundary**. Real isolation needs a process or sandbox boundary, which is like separate services with mutual TLS.

NVI is the **gateway pattern**: one public entry point owns authentication, logging and rate limiting, and the pluggable internal handlers cannot be called directly.

### Interview traps

- "Is `private` secure?" The trap is saying yes. It is a compile-time naming rule. Memory in the same process can be read through `memcpy`, pointer casts, debuggers, or Java reflection and `Unsafe`.
- **Overload-then-access puzzles:** a private exact-match overload takes over a call and then fails the access check.
- **Static vs. dynamic type:** private overrides of public virtual functions can still be called through a base pointer, so "private" does not mean "cannot be invoked."
- **`protected` through a base reference** causes compile errors. Also, protected _data_ becomes a public-like API for every subclass, forever. Prefer protected _functions_.
- `friend` does not break encapsulation if used for tight coupling (such as `operator<<` or test fixtures). It is not inherited and not transitive.
- Private inheritance means "implemented in terms of" (there is no `Base*` conversion outside the class). Prefer composition.
- Java: "Does a `private` method in a superclass get overridden?" No, it is not inherited. "`protected` vs. package-private?" "Can reflection read a private field? What stops it in Java 17?" (JPMS and `--add-opens`.)
  #- `#define private public` before an include is an ODR violation and UB. It may come up as a "hack" question. The right answer is "it is undefined," not "it works."

### Tricky questions and answers

#### Q1 \[SDE-2\]: Which lines compile, and what do the valid calls print?

```cpp
class Base {
public:
    virtual void run() { std::cout << "Base\n"; }
protected:
    int shared_ = 1;
private:
    int secret_ = 2;
};
class Mid : public Base {
    void run() override { std::cout << "Mid\n"; }     // (A)
public:
    void touch(Mid& m, Base& b) {
        m.shared_ = 3;        // (B)
        b.shared_ = 4;        // (C)
        secret_   = 5;        // (D)
    }
};
int main() {
    Mid m; Base* bp = &m;
    bp->run();                // (E)
    m.run();                  // (F)
}
```

**Answer:**

- **(A) compiles.** An override ignores the access level of the base declaration.
- **(B) compiles.** The access goes through a `Mid` object.
- **(C) error.** A protected base member cannot be reached through a `Base&`, because `b` might really be another derived class.
- **(D) error.** `secret_` is private to `Base`. Lookup finds the name, but access fails (private members are _inherited_ but _inaccessible_).
- **(E) prints `Mid`.** Access is checked against the static type `Base`, where `run` is public. Then dispatch goes through the vtable to `Mid::run`.
- **(F) error.** The static type is `Mid`, where `run` is private.
- Design smell: (A) plus (E) means one function is public through one type and private through another. It is legal, but it breaks Liskov-style expectations. Use it only on purpose (NVI).

#### Q2 \[SDE-3\]: "Private data is safe from other code, right?" Show the bypass and name the _real_ boundaries.

**Answer:** No. `private` is a compile-time _name_ restriction inside a single address space. The `main` above reads `Account::cents_` using `memcpy` (well-defined for trivially copyable types). A debugger, a core dump, `/proc/<pid>/mem`, or a memory-corruption bug can do the same. In Java, `Field.setAccessible(true)` or `Unsafe` does it, and deserialization even creates objects **without running constructors**.

What actually gives you a boundary:

1. **Process isolation** (separate processes, seccomp, containers, WASM sandboxes).
2. **Language module systems** (JPMS `exports` and `opens`, C++ modules or hidden visibility with `-fvisibility=hidden`).
3. **Opaque handles and PIMPL** (Topic 2.5): clients never receive the layout.
4. **Validation at trust boundaries:** validate again after deserialization (Topic 2.2).

Interview framing: _encapsulation protects invariants from well-meaning callers (bugs), not from adversaries._

## Topic 2.2: Class Invariants & Mutators

### The idea in plain words

Some things must **always be true** about an object. A bank account balance should never be negative. A date range should never have an end before its start. A rule like that is called a **class invariant**.

An invariant is true:

- right after the constructor finishes, and
- after every public function returns.

It is allowed to be briefly broken _inside_ a member function, while the function is working, as long as it is repaired before the function returns.

A **mutator** is any function that changes the object. The reason we hide data and expose functions is simple: if all changes go through a few functions, the invariant can be enforced **in one place**, not scattered across the whole program.

### How to protect an invariant (the rules, step by step)

- **Constructors establish the invariant or throw.** If a constructor throws, no object exists: the destructor does not run, and members that were already built are destroyed. So there are **no zombie objects**. Avoid `init()` or `isValid()` two-phase patterns, because they bring back "half-built" states.
- **Validate before you mutate (strong exception guarantee).** Check everything first. Then commit with operations that cannot fail (`noexcept` swaps, integer stores). If something fails, the object is unchanged.
- **Check, don't compute, near limits.** _Signed overflow is UB_, so `a + b > MAX` is wrong because the compiler may delete the check. Write `b > MAX - a`. Unsigned underflow wraps silently (`size_t(0) - 1` becomes a huge number), and mixed signed/unsigned arithmetic converts the signed value to unsigned first.
- **Setters are a smell.** Pairs like `setX`/`setY` expose in-between states that violate multi-field invariants. Prefer _intent-revealing_ operations (`deposit`, `reschedule(start, end)`) that move from one valid state to another in one step.
- **Defend against input aliasing (TOCTOU).** Take parameters _by value_ (or copy first) and validate **your own copy**, so the caller or another thread cannot change the data between your check and your use. Never keep a `string_view`, pointer or reference to caller memory as state.
- **Concurrency:** an invariant over two fields cannot be protected by two atomics. It needs one mutex, or one immutable snapshot. With `std::atomic<int> lo, hi`, another thread can still see `lo > hi`.
- **Debug checks vs. release checks:** `assert` documents _internal_ invariants and is compiled out with `-DNDEBUG`. **Never use `assert` to validate external input.** Throw an exception (or return an error type) instead.
- **Make illegal states impossible to represent:** strong types (`Percentage`, `Cents`), `enum class`, `std::optional`, and non-null references remove whole classes of checks.
- **Java comparison:** use `Objects.requireNonNull`, make defensive copies of mutable parameters (_before_ validating), and remember that **serialization bypasses constructors**, so `readObject` must re-validate (or use a serialization proxy or records). Frameworks like Jackson and Hibernate build objects through reflection, so invariants must live in the constructor (or the record's canonical constructor).

### Code: a bank account that guards its rules

```cpp
##include <cassert>
##include <cstdint>
##include <stdexcept>
##include <string>
##include <utility>

class Account {
    static constexpr std::int64_t kMaxCents = 1'000'000'000'000LL;   // hard cap: $10B

    std::string  id_;      // INVARIANT: non-empty
    std::int64_t cents_;   // INVARIANT: 0 <= cents_ <= kMaxCents

    // Internal self-check. Compiled out in release, so never a substitute for validation.
    void checkInvariant() const noexcept {
        assert(!id_.empty() && cents_ >= 0 && cents_ <= kMaxCents);
    }

public:
    // By-value 'id' => we own our copy; nobody can mutate it behind our back after validation.
    Account(std::string id, std::int64_t openingCents)
        : id_(std::move(id)), cents_(openingCents) {
        if (id_.empty())
            throw std::invalid_argument("Account: empty id");
        if (openingCents < 0 || openingCents > kMaxCents)
            throw std::out_of_range("Account: opening balance out of range");
        checkInvariant();            // if we got here the invariant holds; if we threw, no object exists
    }

    // Intent-revealing mutator, not setBalance(). Validate FIRST, mutate LAST.
    void deposit(std::int64_t c) {
        if (c <= 0)                  throw std::invalid_argument("deposit must be positive");
        if (c > kMaxCents - cents_)  throw std::overflow_error("balance cap exceeded");
        // ^ overflow-safe: cents_ is in [0,max], so (max - cents_) can't overflow. We never
        //   compute cents_ + c, which could be UB for large inputs.
        cents_ += c;                 // commit: cannot throw
        checkInvariant();
    }

    void withdraw(std::int64_t c) {
        if (c <= 0)       throw std::invalid_argument("withdraw must be positive");
        if (c > cents_)   throw std::runtime_error("insufficient funds");
        cents_ -= c;
        checkInvariant();
    }

    // Multi-object operation with the STRONG guarantee: check everything, then commit.
    static void transfer(Account& from, Account& to, std::int64_t c) {
        if (&from == &to)                  throw std::invalid_argument("self-transfer");
        if (c <= 0)                        throw std::invalid_argument("amount must be positive");
        if (c > from.cents_)               throw std::runtime_error("insufficient funds");
        if (c > kMaxCents - to.cents_)     throw std::overflow_error("destination cap exceeded");
        from.cents_ -= c;                  // commit phase: only non-throwing integer ops
        to.cents_   += c;
        from.checkInvariant(); to.checkInvariant();
    }

    std::int64_t cents() const noexcept { return cents_; }          // read-only observer
    const std::string& id() const noexcept { return id_; }
};
```

Walk through `deposit`. First it rejects bad input. Then it checks the cap with `c > kMaxCents - cents_`, a subtraction that can never overflow, instead of `cents_ + c`, which could. Only after every check passes does it do the one line that changes state. That "check everything, then change" shape is the same in `withdraw` and `transfer`.

### Real-world picture

Invariants are **database constraints plus transactions**. A `CHECK (balance >= 0)` constraint is enforced at one choke point, no matter which service writes. A class with public fields is a database where every microservice has direct `UPDATE` rights and no constraints.

`setStart` and `setEnd` as separate calls are **two separate commits for what should be one transaction**: a reader between them sees an inconsistent row. `transfer` is a real _transaction_: validate all preconditions, then commit in one step.

Deserializing without validation is **bulk-loading a CSV straight into the table and bypassing the constraints**.

### Interview traps

- **Public fields or reflexive getters and setters** ("anemic" classes): the invariant is enforced nowhere.
- `if (a + b > MAX)` is a **signed overflow** check that the optimizer may delete. Also watch for `size_t` underflow (`vec.size() - 1` on an empty vector) and `int` ← `size_t` narrowing.
- **Setter pairs** with a multi-field invariant (`start <= end`, `min <= max`).
- **Validation of a copy that was never made:** validating a reference parameter and then storing it (TOCTOU), or, in Java, validating `Date` or array arguments _before_ the defensive copy.
- **`assert` used as input validation** silently disappears in release builds.
- **Exceptions in the middle of a mutation** leave half-updated state (no strong guarantee). Interviewers ask "what if the second `push_back` throws?"
- **Constructor leaks and half-built objects:** `init()` methods, virtual calls in constructors, publishing `this` early.
- Java: deserialization or reflection creating objects that skip your constructor, `Serializable` with a non-validating `readObject`, mutable `Date` fields, and `equals`/`hashCode` on mutable keys in a `HashMap`.
- Thread safety: two atomics do not make a two-field invariant atomic.

### Tricky questions and answers

#### Q1 \[SDE-1/2\]: Why is this class dangerous even though every field is private? Fix it.

```cpp
class DateRange {
    int start_, end_;                       // intended invariant: start_ <= end_
public:
    DateRange(int s, int e) : start_(s), end_(e) {}
    void setStart(int s) { start_ = s; }
    void setEnd(int e)   { end_ = e; }
};
// Move [1,5] to [10,20]:
r.setStart(10);   // now [10,5]: invariant broken
r.setEnd(20);
```

**Answer:** Four problems:

1. The constructor does not validate, so `DateRange(9, 1)` is accepted.
2. Individual setters make **valid transitions impossible**. Moving `[1,5]` to `[10,20]` forces a temporary `[10,5]`. Moving the other way (`[10,20]` to `[1,5]`) needs `setEnd` first, so the caller must know the right order.
3. A concurrent reader, or an exception between the two calls, sees a corrupt object.
4. The class cannot protect its own invariant, because each setter lacks the other half of the information.

Fix: **one mutator that takes both values**, or make the class immutable (Topic 2.3):

```cpp
class DateRange {
    int start_, end_;
    static void check(int s, int e) { if (s > e) throw std::invalid_argument("start > end"); }
public:
    DateRange(int s, int e) : start_(s), end_(e) { check(s, e); }
    void reschedule(int s, int e) {           // atomic w.r.t. the invariant: validate, then commit
        check(s, e);
        start_ = s; end_ = e;                 // int stores: cannot throw
    }
    int start() const noexcept { return start_; }
    int end()   const noexcept { return end_; }
};
```

#### Q2 \[SDE-2/3\]: List every way this class can break "all stock counts are ≥ 0", and fix them all.

```cpp
class Inventory {
    std::vector<int> stock_;                               // invariant: every element >= 0
public:
    Inventory(const std::vector<int>& s) : stock_(s) {}    // (1)
    std::vector<int>& stock() { return stock_; }           // (2)
    void remove(std::size_t i, unsigned n) { stock_[i] -= n; }   // (3)
};
```

**Answer:**

1. **No validation** in the constructor: `Inventory({-5})` starts with a broken invariant.
2. **The getter leaks a mutable reference:** `inv.stock().push_back(-1)` or `inv.stock()[0] = -100` bypasses everything (this is Topic 2.4).
3. **`stock_[i] -= n`** has three problems: (a) no bounds check (UB), (b) no check that stock ≥ n, and (c) `int -= unsigned` converts the int to `unsigned`, subtracts modulo 2³², and converts back, so `5 - 10` becomes a wrapped value (negative on typical platforms, implementation-defined before C++20).

The implicit constructor also allows `Inventory inv = vec;`.

```cpp
##include <algorithm>
class Inventory {
    std::vector<int> stock_;                                   // invariant: every element >= 0
public:
    explicit Inventory(std::vector<int> s) : stock_(std::move(s)) {   // by value: our own copy
        if (std::any_of(stock_.begin(), stock_.end(), [](int x) { return x < 0; }))
            throw std::invalid_argument("negative stock");     // validate OUR copy (no TOCTOU)
    }
    int at(std::size_t i) const { return stock_.at(i); }       // read-only; bounds-checked
    std::size_t size() const noexcept { return stock_.size(); }
    void remove(std::size_t i, int n) {
        if (n <= 0) throw std::invalid_argument("n must be positive");
        int& slot = stock_.at(i);                              // throws std::out_of_range
        if (slot < n) throw std::runtime_error("insufficient stock");
        slot -= n;                                             // cannot go negative or overflow
    }
};
```

If several threads call `remove`, add a `std::mutex` held across the check and the decrement. Check-then-act is **not** atomic on its own.

## Topic 2.3: True Immutability

### The idea in plain words

An **immutable** object never changes after it is built. A printed book is immutable: to "change" it you print a new edition. Immutable objects are wonderful because nobody can break their rules, and many threads can read them with no locks.

The word to be careful about is **true** (deep) immutability. An object is truly immutable only if _everything reachable from it_ is also unchangeable, through every path. A glass jar with a locked lid is not immutable if the jar holds a pointer to something you can still edit.

The biggest beginner trap: **`const` does not mean immutable.** `const` is shallow, and it only promises that _this particular name_ will not be used to change things.

### How it works inside

- **`const` is not immutability.** A `const` member function makes `this` a `const T* const`, so the members become const. That is **shallow**: a `const std::shared_ptr<X>` still lets you change what the pointer points to (`*ptr`). A `const T&` parameter is only a read-only _view_, and another alias elsewhere can still change the object.
- **Modifying an object that was _defined_ `const` is UB** (via `const_cast` and a write). The compiler may fold reads into constants (every use of `kLimit` replaced by `10`) and place the object in **`.rodata`** (read-only data), whose pages the memory hardware marks read-only, so the write usually crashes with `SIGSEGV`. `const_cast` is only legal when the underlying object was not originally const.
- **`const` data members have a cost:** they delete the implicit copy and move _assignment_, and a `const std::string` cannot be moved _from_, so "moves" silently copy. For classes that live in a `std::vector` or get swapped, prefer **private non-const members with no mutators**, or hold the data through `shared_ptr<const Data>`.
- **`mutable`** allows caches, lazy fields and mutexes inside `const` methods. The standard-library convention is that **`const` member functions must be safe to call from many threads at once**, so `mutable` state needs a mutex or atomic. Otherwise it is a **data race = UB**, even if it "looks harmless."
- **Why immutables are fast with threads:** after construction there are no writes, so there are no locks and no cache-line fighting. The one requirement is **safe publication**: a release/acquire edge (mutex, `atomic_store`/`atomic_load`, thread start/join, `shared_ptr` control-block atomics) so readers see the _fully built_ object.
- **Structural sharing / copy-on-write:** instead of copying a big structure for every "change," build a small new wrapper that points to shared immutable parts. Copying a `shared_ptr<const Data>` is O(1): one atomic increment.
- **Snapshot publishing (RCU-like):** readers grab a `shared_ptr<const T>` and use it as long as they like. The writer builds a new `T` and swaps the pointer atomically. The last reader frees the old version.
- **Java comparison:** `final` fields get a **freeze action** (JLS §17.5): once the constructor finishes (and `this` did not escape), _any_ thread that sees the reference sees the final fields and what they point to, **even if the publication itself was a data race**. Non-final fields get no such guarantee (a reader may see default values). `String` is the classic immutable class: it enables the string pool, a cached hash, and safe use as map keys and in security checks. `Collections.unmodifiableList` is a _view_ over a mutable list (changes show through), while `List.copyOf` and `List.of` are immutable _snapshots_. A `record` is only shallowly immutable.

### Code: a deeply immutable config and a lock-free snapshot holder

```cpp
##include <atomic>
##include <memory>
##include <stdexcept>
##include <string>
##include <utility>
##include <vector>

// ---- Deeply immutable value with O(1) copies and "wither" updates ----
class Config {
    struct Data {                                  // all state lives behind a pointer-to-CONST
        std::string              name;
        std::vector<std::string> hosts;
        int                      timeoutMs;
    };
    std::shared_ptr<const Data> d_;                // const pointee => nobody can mutate, even us

    static void validate(const Data& d) {          // invariants checked ONCE, at every creation path
        if (d.name.empty())      throw std::invalid_argument("name empty");
        if (d.hosts.empty())     throw std::invalid_argument("no hosts");
        if (d.timeoutMs <= 0)    throw std::invalid_argument("timeout must be > 0");
    }
    explicit Config(Data&& d) {
        validate(d);
        d_ = std::make_shared<Data>(std::move(d)); // make_shared<Data> then converts to shared_ptr<const Data>
    }
public:
    // By-value params + move: we own independent copies; the caller's vector can't alias ours.
    Config(std::string name, std::vector<std::string> hosts, int timeoutMs)
        : Config(Data{std::move(name), std::move(hosts), timeoutMs}) {}

    // Only const observers. Returning const& is safe: the pointee never changes and lives
    // as long as this Config (or any copy of it) does.
    const std::string&              name()      const noexcept { return d_->name; }
    const std::vector<std::string>& hosts()     const noexcept { return d_->hosts; }
    int                             timeoutMs() const noexcept { return d_->timeoutMs; }

    // "Wither": returns a NEW object; the original is untouched and still valid for other threads.
    Config withTimeout(int ms) const {
        Data copy = *d_;                           // O(n) copy of the changed version only
        copy.timeoutMs = ms;
        return Config(std::move(copy));            // re-validates
    }
    // Compiler-generated copy/move: copies a shared_ptr (one atomic refcount op). Safe & cheap.
};

// ---- Lock-free-for-readers snapshot holder (hot-reloadable config, feature flags, routing tables) ----
template <class T>
class Snapshot {
    std::shared_ptr<const T> cur_;
public:
    explicit Snapshot(std::shared_ptr<const T> init) : cur_(std::move(init)) {}
    // atomic_load/store on shared_ptr give the release/acquire edge = SAFE PUBLICATION.
    // (C++20: prefer std::atomic<std::shared_ptr<T>>; libstdc++'s impl uses a small internal lock pool.)
    std::shared_ptr<const T> load() const { return std::atomic_load(&cur_); }
    void store(std::shared_ptr<const T> next) { std::atomic_store(&cur_, std::move(next)); }
};

// Reader thread:  auto cfg = holder.load();  use cfg->timeoutMs() as long as you like: no locks, no tearing.
// Writer thread:  holder.store(std::make_shared<const Config>(old->withTimeout(500)));

// ---- const OBJECT vs const REFERENCE ----
const int kLimit = 10;                             // defined const: may be folded, may live in .rodata
// void evil() { *const_cast<int*>(&kLimit) = 20; }  // UB: typically SIGSEGV; reads may still see 10
```

How to read this code: `Config` holds all its data behind `shared_ptr<const Data>`. Because the thing it points to is `const`, even `Config`'s own member functions cannot change it. To "change" a timeout you call `withTimeout`, which builds a **new** `Config` and leaves the old one alone. `Snapshot` then lets a writer swap in the new version while readers keep using whichever version they already grabbed.

### Real-world picture

An immutable object is a **versioned, content-addressed artifact**: a Docker image digest, a Git commit, an S3 object version. You never edit `image@sha256:abc`. You publish a new digest, and the running fleet switches the _pointer_ (the tag or deployment) atomically. Every reader of the old digest keeps working until it drains.

`Snapshot<T>` is exactly **blue/green deployment of a config, inside one process**. Mutable shared state, by contrast, is logging in to production with `ssh` and editing files in place while requests are in flight.

### Interview traps

- **Shallow immutability:** `const` members that are pointers, references or `shared_ptr` to _mutable_ data, or a getter that returns a mutable handle. The class _looks_ immutable but is not.
- **Constructor aliases caller state:** storing the caller's vector, array or `shared_ptr` without copying, so the caller can still change "your" fields.
- **`const_cast` plus a write on a truly const object** (UB), and a "const method that mutates through `mutable` without synchronization" (a data race).
- **Lazy `mutable` caches in shared objects** (such as a cached hash). A "benign race" does not exist in the C++ memory model. Java's `String.hash` gets away with it only because int writes are atomic and the calculation gives the same result every time.
- **`const` members kill move and assignment,** causing silent copies inside `vector` and `sort`.
- **Safe publication:** immutable objects published through a plain non-atomic pointer are still racy. The _pointer_ needs synchronization. In Java, non-final fields in an "immutable" class, or leaking `this` from the constructor, defeat the final-field freeze.
- `unmodifiableList` is a view, not a copy. A `public static final int[]` is still mutable. A `record` with a `List` component is shallow.
- **Performance trap:** "copy the whole thing on every update" costs O(n) per write. A good answer talks about structural sharing, persistent data structures and batching writes.
- **Immutable keys:** a mutable object used as a `HashMap` or `unordered_map` key whose hash changes after insertion becomes unreachable.

### Tricky questions and answers

#### Q1 \[SDE-1/2\]: Is `Event` immutable? Justify, then fix it.

```cpp
class Event {
    const std::string                   name_;
    std::shared_ptr<std::vector<int>>   tags_;
public:
    Event(std::string n, std::shared_ptr<std::vector<int>> t)
        : name_(std::move(n)), tags_(std::move(t)) {}
    const std::string& name() const { return name_; }
    std::shared_ptr<std::vector<int>> tags() const { return tags_; }
};
```

**Answer:** **No.** Three reasons:

1. The constructor stores the caller's `shared_ptr`, so the caller keeps a handle and can `push_back` into "our" tags at any time (aliasing).
2. `tags() const` is a `const` method, but `const` only makes the _pointer member_ const. It returns a handle to a **mutable** vector, so any caller can change it.
3. A `const std::string` member makes `Event` non-assignable, and moving from an `Event` silently copies the string (cheap to fix, expensive at scale).

The `const` keyword created a false sense of safety, because it is shallow. Fix: own the data by value, or share it only through a pointer-to-const that nobody else can change:

```cpp
class Event {
    std::string                         name_;     // private, no mutators => immutable, still movable
    std::shared_ptr<const std::vector<int>> tags_; // pointee is const
public:
    Event(std::string n, std::vector<int> t)       // take by value, so the caller can't keep a handle
        : name_(std::move(n)),
          tags_(std::make_shared<const std::vector<int>>(std::move(t))) {}
    const std::string&      name() const noexcept { return name_; }
    const std::vector<int>& tags() const noexcept { return *tags_; }   // read-only view
};
```

#### Q2 \[SDE-3\]: 64 reader threads share one `Key` and call `hash()`. What is wrong, and what are the fixes?

```cpp
class Key {
    std::string       s_;
    mutable std::size_t hash_   = 0;
    mutable bool        hashed_ = false;
public:
    explicit Key(std::string s) : s_(std::move(s)) {}
    std::size_t hash() const {
        if (!hashed_) { hash_ = std::hash<std::string>{}(s_); hashed_ = true; }
        return hash_;
    }
};
```

**Answer:** Two threads can both see `hashed_ == false` and write `hash_` and `hashed_` at the same time. That is a **data race, which is UB**, even though both would write the same value. It is worse than that: with no ordering between the two stores, a reader can see `hashed_ == true` while still seeing the _old_ `hash_ == 0` (the compiler or CPU may reorder the stores), so it returns a wrong hash.

Fixes, best first:

```cpp
// Fix 1 (best): compute eagerly. The class becomes truly immutable: no mutable, no races.
explicit Key(std::string s) : s_(std::move(s)), hash_(std::hash<std::string>{}(s_)) {}
// (declare 's_' before 'hash_' so it is initialized first; see Module 1 init-order trap)

// Fix 2: lazy but race-free, the Java String.hash approach. One atomic word, 0 = "not computed".
class Key {
    std::string s_;
    mutable std::atomic<std::size_t> hash_{0};
public:
    explicit Key(std::string s) : s_(std::move(s)) {}
    Key(const Key& o) : s_(o.s_), hash_(o.hash_.load(std::memory_order_relaxed)) {}  // atomic is non-copyable
    std::size_t hash() const {
        std::size_t h = hash_.load(std::memory_order_relaxed);
        if (h == 0) {                                   // idempotent computation => relaxed is enough
            h = std::hash<std::string>{}(s_);
            hash_.store(h, std::memory_order_relaxed);  // a true hash of 0 is just recomputed: harmless
        }
        return h;
    }
};
```

`s_` is never written after construction, so relaxed ordering is enough as long as the `Key` itself was published safely. Avoid `std::mutex` and `std::once_flag` here: they make the class non-copyable and add cost to a hot path.

Interview takeaway: **"immutable" objects with `mutable` caches must pay for their own synchronization.**

## Topic 2.4 (Added): Leaky Encapsulation — Returned References, Views & Defensive Copies

### The idea in plain words

You can make every field private and still lose all your protection with one careless getter. Imagine a safe with a strong lock, but you hand a visitor a window that opens inside the safe. Encapsulation **leaks** when a function gives out something that lets callers reach, change, or outlive the object's internal state. That "something" could be:

- a non-const reference, pointer or iterator,
- a mutable handle, or
- a view whose lifetime is not tied to its owner.

### How leaks happen

- A getter that returns `T&` or `T*` hands out an **address inside the object's storage**. The caller can write through it (bypassing every invariant). The address can also **dangle** if the container reallocates (`vector::push_back` moves its buffer), if the member is destroyed, or if the owner was a temporary.
- **`const T&` blocks writes but not lifetime bugs.** And `const_cast` or another alias can still defeat it.
- **Returning by value** copies (O(n) for containers) but gives the caller _its own independent copy_. Small types (`int`, `string_view`, short SSO strings) are cheap. Large containers are not, but a copy of a _consistent snapshot_ is exactly what thread-safe classes need.
- **Views** (`std::string_view`; `std::span` in C++20) are a pointer plus a length, passed in two registers. They avoid copying but are **non-owning**, so lifetime is the caller's problem. Returning a view of a member is fine as long as the owner outlives the view.
- **Temporaries and range-for:** `for (auto& x : makeObj().items())` binds only the _returned reference_ to the loop. The temporary `makeObj()` dies at the end of the range initializer, so the loop walks a dangling reference (UB; fixed in C++23 for this specific case only). **Ref-qualified overloads** (`&` vs. `&&`) solve it: a temporary hands over ownership by value.
- **Visitor or callback access** (`forEach(fn)`) never lets the container escape, and lets the owner hold a lock for the whole walk.
- **Java comparison:** returning an internal `ArrayList` or array exposes everything. `Collections.unmodifiableList` is a _live view_, and `List.copyOf` is a snapshot. **Defensive copy order** (Effective Java Item 50): copy first, _then_ validate the copy, and do not use `clone()` on parameters whose type an attacker can subclass.

### Code: three safe ways to give out data

```cpp
##include <mutex>
##include <string>
##include <utility>
##include <vector>

class Roster {
    mutable std::mutex       m_;
    std::vector<std::string> names_;
public:
    void add(std::string n) { std::lock_guard<std::mutex> g(m_); names_.push_back(std::move(n)); }

    // BAD: leaks the internal container. Callers can clear()/sort()/push_back with no lock held,
    //      and any reference they keep dangles on the next reallocation.
    std::vector<std::string>& badNames() { return names_; }

    // GOOD 1: snapshot by value. O(n), but a consistent, independently owned copy.
    std::vector<std::string> snapshot() const {
        std::lock_guard<std::mutex> g(m_);
        return names_;                              // copied under the lock
    }

    // GOOD 2: visitor. The container never escapes, and the lock is held for the whole traversal.
    template <class Fn>
    void forEach(Fn&& fn) const {
        std::lock_guard<std::mutex> g(m_);
        for (const auto& n : names_) fn(n);         // WARNING: fn must not call back into Roster (deadlock)
    }
};

// Ref-qualified accessors: safe for both lvalues and temporaries (single-threaded value type).
class Playlist {
    std::vector<std::string> songs_;
public:
    explicit Playlist(std::vector<std::string> s) : songs_(std::move(s)) {}
    const std::vector<std::string>& songs() const &  { return songs_; }            // lvalue: borrow
    std::vector<std::string>        songs() &&       { return std::move(songs_); } // temporary: give ownership
};
```

(A `std::lock_guard` locks a mutex when created and unlocks it when destroyed, so the lock is released even if an exception is thrown. The `&` and `&&` after `const` mean "this overload is used when the object is an lvalue (a named object)" and "when it is an rvalue (a temporary)".)

### Real-world picture

A leaky getter is an API that **hands out a database connection with write privileges** when the caller only asked for a report. A snapshot is a **read replica or an exported CSV**: independent, slightly stale but consistent, and safe to hand out. A visitor is a **cursor with the lock held on the server**: the client sees rows, never the table.

Returning a `const&` to an internal map from a locked method is like **handing out a read-replica credential after the lease has already expired**.

### Interview traps

- A getter that returns a **non-const reference or pointer to an internal container**: invariants are bypassed (Topic 2.2).
- A getter that returns **`const&` under a lock**: the lock ends at `return`, and the caller reads unprotected (see Q2).
- **Dangling reference to a temporary's member** in a range-for (`makeObj().items()`).
- **Iterator and reference invalidation:** keeping `&v[0]` or `v.begin()` across a `push_back`.
- **`string_view` of a `std::string` member** returned after the owner is destroyed, or of a temporary `std::string` argument.
- **A constructor that stores the caller's pointer or reference** (the aliasing problem from Topic 2.3), or exposes `this` through callbacks.
- **Returning `T&&` or `std::move(member)` from a `const` method** (compile surprises), or from lvalues (steals state unexpectedly).
- Java: `getItems()` returning the live list, `Arrays.asList` aliasing the array, `Date` getters, and `public static final` arrays.
- Cost question: "Return by value is slow, isn't it?" Answer: measure. RVO and moves, small sizes, and snapshots for consistency often win. Use views when the lifetime is obvious.

### Tricky questions and answers

#### Q1 \[SDE-2\]: Why does this loop sometimes crash, and what is the idiomatic fix?

```cpp
class Playlist {
    std::vector<std::string> songs_;
public:
    const std::vector<std::string>& songs() const { return songs_; }
};
Playlist makePlaylist();
for (const auto& s : makePlaylist().songs()) { std::cout << s << '\n'; }
```

**Answer:** A range-for expands to `auto&& __r = makePlaylist().songs();`. Lifetime extension applies to the _result of `songs()`_ (a reference), **not** to the temporary `Playlist` that was only used to call it. That temporary is destroyed at the end of the full expression, so `__r` dangles and the loop reads freed memory (ASan reports `heap-use-after-free`).

C++23 fixes this specific range-for case, but you cannot rely on that before C++23. The fix is the **ref-qualified overload pair** shown in `Playlist` above: a temporary `Playlist` returns `songs()` _by value_ (moving the vector out), and the range-for extends the life of that returned value.

Alternatives: `auto p = makePlaylist(); for (auto& s : p.songs())`, or return a snapshot by value from the start.

#### Q2 \[SDE-2/3\]: This class has a mutex around every access. Why is it still racy?

```cpp
class Registry {
    mutable std::mutex           m_;
    std::map<std::string, int>   data_;
public:
    void put(std::string k, int v) { std::lock_guard<std::mutex> g(m_); data_[std::move(k)] = v; }
    const std::map<std::string, int>& all() const {
        std::lock_guard<std::mutex> g(m_);
        return data_;                       // lock released here
    }
};
// Thread A: for (auto& [k, v] : reg.all()) {...}     Thread B: reg.put("x", 1);
```

**Answer:** The mutex protects only the **return statement**. The `lock_guard` is destroyed as `all()` returns, and Thread A then walks the live map with no lock while Thread B inserts. That is a **data race**, and it may even crash inside the tree's rebalancing code. The _lock_ was fully encapsulated, but the _data_ leaked.

Options:

```cpp
// 1. Snapshot by value (consistent copy; O(n) per call).
std::map<std::string, int> all() const { std::lock_guard<std::mutex> g(m_); return data_; }

// 2. Visitor: lock held while the caller iterates (document: callback must not re-enter).
template <class Fn> void forEach(Fn&& fn) const {
    std::lock_guard<std::mutex> g(m_);
    for (const auto& kv : data_) fn(kv.first, kv.second);
}

// 3. Copy-on-write snapshot (Topic 2.3): O(1) reads, writers pay O(n).
//    std::shared_ptr<const std::map<...>> cur_;  readers: atomic_load(&cur_);
//    writers: copy map, modify, atomic_store(&cur_, newMap).
```

Choose by the read/write ratio: a visitor for rare reads, a snapshot for small data, copy-on-write snapshots for read-heavy hot paths. The general rule: **never return a reference to guarded state.**

## Topic 2.5 (Added): PIMPL, ABI Stability & the Compilation Firewall

### The idea in plain words

Here is a surprise for beginners: `private` hides a member's _name_, but the member is still written in your header file. Anyone who includes the header can see exactly which private variables exist, and the compiler uses them to decide how big your object is.

The **PIMPL idiom** ("pointer to implementation") goes one step further. You put _all_ the private data into a separate struct that is defined only in the `.cpp` file. The header contains only a pointer to it. Clients no longer see your private layout at all. It hides data at the **build and binary level**, not just at the language level.

Think of the header as a restaurant menu and the `.cpp` as the kitchen. Customers see the menu. You can rearrange the kitchen as much as you like without reprinting the menu.

### How it works inside

- `private` hides _names_ but not **layout**. Private data members still sit in the header, so they decide `sizeof`, member offsets, and inline-function code **in every client**. Changing a private member forces a **recompile of everything that includes the header**, and for a shared library it silently breaks programs built against the old header (see Q2).
- With PIMPL, the header exposes only a `unique_ptr<Impl>` (8 bytes, stable forever). `Impl` can change freely, and third-party types it uses never leak into client includes (faster builds, no transitive dependencies).
- **The incomplete-type rule:** `unique_ptr<Impl>`'s deleter needs `sizeof(Impl)` and `delete`, which need a _complete_ type at the point where the destructor is generated. So **declare** the destructor (and the move operations) in the header and **define** them (`= default`) in the `.cpp`, where `Impl` is complete. Otherwise you get: _"invalid application of 'sizeof' to incomplete type"_.
- **Copy semantics must be written by hand** (deep-copy `Impl`), and a moved-from object has a null `impl_`, so guard against that.
- **Cost:** one heap allocation per object, one extra pointer hop (a possible cache miss), and no inlining of hidden members. Mitigations: a "fast-pimpl" with in-object `aligned_storage` plus `static_assert(sizeof(Impl) <= N)`, pools, or using PIMPL only for non-hot-path public types.
- **A real incident:** GCC 5 changed the layout of `std::string` (the "dual ABI"), breaking binaries across the boundary. This is why library authors ship PIMPL or C-ABI facades.
- **Java comparison:** the JVM resolves fields and methods by **name and descriptor at link time** instead of baking offsets into callers. So adding a private field or reordering fields does not break binary compatibility (JLS §13). The cost is paid at run time (symbolic resolution, the JIT). C++ pays at build time and through ABI fragility.

### Code: a PIMPL class split across header and source

```cpp
// ============ widget.h : what clients see (no private data, no heavy includes) ============
##pragma once
##include <memory>
##include <string>

class Widget {
public:
    explicit Widget(std::string name);
    ~Widget();                                      // DECLARED here, defined where Impl is complete
    Widget(Widget&&) noexcept;                      // move: declared; defined = default in .cpp
    Widget& operator=(Widget&&) noexcept;
    Widget(const Widget&);                          // copy: user-written deep copy
    Widget& operator=(const Widget&);

    void render() const;
private:
    struct Impl;                                    // incomplete type: layout hidden from clients
    std::unique_ptr<Impl> impl_;                    // 8 bytes, ABI-stable forever
};

// ============ widget.cpp : free to change without recompiling clients ============
##include "widget.h"
##include <iostream>
##include <vector>                                   // heavy / third-party deps stay HERE

struct Widget::Impl {
    std::string      name;
    std::vector<int> cache;                         // add/remove/reorder fields: clients unaffected
};

Widget::Widget(std::string name)
    : impl_(std::make_unique<Impl>(Impl{std::move(name), {}})) {}

Widget::~Widget() = default;                        // Impl is complete HERE, so unique_ptr<Impl> can delete it
Widget::Widget(Widget&&) noexcept = default;
Widget& Widget::operator=(Widget&&) noexcept = default;

Widget::Widget(const Widget& o) {
    if (o.impl_) impl_ = std::make_unique<Impl>(*o.impl_);   // deep copy; moved-from source has null impl_
}
Widget& Widget::operator=(const Widget& o) {
    if (this != &o) {
        if (o.impl_) impl_ = std::make_unique<Impl>(*o.impl_);
        else         impl_.reset();
    }
    return *this;
}
void Widget::render() const { std::cout << impl_->name << '\n'; }
```

#(`#pragma once` tells the compiler to include a header only once per file. "Incomplete type" means a type that has been _named_ but whose contents have not been shown yet.)

### Real-world picture

PIMPL is the **versioned public API in front of a service's internals**. Clients depend on the stable REST or gRPC contract (the header), not on the database schema (the private layout). You can migrate tables, add columns, or switch databases (change `Impl`) without redeploying every client.

A class with private members written inline in the header is a service that **exposes its database schema to clients through an ORM they compile in**. Any column change forces every client to rebuild and redeploy.

### Interview traps

- **Destructor defined (or implicitly generated) in the header** while `Impl` is incomplete: the classic `unique_ptr` compile error. The same trap applies to the implicit move operations.
- **Forgetting to write the copy operations:** `unique_ptr` makes the class move-only, so `Widget w2 = w1;` will not compile. Writing them carelessly dereferences a null `impl_` after a move.
- **`shared_ptr<Impl>` as a shortcut** silently gives _shared_ (aliasing) copy behavior, so changing one `Widget` changes its copies.
- **Performance question:** "Is PIMPL OK in a hot loop?" Usually not: the allocation and indirection cost matter. Know fast-pimpl, and know when to skip PIMPL.
- **ABI breakers:** adding or reordering data members, adding or reordering virtual functions (vtable slots shift), changing inline function bodies, changing the `sizeof` of a base class, and switching compilers or standard-library ABIs.
- **Exceptions across shared-library boundaries**, and returning `std::string` or other STL types through a `.so` API, are fragile across compiler versions. A C-ABI facade avoids this.
- Java: "Why does adding a private field not break dependents?" (symbolic linking, no baked-in offsets.) "What does JPMS add beyond `private`?" (module-level `exports`.)

### Tricky questions and answers

#### Q1 \[SDE-2\]: This PIMPL class fails to compile. Why? Fix it without touching `Impl`'s definition.

```cpp
// widget.h
class Widget {
    struct Impl;
    std::unique_ptr<Impl> impl_;
public:
    Widget();
};
// main.cpp
##include "widget.h"
int main() { Widget w; }            // error: invalid application of 'sizeof' to incomplete type 'Widget::Impl'
```

**Answer:** The class has an **implicitly declared destructor**, which is generated inline wherever it is first used. Here that is `main.cpp`, where `Impl` is incomplete. `~unique_ptr<Impl>` instantiates `default_delete<Impl>`, which has a `static_assert(sizeof(Impl) > 0)` that fails. The constructor has the same problem on its exception path (it must be able to destroy `impl_`).

Fix: declare the destructor in the header and define it in the `.cpp` where `Impl` is complete. Because declaring a destructor suppresses the implicit moves, also declare and default them there:

```cpp
// widget.h
class Widget {
    struct Impl;
    std::unique_ptr<Impl> impl_;
public:
    Widget();
    ~Widget();
    Widget(Widget&&) noexcept;
    Widget& operator=(Widget&&) noexcept;
};
// widget.cpp (Impl is complete here)
struct Widget::Impl { /* ... */ };
Widget::Widget() : impl_(std::make_unique<Impl>()) {}
Widget::~Widget() = default;
Widget::Widget(Widget&&) noexcept = default;
Widget& Widget::operator=(Widget&&) noexcept = default;
```

#### Q2 \[SDE-3\]: Your shared library v1 has `class Foo { int a_; public: Foo(); int get() const { return a_; } };`. v2 adds a `double b_`. Clients are NOT recompiled, and only `libfoo.so` is swapped. What breaks, and how do you design to survive this?

**Answer:** `sizeof(Foo)` changes from 4 to 16 (with alignment padding), but old clients were compiled believing it is 4:

- A client that does `Foo f;` (on the stack) or `new Foo` allocates **4 bytes**, and the v2 constructor inside the `.so` writes `b_` at offset 8: a **stack or heap buffer overflow** and silent memory corruption.
- Inline accessors compiled into the client (`get()`) use the _old_ offsets. If members were reordered, they read the wrong field.
- Adding a virtual function shifts the vtable slots, so old clients call the **wrong function** through the vtable.
- Changing the body of an inline function has no effect on old clients (they hold their own copy of the code).

Design options, strongest first:

```cpp
// 1. C-ABI opaque handle: clients never see layout, allocation, or exceptions.
extern "C" {
    typedef struct FooHandle FooHandle;                  // incomplete: layout is private to the .so
    FooHandle* foo_create(void);
    int        foo_get(const FooHandle*);
    void       foo_destroy(FooHandle*);
}

// 2. PIMPL on the C++ class (this topic): sizeof(Foo) == sizeof(unique_ptr) forever.

// 3. Inline namespaces for deliberate ABI versioning: v1 and v2 symbols coexist.
namespace foo { inline namespace v2 { class Foo { /* new layout */ }; }
                namespace v1 { class Foo { /* old layout, kept for old clients */ }; } }
```

Plus discipline:

- Never reorder or remove members or virtual functions in a shipped header. Only append (and only if clients never allocate the type themselves).
- Keep STL types out of the exported ABI.
- Enforce it in CI with an ABI checker (`abidiff`, `abi-compliance-checker`).

Interview framing: **in C++, "private" protects names, but only an opaque pointer protects layout.**

## Module 2 Cheat Sheet

| Concept           | One-line interview answer                                                                                                                      |
| ----------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| Access modifiers  | Compile-time name checks on the static type, per class not per object. No runtime protection, and not a security boundary.                     |
| Overload + access | Overload resolution first, access check second. A private best match is an error.                                                              |
| NVI               | Public non-virtual API owns the contract, private virtual hooks customize behavior.                                                            |
| `protected`       | Access only through the derived type. Prefer protected functions over protected data.                                                          |
| Invariants        | Established by the constructor (or throw), preserved by every public mutator. Validate first, commit last.                                     |
| Mutators          | Intent-revealing operations (`deposit`, `reschedule`), not paired setters.                                                                     |
| Overflow          | Signed overflow is UB, so write `b > MAX - a`. Beware `size_t` underflow and mixed signedness.                                                 |
| `assert`          | Internal invariants only, never external input.                                                                                                |
| `const`           | Shallow. A `const` method means const members, not a const pointee. Writing to a defined-const object is UB.                                   |
| Immutability      | Private state, no mutators, deep (pointer-to-const), safe publication, wither methods.                                                         |
| `mutable`         | Needs its own synchronization. There is no "benign race" in C++.                                                                               |
| Snapshot pattern  | `shared_ptr<const T>` + atomic swap gives lock-free reads and RCU-style updates.                                                               |
| Leaky getters     | Never return a mutable or lock-guarded reference. Return a snapshot, a visitor, or a ref-qualified value.                                      |
| PIMPL             | Hides layout and dependencies. Declare the destructor in the header and define it in the `.cpp`, and hand-write copies.                        |
| ABI               | Adding members or virtuals changes `sizeof` and the vtable, so use opaque pointers, PIMPL, or C-ABI facades.                                   |
| Java contrast     | The JVM re-checks access at link time, JPMS seals modules, `final` fields get freeze semantics, and fields link by name (no baked-in offsets). |
