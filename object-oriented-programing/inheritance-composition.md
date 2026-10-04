---
title: "Module 3: Inheritance & Composition"
description: "Explore how a child object contains its parent in memory, how the diamond problem and constructor chains work, and when to choose composition over inheritance."
author: ["name": "Rajendra Pancholi", "email": "rpancholi522@gmail.com"]
thumbnail: "/images/oop-inheritance-composition.png"
tags: [OOP, C++, Inheritance, Composition, Liskov-Substitution]
keywords: ["Inheritance vs composition", "Diamond problem virtual inheritance", "Liskov Substitution Principle", "Constructor and destructor order", "Fragile base class problem"]
---

# OOP Mastery for MAANG Interviews: Module 3 — Inheritance & Composition

> Inheritance is an "Is-A" relationship where a derived (child) class inherits fields and methods from a base (parent) class. In memory, the child object physically contains the parent object's data.

![Inheritance & Composition](/images/oop-inheritance-composition.png)

### How this module connects to Module 2

Module 2 taught you how to protect one object's rules. Module 3 asks what happens when one class is **built on top of another**. A child class is allowed to reach into its parent, and that coupling is exactly where encapsulation can quietly break. You will see how the parent's data sits _inside_ the child, why that matters for memory and safety, and when you should choose "has-a" (composition) over "is-a" (inheritance).

### Words you will meet in this module

| Word                      | Simple meaning                                                                                                                  |
| ------------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| **Base / derived class**  | Parent / child. `Dog` is derived from `Animal`, so `Animal` is the base.                                                        |
| **Subobject**             | The base-class part that lives _inside_ a derived object.                                                                       |
| **Upcast**                | Treating a child as its parent (`Dog*` → `Animal*`). Always safe.                                                               |
| **Downcast**              | Treating a parent as a child (`Animal*` → `Dog*`). Only safe if the object really is a Dog.                                     |
| **vptr / vtable**         | A hidden pointer in each polymorphic object, pointing to a per-class table of function pointers (Module 4 explains this fully). |
| **Thunk**                 | A tiny adjustment function the compiler inserts to fix the `this` pointer.                                                      |
| **Virtual base**          | A base class shared by all paths in a diamond. Only one copy exists.                                                            |
| **LSP**                   | Liskov Substitution Principle: a child must work anywhere its parent works.                                                     |
| **ABI**                   | The binary-level agreement (sizes, layouts, call rules) that lets separately compiled code work together.                       |
| **Static / dynamic type** | Static = the type written in the code. Dynamic = the real type of the object at run time.                                       |

## 3.1: The "Is-A" Relationship & Memory Layouts

### The idea in plain words

A `Dog` **is an** `Animal`. Everywhere an `Animal` is expected, you can use a `Dog`. That is called **public inheritance**, and it models an "is-a" relationship (this idea of "can be used in place of" is called _substitutability_, and Topic 3.4 explores it deeply).

The key fact for this topic is physical: **a `Dog` object contains a complete `Animal` object inside it.** Picture a Russian nesting doll. The Animal part is stored first, and the Dog's own extra fields come right after.

Once you see it this way, many rules stop being mysterious. For example, treating a `Dog*` as an `Animal*` costs nothing, because the Animal part starts at the very same address.

### How the layout works, step by step

#### Single inheritance

- The `Base` subobject sits at **offset 0**, followed by `Derived`'s own members. Because of that, converting `Derived*` → `Base*` is a **no-op** (same address), and every `Base` member function works unchanged on the first `sizeof(Base)` bytes.
- **Tail-padding reuse (Itanium ABI):** if `Base` is _not_ "POD for layout" (it has a user-provided constructor, virtual functions, and so on), the compiler may place `Derived`'s first member **inside `Base`'s tail padding** (the unused gap at the end of `Base`). If `Base` is a plain POD, it may not. MSVC never reuses it. So the same source code gives different `sizeof` on different ABIs, and _removing a constructor can make the derived class bigger_.

#### Polymorphic classes (with virtual functions)

- The vptr lives at offset 0 of the _primary_ base. A derived class **reuses that same vptr slot** and just points it at its own vtable.
- The derived vtable **starts with the base's slots in the same order** and adds new virtuals after them. That is why `base->f()` can use slot _k_ without knowing the real type.
- Constructors set the vptr (Topic 3.3).

#### Multiple inheritance

- Subobjects are laid out in base-declaration order. `class C : A, B` gives `A` at 0, `B` at `sizeof(A)` (aligned), then `C`'s own members.
- The first polymorphic base shares `C`'s primary vptr, and `B` has its _own_ vptr.
- Converting `C*` → `B*` **adds a constant offset** (and keeps `nullptr` null). Overriding `B`'s virtual functions in `C` goes through **thunks** that subtract the offset to get back to the `C* this`.

#### Other layout facts

- **Empty Base Optimization (EBO):** an empty base takes 0 bytes, while an empty _member_ costs at least 1 byte (plus padding). Policy, allocator and comparator classes are inherited from for this reason.
- **Standard-layout rule:** for `offsetof` and C interop, a class cannot have non-static data members in both the derived class and a base, and all members need the same access level (Topic 2.1).
- **Slicing** (Module 1): `Base b = derived;` copies only the base subobject.

#### Java comparison

The object has **one header** (mark word plus klass pointer, 12–16 bytes). The JVM lays out superclass fields _first_, then subclass fields (HotSpot may fill gaps in the superclass with small subclass fields). Upcasts are free (a reference is the same pointer). `invokevirtual` uses the klass pointer's vtable, and `invokeinterface` uses itables. Interfaces carry no instance fields. You can inspect layouts with the **JOL** tool.

### Code: layouts you can check with `static_assert`

```cpp
#include <cstdio>

// Tail-padding reuse: same fields, different sizeof depending on "POD-ness" of the base
struct Animal { int age; char kind; Animal() : age(0), kind('a') {} };  // non-POD: age@0, kind@4, pad 5..7
struct Dog : Animal { char tricks; };      // Itanium: tricks@5 (inside Animal's padding) => sizeof 8
                                           // MSVC: tricks@8 => sizeof 12
struct PodA   { int age; char kind; };     // POD base: tail padding is NOT reused (Itanium)
struct PodDog : PodA { char tricks; };     // tricks@8 => sizeof 12

// Polymorphic single inheritance: one vptr, base fields first
struct Shape  { virtual ~Shape() = default; double x = 0; };   // vptr@0, x@8      => 16
struct Circle : Shape { double r = 1; };                        // r@16 (same vptr) => 24
static_assert(sizeof(Shape) == 16 && sizeof(Circle) == 24, "LP64 Itanium/MSVC x64");

// Multiple inheritance: two subobjects, two vptrs, pointer ADJUSTMENT on upcast
struct A { virtual ~A() = default; double a = 1; };            // 16
struct B { virtual ~B() = default; double b = 2; };            // 16
struct C : A, B { double c = 3; };                              // A@0, B@16, c@32 => 40
static_assert(sizeof(C) == 40, "A(16) + B(16) + double(8)");

// EBO: an empty base is free, an empty member is not
struct Empty {};
struct WithBase   : Empty { int x; };      // sizeof 4
struct WithMember { Empty e; int x; };     // sizeof 8 (1 byte + 3 pad + 4)
static_assert(sizeof(WithBase) == 4 && sizeof(WithMember) == 8, "");

int main() {
    std::printf("Dog=%zu PodDog=%zu (ABI-dependent)\n", sizeof(Dog), sizeof(PodDog));
    C c;
    A* pa = &c;                                // primary base: same address
    B* pb = &c;                                // secondary base: address + 16 (implicit adjustment)
    auto off = [&](void* p) { return reinterpret_cast<char*>(p) - reinterpret_cast<char*>(&c); };
    std::printf("A* offset=%td, B* offset=%td\n", off(pa), off(pb));   // 0, 16
    C* none = nullptr; B* nb = none;           // compiler emits a null check: null stays null (not +16)
    std::printf("null B* stays null: %d\n", nb == nullptr);
    // B* bad = reinterpret_cast<B*>(&c);      // WRONG: no adjustment, points at A's vptr/data
}
```

(**POD** means "plain old data": a simple C-style struct with no constructors or virtual functions.)

The most important line is `B* pb = &c;`. The compiler quietly adds 16 to the address, because the `B` part of `c` really does start 16 bytes after `c` itself. If you used `reinterpret_cast` instead, no adjustment happens, and `pb` would point at the wrong place.

### Real-world picture

A derived object is a **Docker image built `FROM` a base image**. The base layers sit at the bottom, unchanged, and the new layers are stacked on top. Anything that understands the base layout can read the first N bytes and work.

#Multiple inheritance is **two base images merged into one container**. Each base expects its own filesystem root, so a mount point must be _translated_ (pointer adjustment) when a component that only knows base #2 is handed the container.

Tail-padding reuse is **squeezing a sidecar into leftover space in a pod**. It is efficient, but the footprint now depends on the scheduler (the ABI) you run on.

### Interview traps

- "What is `sizeof(Derived)`?" with some `char`s and a base with padding. The answer depends on tail-padding reuse. Say "**ABI-dependent**: Itanium gives X, MSVC gives Y." Do not guess one number.
- **Pointer adjustment:** `static_cast` and implicit conversions adjust, but **`reinterpret_cast`, C-style casts between unrelated bases, and `void*` round-trips do not** (see Q2).
- Adding a **virtual function or a data member to a base** changes the size and layout of every derived class (recompile everything, ABI break: Topic 2.5).
- **Slicing** when passing or storing polymorphic objects by value.
- **Arrays of derived through base pointers:** `Base* arr = new Derived[10]; arr[1]` steps by `sizeof(Base)`, giving UB and garbage.
- Inheriting from a class "just for the data" (`struct Foo : std::vector<int>`) leaks the whole interface, and there is no virtual destructor, so `delete` through the base is UB.
- Empty-class inheritance for EBO, versus `[[no_unique_address]]` (C++20).
- Java: "How are fields laid out in a subclass?" "Where does the interface method table live?" "Does a subclass cost extra header bytes?" (No.)

### Tricky questions and answers

#### Q1 \[SDE-1/2\]: What are `sizeof(Base)` and `sizeof(Derived)` on 64-bit GCC? Does MSVC agree?

```cpp
struct Base    { char a; virtual void f() {} };
struct Derived : Base { char b; int c; };
```

**Answer:**

- `Base`: vptr at 0 (8 bytes), `a` at 8, tail padding 9–15, so **sizeof = 16**. Its _data size_ (without tail padding) is 9, because it is non-POD (it has a virtual function).
- `Derived` on **Itanium (GCC/Clang)**: `b` goes into `Base`'s tail padding at offset **9**, and `c` (alignment 4) goes at offset **12**. That ends at 16, so **sizeof = 16**.
- On **MSVC**, tail padding is never reused: `b` at 16, `c` at 20, so **sizeof = 24**.
- Takeaway: inheritance is not "base size + derived size." Padding rules differ per ABI, so never hard-code layouts in a shipped binary interface. If layout matters, check with `static_assert(sizeof/offsetof)` for every ABI you support.

#### Q2 \[SDE-2/3\]: This crashes. Why, and what is the correct round-trip?

```cpp
struct A { virtual ~A() = default; double a = 1; };
struct B { virtual ~B() = default; double b = 2; virtual void hello() { std::puts("hi"); } };
struct C : A, B {};

void* store(C& c)        { return &c; }                    // type-erased handle (callback ctx, C API)
void  use(void* v)       { B* pb = static_cast<B*>(v); pb->hello(); }   // BUG
int main() { C c; use(store(c)); }
```

**Answer:** `&c` has the same address as `c`'s **`A` subobject** (offset 0). `static_cast<B*>(void*)` is _not_ an upcast. It only reinterprets that address as a `B*`, with **no +16 adjustment**. So `pb` points at `A`'s vptr and fields, and `pb->hello()` loads the wrong vtable slot and jumps to garbage (or calls the wrong virtual function).

`void*` erases the type, and with it the information needed to adjust. Rule: **convert to `void*` and back through the _same_ static type.**

```cpp
void* store(C& c)  { return static_cast<void*>(&c); }       // erase from C*
void  use(void* v) {
    C* pc = static_cast<C*>(v);                              // restore the SAME type first
    B* pb = pc;                                              // implicit upcast: +16 adjustment applied
    pb->hello();
}
// Better: avoid void* (use std::any/std::function/templates), or store a B* if B is what you need.
```

## 3.2: The Diamond Problem & Multiple Inheritance

### The idea in plain words

Suppose `Scanner` and `Printer` both inherit from `Device`, and then `Copier` inherits from both `Scanner` and `Printer`. The inheritance diagram looks like a diamond.

The problem: by default, `Copier` contains **two separate `Device` parts**, one through `Scanner` and one through `Printer`. That causes two headaches:

1. **Ambiguity:** when you write `copier.id`, which `Device`'s `id` do you mean?
2. **Duplicate, drifting state:** you can set one copy to 1 and the other to 2. They are supposed to be "the same device," but they now disagree.

C++ fixes this with **virtual inheritance**, which says "all paths share exactly one `Device`." It works, but it has real costs, which you will see below.

### How the diamond works, step by step

#### Without virtual inheritance

- `D` is laid out as `[B[A]] [C[A]] D-members`.
- `d.v` is ambiguous (`B::A::v` vs. `C::A::v`). `A* p = &d;` is ambiguous too.
- `d.B::v = 1; d.C::v = 2;` compiles but leaves **two independent copies**. That is the actual bug.

#### With virtual inheritance

Writing `struct B : virtual A` makes `A` a **shared virtual base**: exactly one `A` per most-derived object.

- The shared `A` cannot have a fixed offset from `B`, because the offset depends on the most-derived type (`B` alone vs. `B` inside `D`). So `B`'s code finds `A`'s members through a **vbase offset** stored in the vtable (Itanium; MSVC uses a `vbptr`). The steps are: _load vptr → load offset → add → load member_. That is **an extra dependent load for every access**, plus a vptr in classes that otherwise would not need one.
- Virtual bases are placed **after** the non-virtual subobjects, and **`static_cast` downcasts from a virtual base are forbidden** (the offset is unknown at compile time). Use `dynamic_cast` instead.
- **Construction rule:** the **most-derived class** initializes every virtual base _directly_, and the `A(...)` calls in `B`'s and `C`'s constructors are **ignored** when building a `D`. Virtual bases are constructed **first**, before any non-virtual base. If `A` has no default constructor, _every_ class down the hierarchy that could ever be the most-derived class (including grandchildren) must name it.

#### Which virtual function wins? (final overrider)

- If `B` and `C` both override `A::f`, then `D` **must** supply its own final overrider. Otherwise the program is ill-formed.
- If only one of them overrides, that override **dominates** and `D` is fine (see Q2).

#### Where you already use this

The `<iostream>` hierarchy: `basic_istream` and `basic_ostream` both inherit `virtual basic_ios`, and `basic_iostream` joins them.

#### Pure interfaces

Pure interfaces have no data, so duplication only costs a vptr, but the ambiguous-conversion problem is the same. Virtual inheritance for interface bases is the standard fix.

#### Java comparison

Java classes have **single inheritance**, so there is no state diamond. But interfaces with `default` methods (Java 8) can still conflict. If `D implements B, C` and both provide a default `f()`, `javac` rejects it ("inherits unrelated defaults") and you must override and choose with **`B.super.f()`**. The rules are:

1. A _class_ method beats any interface default.
2. A more-specific interface beats its super-interface.
3. Otherwise, resolve explicitly.

#Python resolves this with **C3 MRO linearization** (`super()` follows one linear order). Kotlin and C# require explicit `super<B>.f()` or explicit interface implementation.

### Code: broken diamond vs. fixed diamond

```cpp
#include <cstdio>

// BROKEN: non-virtual diamond
struct A0 { int v = 0; };
struct B0 : A0 {};
struct C0 : A0 {};
struct D0 : B0, C0 {};
static_assert(sizeof(D0) == 8, "TWO copies of A0::v (4 + 4)");
// D0 d; d.v = 1;           // ERROR: ambiguous (B0::A0::v or C0::A0::v?)
// A0* p = static_cast<A0*>(&d);  // ERROR: ambiguous base
// d.B0::v = 1; d.C0::v = 2;     // compiles; two divergent copies of "the same" state

// FIXED: virtual inheritance, one shared Device
struct Device {
    int id;
    explicit Device(int i) : id(i) { std::puts("Device()"); }      // no default ctor
    virtual ~Device() = default;
    virtual void describe() const { std::printf("Device %d\n", id); }
};
struct Scanner : virtual Device {
    explicit Scanner(int i) : Device(i) { std::puts("Scanner()"); } // Device(i) is IGNORED when a
    void describe() const override { std::puts("scans"); }          // Copier is being built
};
struct Printer : virtual Device {
    explicit Printer(int i) : Device(i) { std::puts("Printer()"); }
    void describe() const override { std::puts("prints"); }
};
struct Copier : Scanner, Printer {
    // The MOST-DERIVED class initializes the virtual base. Omitting Device(7) = compile error
    // (Device has no default constructor), even though Copier isn't a direct child of Device.
    Copier() : Device(7), Scanner(1), Printer(2) { std::puts("Copier()"); }
    // Both parents override Device::describe, so a unique FINAL OVERRIDER is mandatory:
    void describe() const override { Scanner::describe(); Printer::describe(); }
};

int main() {
    Copier c;                          // Device() Scanner() Printer() Copier()
    std::printf("id=%d\n", c.id);      // 7: one Device, initialized by Copier (not 1 or 2)
    Device& d = c;                     // unambiguous upcast; Device* d -> Copier* needs dynamic_cast
    d.describe();
    // Copier& back = static_cast<Copier&>(d);   // ERROR: static_cast from a virtual base is forbidden
    Copier& back = dynamic_cast<Copier&>(d);     // OK: uses RTTI to locate the full object
    (void)back;
}
```

(**RTTI** is run-time type information: extra data that lets `dynamic_cast` and `typeid` discover an object's real type.)

Notice that `Copier` writes `Device(7)` itself, even though `Device` is not its direct parent. And the `id=7` output proves the rule: `Scanner`'s `Device(1)` and `Printer`'s `Device(2)` were both ignored.

### Real-world picture

The diamond is **two microservices (B and C) that each embed their own copy of a shared library or config store (A)**, with a gateway (D) that combines them. You now have two caches that drift apart: B updates "its" copy and C reads "its" copy.

Virtual inheritance is **dependency injection of one shared instance**. The _composition root_ (the most-derived class) creates the shared dependency once and wires it into everyone. That is why B's and C's own wiring is ignored.

Interface default-method conflicts in Java are **two libraries exporting the same endpoint**: the gateway must explicitly pick one or merge them.

### Interview traps

- "What is the diamond problem and how does C++ solve it?" Expected answer: ambiguity **and** duplicate state, then `virtual` inheritance, **then the costs** (indirection, bigger objects, constructor-ordering rules, no `static_cast` downcast).
- **The constructor-initialization trap:** the most-derived class initializes the virtual base, so intermediate classes' `A(x)` arguments are silently ignored. Grandchildren must also initialize the virtual base.
- "Why not make _all_ inheritance virtual?" Cost: extra loads on every base-member access, bigger objects, and harder casting.
- **No unique final overrider** compile errors, versus the **dominance rule** that makes the same code compile when only one side overrides.
- `virtual` must go on the _intermediate_ inheritance (`B : virtual A`), not on `D : B, C`. Adding it at the bottom of the diamond fixes nothing.
- **Cross-casts** (`B*` → `C*` inside a `D`) need `dynamic_cast`. `static_cast` does not compile.
- Prefer **composition or pure interfaces** to multiple inheritance of _state_.
- Java: default-method conflicts and the `Iface.super.method()` syntax, "Why doesn't Java allow multiple class inheritance?", "What does the 'class wins' rule mean?" Python: MRO and C3, and `super()` in cooperative multiple inheritance.

### Tricky questions and answers

#### Q1 \[SDE-2\]: What does this print? Which line would fail to compile if I deleted `A(4)` from `E`?

```cpp
struct A { A(int x) { std::printf("A(%d)\n", x); } };
struct B : virtual A { B() : A(1) { std::puts("B"); } };
struct C : virtual A { C() : A(2) { std::puts("C"); } };
struct D : B, C       { D() : A(3) { std::puts("D"); } };
struct E : D          { E() : A(4) { std::puts("E"); } };
int main() { D d; E e; }
```

**Answer:**

```
A(3)
B
C
D
A(4)
B
C
D
E
```

- For `D d;`, `D` is the most-derived class, so **`A(3)` runs first** (virtual base before `B`, then `C`). `B`'s `A(1)` and `C`'s `A(2)` are ignored.
- For `E e;`, `E` is now the most-derived class, so **`A(4)`** wins, even though `E` is not a direct child of `A`. `D`'s `A(3)` is ignored.
- If `A(4)` is removed from `E`, `E`'s constructor implicitly tries `A()`, which does not exist, so `E::E()` fails to compile ("no matching function for call to `A::A()`").

Lesson: when a virtual base has no default constructor, **every concrete descendant must name the base**. So give shared virtual bases a default constructor, or make them stateless interfaces.

#### Q2 \[SDE-3\]: Does `d.f()` compile? What changes with or without `virtual`? How does Java handle the equivalent interface case?

```cpp
struct A { virtual void f() { std::puts("A"); } virtual ~A() = default; };
struct B : virtual A { void f() override { std::puts("B"); } };
struct C : virtual A { };
struct D : B, C { };
D d; d.f();                       // (1)
// Variant: non-virtual inheritance
//   struct B : A {...}; struct C : A {};  struct D : B, C {};   d.f();   // (2)
// Variant: both override
//   struct C : virtual A { void f() override {...} };                   // (3)
```

**Answer:**

- **(1) compiles and prints `B`.** Name lookup finds `B::f` and `A::f` (through `C`), but `B::f` **overrides** `A::f` in the _same_ shared `A` subobject, so it **dominates** (it hides `A::f`). The final overrider of `A::f` in `D` is `B::f`, which is unique.
- **(2) is ambiguous.** There are two `A` subobjects, so lookup finds `B::f` (for one `A`) and `A::f` (for the other), and neither hides the other. Error.
- **(3) is ill-formed** (_no unique final overrider_): both `B::f` and `C::f` override `A::f` in the one shared `A`. Fix by overriding in `D`: `void f() override { B::f(); C::f(); }`.
- **Java equivalent:**

```java
interface A { default String f() { return "A"; } }
interface B extends A { default String f() { return "B"; } }
interface C extends A { }
class D implements B, C { }                       // OK: B is more specific than A => "B" (like dominance)

interface C2 extends A { default String f() { return "C"; } }
class E implements B, C2 {                         // ERROR: unrelated defaults
    public String f() { return B.super.f() + C2.super.f(); }   // must resolve explicitly
}
```

Java's rules (class beats interface, more-specific interface beats less-specific, otherwise explicit `X.super.f()`) mirror C++'s dominance and final-overrider rules. But Java never has duplicate _state_, which is the expensive half of the C++ problem.

## 3.3: Object Initialization Chains

### The idea in plain words

When you build an object that sits at the bottom of a family tree, **the parts are built in a strict order**, like building a house: foundation first, then walls, then the roof, then you move in. When the object is destroyed, everything happens in **exact reverse**, like demolition.

The **initialization chain** is that exact, language-defined sequence. If you know it, you can predict every constructor and destructor call. If you do not, you will get bugs where one part uses another part that does not exist yet.

### The chain in C++, step by step

**Construction order** for `new D` or `D d;`:

1. Storage is obtained (`operator new`, or stack space).
2. **Virtual bases**, in depth-first, left-to-right order of the inheritance graph (done only by the _most-derived_ class).
3. **Direct non-virtual bases**, in **declaration order**.
4. **Non-static data members, in declaration order** (using the mem-initializer or default member initializer; otherwise they are default-initialized, which means _indeterminate for scalars_).
5. The **constructor body**.

The order in the mem-initializer list is _irrelevant_. `-Wreorder` warns when you write it in a different order.

**Destruction is the exact reverse:** destructor body → members in reverse declaration order → direct bases in reverse order → virtual bases.

#### The vptr changes while the object is being built

Each class's constructor sets the vptr to _its own_ vtable **after its bases are built and before its own member initializers run**. So:

- A virtual call inside `Base::Base()` resolves to `Base`'s version. (The `Derived` part does not exist yet.)
- A **pure virtual call in a constructor or destructor is UB** ("pure virtual method called", then abort).
- Destructors likewise reset the vptr when they begin.

#### What happens when a constructor throws

- **Fully constructed bases and members are destroyed in reverse order**, but that class's own destructor does **not** run, because the object never fully existed.
- Raw owning pointers that were set up _before_ the throwing member leak, so use RAII members.
- A **function-try-block** can observe the exception, but the exception is always re-thrown.
- With a **delegating constructor**, the object counts as constructed once the _target_ constructor finishes. So if the delegating constructor's body then throws, the destructor _does_ run.

#### Other facts

- **Inheriting constructors** (`using Base::Base;`) run the base constructor, and the derived-only members are default-initialized or take their default member initializers.
- **Static initialization order:** across translation units (separate `.cpp` files) the order is _unspecified_. This is the **static initialization order fiasco**. Use function-local statics (thread-safe "magic statics" since C++11).

#### Java comparison

Class initialization (`<clinit>`, superclass first) happens once. For `new D()`:

1. Memory is **zeroed**.
2. `D()` calls `super()` **first**.
3. Then `D`'s field initializers and instance-initializer blocks run.
4. Then the constructor body runs.

**A virtual call in a superclass constructor dispatches to the subclass override**, which sees _default (null or 0) fields_. This is the opposite of C++. Java has no destructors (`finalize` is deprecated for removal), so there is no reverse chain, only `AutoCloseable`.

### Code: watch the whole chain run

```cpp
#include <cstdio>
inline void log(const char* s) { std::puts(s); }

struct VBase { VBase() { log("VBase()"); } virtual ~VBase() { log("~VBase()"); } };
struct Base1 { Base1() { log("Base1()"); } virtual ~Base1() { log("~Base1()"); } };
struct Base2 { Base2() { log("Base2()"); } ~Base2() { log("~Base2()"); } };
struct M1    { M1() { log("M1()"); } ~M1() { log("~M1()"); } };
struct M2    { M2() { log("M2()"); } ~M2() { log("~M2()"); } };

// Bases: Base2, virtual VBase, Base1.   Members declared: m2_, then m1_.
struct Derived : Base2, virtual VBase, Base1 {
    M2 m2_;
    M1 m1_;
    // List order is scrambled on purpose: it does NOT matter. Declaration order wins.
    Derived() : m1_(), Base1(), Base2(), VBase() { log("Derived()"); }
    ~Derived() { log("~Derived()"); }
};

int main() {
    Derived d;
    // Construction:  VBase() Base2() Base1() M2() M1() Derived()
    //   virtual base first, then bases in declaration order, then members in declaration order
    // Destruction:   ~Derived() ~M1() ~M2() ~Base1() ~Base2() ~VBase()   (exact reverse)
}
```

The initializer list in `Derived()` is scrambled on purpose. The output proves that the compiler ignores that order: the virtual base comes first, then `Base2` and `Base1` (as declared in the class header), then `m2_` and `m1_` (as declared in the class body).

### Real-world picture

The chain is **service startup ordering with dependencies**. Shared infrastructure (virtual bases: the database, the message bus) comes up first. Then platform layers (bases, in declared order). Then this service's own components (members, in declared order). Only then does `main()` start serving (the constructor body). Shutdown is the **reverse drain**.

A virtual call in a base constructor is a **platform layer calling into an app plugin that has not started yet**. C++ routes it to the platform's default handler (safe but surprising). Java routes it to the not-yet-initialized plugin (crash-prone).

An exception during startup triggers a **partial rollback**: components that are already up are torn down in reverse, but the service's own `shutdown()` hook never runs, because it never fully started.

### Interview traps

- **"In what order do constructors run?"** Candidates forget that virtual bases come first, and that members follow _declaration_ order, not list order.
- **Virtual call in a constructor or destructor:** C++ calls the current class's version. Java calls the subclass override on a half-initialized object. A pure virtual call is UB (and usually an abort).
- **A member that depends on another member** (Module 1's `Window` trap): the dependency must be _declared_ earlier.
- **Exception in the middle of a chain:** which destructors run? (Only completed bases and members, never the object's own, and raw `new` members leak.)
- **A destructor that throws during unwinding** calls `std::terminate`.
- **Virtual base with no default constructor:** every most-derived class must initialize it (Topic 3.2).
- **Static init order fiasco** across translation units, plus the thread safety of function-local statics and destruction order at exit.
- **Leaking `this`** from a constructor (registering callbacks, starting threads) exposes half-built objects (in C++ and Java).
- Java: initialization blocks vs. field initializers vs. constructor order, `static` blocks and class-init deadlocks, and the "overridable method in constructor" warning.

### Tricky questions and answers

#### Q1 \[SDE-2\]: What does this print in C++? What would the Java translation do differently?

```cpp
struct A {
    A() { std::puts("A()"); init(); }
    virtual void init() { std::puts("A::init"); }
    virtual ~A() { std::puts("~A()"); }
};
struct B : A {
    int* p = new int(5);
    B() { std::puts("B()"); }
    void init() override { std::printf("B::init p=%p\n", (void*)p); }
    ~B() { delete p; std::puts("~B()"); }
};
int main() { B b; }
```

**Answer:** C++ prints:

```
A()
A::init
B()
~B()
~A()
```

While `A()` runs, the object's dynamic type is **`A`** (the vptr points to `A`'s vtable), so `init()` resolves to `A::init`, and `B::init` is never called. This is _safe_, because `B::p` does not exist yet.

In **Java**, the same code calls **`B.init()` from `A`'s constructor**, with `B`'s fields still at their _default values_ (`p == null`, since `B`'s field initializers run only after `super()` returns). The typical result is a `NullPointerException` or a silently wrong value.

Rule for both languages: **do not call overridable methods from constructors.** Do two-phase work in a factory (`static create()`), or make the hook `final` or non-virtual.

#### Q2 \[SDE-3\]: What is the exact output, and what leaks? Fix it.

```cpp
struct Res    { explicit Res(const char* n) { std::puts(n); } ~Res() { std::puts("~Res"); } };
struct Base   { Base() { std::puts("Base"); } ~Base() { std::puts("~Base"); } };
struct Thrower{ Thrower() { std::puts("Thrower"); throw std::runtime_error("x"); }
                ~Thrower() { std::puts("~Thrower"); } };
struct D : Base {
    int* raw;
    Res a{"a"};
    Thrower t;
    Res b{"b"};
    D() : raw(new int[10]) { std::puts("D body"); }
    ~D() { delete[] raw; std::puts("~D"); }
};
int main() { try { D d; } catch (...) { std::puts("caught"); } }
```

**Answer:**

```
Base
a
Thrower
~Res        <- member a, destroyed during unwinding
~Base
caught
```

Members initialize in declaration order: `raw` (`new int[10]` succeeds), `a` (prints `a`), then `Thrower` prints and throws. Unwinding destroys **only what was fully constructed**, in reverse: `a` (`~Res`), then the base (`~Base`).

**Not run:** `~Thrower` (it never finished), `b` (never started), the `D` body, and **`~D()`** (the object was never fully built). Therefore **`raw` leaks 40 bytes**, because a plain pointer has no destructor.

Fix with RAII members, so the unwinding path frees automatically (Rule of Zero):

```cpp
struct D : Base {
    std::unique_ptr<int[]> raw = std::make_unique<int[]>(10);  // destroyed during unwinding
    Res a{"a"};
    Thrower t;
    Res b{"b"};
    // no user-written destructor needed
};
```

Also valid but less common: a **function-try-block** (`D() try : ... {} catch (...) { /* log; rethrown automatically */ }`) lets you _observe_ the exception, but it cannot suppress it or access members.

## 3.4: Composition vs. Inheritance & the Liskov Substitution Principle

### The idea in plain words

There are two ways to reuse a class:

- **Inheritance ("is-a")**: `Dog` is an `Animal`. The subclass reuses the base class _and shows all of its public interface to the world_. You can see inside the base (this is called white-box reuse).
- **Composition ("has-a")**: a `Car` has an `Engine`. The car holds an engine object and _forwards_ some calls to it. Outsiders cannot see the engine. (This is black-box reuse.)

The **Liskov Substitution Principle (LSP)** is the rule that decides whether "is-a" is honest. It says: code written against `Base` must keep working, unchanged, with any `Derived`.

A famous failure: in mathematics, a square is a rectangle. In code, a _mutable_ `Square` that inherits from a mutable `Rectangle` is a lie, because a rectangle lets you change width and height independently and a square cannot. You will see exactly why below.

The practical rule most experts follow: **prefer composition, and use inheritance only when the child can truly replace the parent everywhere.**

### What LSP formally demands of an override

- **Preconditions cannot be strengthened.** The override must accept at least what the base accepts.
- **Postconditions cannot be weakened.** The override must promise at least what the base promises.
- **Invariants and the history constraint are preserved.** For example, an immutable base cannot get a mutating subclass.
- **No new exception types** that the base contract does not allow.
- **Covariant returns and contravariant parameters** are allowed (Topic 3.5).

### Why inheritance couples tightly (the low-level view)

- **Layout coupling:** base members live _inside_ every derived object (Topic 3.1), so adding a base field changes every derived `sizeof` and offset (recompile or ABI break).
- **Interface coupling:** the derived class exposes _everything_ public in the base, including operations that violate its own invariants (`Square::setHeight`).
- **Implementation coupling (the fragile base class):** overrides depend on _which base methods call which others_ (self-use). A base refactor that keeps its documented behavior can still break subclasses (Q2).
- **Compile-time and static:** you cannot change the base at run time. `final` and virtual dispatch are the only seams.

### What composition costs

- **Forwarding functions:** one extra call, which is inlined if the member's type is concrete.
- A member held **by value** is laid out _inline_ (no allocation, no indirection).
- A `unique_ptr<Interface>` member costs a heap allocation plus a virtual call, and buys **run-time swappability** (the Strategy pattern) and easy test doubles.

### When inheritance is the right tool

- A true behavioral subtype (`Circle` is a `Shape` in the immutable sense).
- You need to **override virtual functions** or access **protected** members.
- EBO for empty policy types.
- Use **private inheritance** for "implemented in terms of" only when composition cannot do the job.

### Java comparison

Effective Java Item 18 says "favor composition over inheritance," and Item 19 says "design for inheritance or prohibit it" (use `final`). The JDK has famous mistakes: `Stack extends Vector` and `Properties extends Hashtable` (callers can bypass the invariants). `HashSet.addAll` internally calls `add`, so a counting subclass double-counts (the Java twin of Q2).

### Code: the Square/Rectangle failure and two fixes

```cpp
#include <cassert>
#include <memory>
#include <set>

// 1. LSP VIOLATION: a Square is-a Rectangle mathematically, but NOT behaviorally (mutable)
class Rectangle {
protected:
    int w_, h_;
public:
    Rectangle(int w, int h) : w_(w), h_(h) {}
    virtual ~Rectangle() = default;
    virtual void setWidth(int w)  { w_ = w; }       // POSTCONDITION (documented): height unchanged
    virtual void setHeight(int h) { h_ = h; }       // POSTCONDITION: width unchanged
    int area() const { return w_ * h_; }
};
class Square : public Rectangle {
public:
    explicit Square(int s) : Rectangle(s, s) {}
    void setWidth(int w)  override { w_ = h_ = w; } // keeps Square's invariant, BREAKS the postcondition
    void setHeight(int h) override { w_ = h_ = h; }
};
void stretch(Rectangle& r) {                        // client code written against Rectangle
    r.setWidth(5);
    r.setHeight(2);
    assert(r.area() == 10);                         // FAILS for Square: 2*2 = 4
}

// 2. FIX A: don't inherit; share a read-only abstraction, make the value types immutable
class Shape {
public:
    virtual ~Shape() = default;
    virtual long long area() const = 0;
};
class Rect final : public Shape {
    int w_, h_;
public:
    Rect(int w, int h) : w_(w), h_(h) {}
    Rect withWidth(int w)  const { return Rect(w, h_); }   // "wither": new object, no contract broken
    Rect withHeight(int h) const { return Rect(w_, h); }
    long long area() const override { return 1LL * w_ * h_; }
};
class Square final : public Shape {
    int s_;
public:
    explicit Square(int s) : s_(s) {}
    Square withSide(int s) const { return Square(s); }
    long long area() const override { return 1LL * s_ * s_; }
};

// 3. FIX B: composition with a narrow, purpose-built surface
template <class T>
class CountingSet {
    std::set<T>  inner_;                            // HAS-A: black-box reuse
    std::size_t  attempts_ = 0;
public:
    bool insert(const T& v)   { ++attempts_; return inner_.insert(v).second; }
    template <class It> void insertAll(It b, It e) { for (; b != e; ++b) insert(*b); }
    bool contains(const T& v) const { return inner_.count(v) != 0; }
    std::size_t size() const noexcept { return inner_.size(); }
    std::size_t attempted() const noexcept { return attempts_; }
    // No erase()/emplace_hint()/iterators leaked: we expose only what we can keep correct.
};
```

(In this single file, both versions of `Square` appear to show the "before" and "after." In a real project you would keep only one. A "wither" is a method like `withWidth` that returns a _new_ object with one thing changed.)

### Real-world picture

Inheritance is **forking a repository and editing its internals**. You get everything for free, but every upstream refactor can conflict with your changes (fragile base class), and you ship all its surface area.

Composition is **depending on a published library through its public API**. Upstream can rewrite internals freely, you can swap the library (or inject a mock in tests), and you re-export only what you choose.

LSP is the **SLA of an interface**. A drop-in replacement service that returns different meanings (a `Square` that silently changes the height) is not a replacement. It is an outage waiting for the right caller.

### Interview traps

- **Rectangle and Square:** the classic LSP question. Be ready to name the broken _contract_ (postcondition: "width and height are independent") and offer fixes (immutability, separate types, composition, interface segregation).
- **Inheriting from standard containers** (`class MyStack : public std::vector<int>`): no virtual destructor (UB on `delete` through the base), and it leaks `insert` and `erase`, which break invariants. Use composition (`std::stack` is an adapter around a held container).
- **Self-use / fragile base class** (Q2): overriding both `add` and `addAll` double-counts.
- **Throwing "not supported"** from an override (`UnsupportedOperationException`, as in `Arrays.asList().add`): a classic LSP violation.
- **Strengthened preconditions:** a subclass that rejects inputs the base accepts (`null`, negative values, larger sizes).
- **Inheritance used only for code reuse** ("I want its `log()` method"): use composition or a free function.
- **Deep hierarchies** (more than 2–3 levels) and **protected data members** (anyone can break the invariant).
- **Performance:** forwarding overhead is real only through virtual dispatch or non-inlinable boundaries. Inline value members are free.
- "Does `final` help LSP?" It prevents unplanned subclassing (design for inheritance), and it enables devirtualization (Topic 3.5).

### Tricky questions and answers

#### Q1 \[SDE-2\]: Which assertion fails, and which LSP rule is broken? Give two fixes.

```cpp
Square s(1);
stretch(s);                     // stretch(Rectangle&) from above
```

**Answer:** `s.setWidth(5)` makes the square 5×5, and `s.setHeight(2)` then makes it 2×2, so `area() == 4` and the **`assert(r.area() == 10)` fails**.

`Rectangle`'s contract says each setter changes only its own dimension (a **postcondition**), and `Square` **weakens** it. (Changing one dimension can only keep `Square`'s own invariant _by breaking the base's_.) This is a behavioral violation, not a structural one: the types compile fine, and the problem shows up only for clients who rely on the contract.

Fixes:

1. **Do not model a mutable Square as a Rectangle subtype.** Share a read-only `Shape` interface, and keep `Rect` and `Square` as separate immutable value types (shown above, `withWidth` and `withSide`).
2. **Remove the contract-breaking operations from the shared type** (Interface Segregation): the common interface offers only `area()`, and `setWidth` and `setHeight` live on `Rect` alone.
3. (Composition variant) `Square` _has a_ `Rect` internally and exposes only `side()` and `area()`.

#### Q2 \[SDE-3\]: This `CountingBag` should count insertions. Why does adding 3 items report 6? Give three fixes and rank them.

```cpp
class Bag {
    std::vector<int> items_;
public:
    virtual ~Bag() = default;
    virtual void add(int x)                      { items_.push_back(x); }
    virtual void addAll(const std::vector<int>& xs) { for (int x : xs) add(x); }   // SELF-USE of add()
};
class CountingBag : public Bag {
    std::size_t count_ = 0;
public:
    void add(int x) override                          { ++count_; Bag::add(x); }
    void addAll(const std::vector<int>& xs) override  { count_ += xs.size(); Bag::addAll(xs); }
    std::size_t count() const { return count_; }
};
// CountingBag b; b.addAll({1, 2, 3}); b.count() == 6, not 3
```

**Answer:** `CountingBag::addAll` adds 3, then calls `Bag::addAll`, which **internally calls the virtual `add()` three times**. Each call dispatches to `CountingBag::add`, adding 3 more. This is the **fragile base class problem**: the subclass's correctness depends on an _undocumented implementation detail_ (that `addAll` calls `add`).

If `Bag` later "optimizes" `addAll` to use `items_.insert(...)` directly, the subclass's count silently becomes 0 _without any code change in the subclass_.

Fixes, best first:

```cpp
// 1. (Best) Composition + forwarding: no dependence on Bag's internals, narrow surface.
class CountingBag2 {
    Bag inner_;                           // or std::unique_ptr<BagInterface> for swappability
    std::size_t count_ = 0;
public:
    void add(int x)                       { ++count_; inner_.add(x); }
    void addAll(const std::vector<int>& xs) { count_ += xs.size(); inner_.addAll(xs); }
    std::size_t count() const { return count_; }
};

// 2. Override only add(): works NOW, but couples you to the self-use detail; Bag must DOCUMENT
//    "addAll calls add" and promise never to change it (an expensive contract).
class CountingBag3 : public Bag {
    std::size_t count_ = 0;
public:
    void add(int x) override { ++count_; Bag::add(x); }       // addAll is inherited, goes through add()
    std::size_t count() const { return count_; }
};

// 3. Make the base "designed for inheritance": NVI with protected non-self-calling hooks,
//    or mark Bag::addAll `final`/non-virtual so there's no override point to get wrong.
```

Java's `HashSet` / `InstrumentedHashSet` (Effective Java Item 18) is the same bug, and the same fix (a forwarding wrapper). Interview sound bite: _"Inheritance breaks encapsulation: the subclass depends on the superclass's implementation, not just its interface."_

## 3.5 Overriding vs. Hiding, `override`/`final`, Default Arguments & Covariant Returns

### The idea in plain words

When a child class defines a function with the same name as a parent's function, two very different things can happen, and mixing them up causes some of the most confusing bugs in C++:

- **Overriding:** the child _replaces_ a **virtual** function, and the replacement is chosen by the object's real (dynamic) type at run time.
- **Hiding:** the child's function **blocks the parent's name** during name lookup. Nothing is replaced. The parent's functions with the same name are simply no longer visible when you call through the child, no matter what their parameters are.

This topic teaches you to tell them apart, and gives you the tools (`override`, `final`, `using`) that stop the compiler from silently doing the wrong thing.

### The rules, step by step

- **Overriding rules:** a derived function overrides a base virtual only if it matches the **name, parameter types, cv-qualifiers (like `const`), and ref-qualifier**. The return type may differ only _covariantly_. A one-character mismatch (`int` vs. `double`, a missing `const`) silently declares a **new, unrelated function**. The keyword **`override`** makes the compiler verify the match. It is the single best defense. Make it mandatory (`-Wsuggest-override`).
- **Name hiding:** unqualified lookup stops at the **first scope that contains the name**. A derived `g(int)` hides all base `g` overloads, including virtual ones. Un-hide them with `using Base::g;`. Java has no such rule: overloads across the hierarchy are merged.
- **Default arguments are bound statically**, at the call site, using the **static type**, while the call itself is dispatched dynamically. So `Base* p = new Derived; p->f();` runs `Derived::f` with **`Base`'s** default. Never redefine defaults on virtual functions (use overloads or NVI instead).
- **`final`** on a class or function blocks further overriding. It also lets the compiler **devirtualize**: when the dynamic type is provably the final class, the call becomes a direct, inlinable call (no vtable load, no indirect branch). Java's JIT does the same through class-hierarchy analysis and inline caches, so `final` is a _hint_ there, not a requirement.
- **Covariant return types:** an override may return a more-derived **pointer or reference** (`Shape*` → `Circle*`). The class must be complete, and **smart pointers are not covariant** (`unique_ptr<Circle>` and `unique_ptr<Shape>` are unrelated class types that are merely convertible). The idiom: a private raw covariant virtual plus a public non-virtual smart-pointer wrapper.
- **Calling the base version:** `Base::f()` is a **qualified, non-virtual** call, resolved at compile time (no vtable).
- **Fragile base class** (Topic 3.4) and **ABI** (Topic 2.5): reordering or adding virtuals shifts vtable slots, so derived classes must be recompiled.
- **Java comparison:** every non-`private`, non-`static`, non-`final` instance method is virtual by default, and `@Override` is the verifying annotation. **Fields and static methods are hidden, not overridden** (`((Base) d).x` reads `Base.x`, chosen by the _static_ type). `super.f()` compiles to `invokespecial`. Java has no default arguments, so that trap does not exist.

### Code: every trap in one file

```cpp
#include <cstdio>
#include <memory>

struct Base {
    virtual void f(int x = 1) const { std::printf("Base::f(%d)\n", x); }
    virtual void g(double)          { std::puts("Base::g(double)"); }
    void h(int)                     { std::puts("Base::h(int)"); }
    virtual ~Base() = default;
};
struct Derived : Base {
    void f(int x = 2) const override { std::printf("Derived::f(%d)\n", x); }   // true override, NEW default
    void g(int)   /* override */     { std::puts("Derived::g(int)"); }         // NOT an override; hides g(double)
    void h(const char*)              { std::puts("Derived::h(cstr)"); }        // hides h(int)
    // Defenses: write `override` on g (compile error), and `using Base::g; using Base::h;` to un-hide.
};

// final enables devirtualization
struct Fast final : Base {
    void f(int x = 3) const override { std::printf("Fast::f(%d)\n", x); }
};
void callFast(const Fast& x) { x.f(); }   // dynamic type known => direct, inlinable call, no vtable load

// Covariant clone: raw covariant virtual (private) + smart-pointer wrapper (public)
class Shape {
public:
    virtual ~Shape() = default;
    std::unique_ptr<Shape> clone() const { return std::unique_ptr<Shape>(doClone()); }
private:
    virtual Shape* doClone() const = 0;                       // covariant-capable raw pointer
};
class Circle final : public Shape {
    Circle* doClone() const override { return new Circle(*this); }   // covariant: Circle* IS-A Shape*
public:
    std::unique_ptr<Circle> clone() const { return std::unique_ptr<Circle>(doClone()); }  // hides Shape::clone
};

int main() {
    Derived d;  Base* p = &d;
    p->f();          // Derived::f(1): dynamic function, STATIC default argument
    p->g(1.5);       // Base::g(double): Derived::g(int) is unrelated
    d.g(1.5);        // Derived::g(int) (1.5 narrowed to 1!): hiding turned a double call into an int call
    // d.h(5);       // ERROR: Derived::h(const char*) hides Base::h(int); 5 is not a pointer
    d.Base::h(5);    // qualified call reaches the hidden base overload
    std::unique_ptr<Circle> c = Circle().clone();   // typed result, no cast
}
```

Study `Derived::g(int)`. It _looks_ like it overrides `Base::g(double)`, but the parameter types differ, so it is a brand-new function. If the author had written `override` on it, the compiler would have refused to compile and caught the mistake at once.

### Real-world picture

`override` is the **schema or contract check in CI**. Without it, a typo in an endpoint signature deploys as a _brand-new_ endpoint, and the old handler keeps serving traffic.

Hiding is a **local config override that shadows the whole parent config block**, not just the key you meant to change.

Static default arguments are a **client SDK that bakes default timeouts in at compile time**, while the server's behavior is chosen at run time: caller and callee silently disagree.

`final` is **sealing a service version** so the load balancer can route directly instead of consulting the service registry on every call.

### Interview traps

- **Missing `override`:** signature drift (`const`, parameter type) silently creates a new virtual function. "Why isn't my override called?" is the number one debugging story.
- **Name hiding:** "I added an overload in the derived class and now the base overloads don't compile." Answer: `using Base::f;`.
- **Default arguments with virtual functions:** the classic output-prediction question (`Derived::f(1)`).
- **Covariance limits:** raw pointers and references only, and smart pointers need the NVI clone idiom.
- **`final` and performance:** devirtualization depends on the compiler _proving_ the type. A `final` class helps even through a `Base*` only if the pointer's static type is the final class.
- **Calling a virtual function through `Base::f()`** bypasses dispatch (useful for decorators, a bug elsewhere).
- **Overriding a private virtual** (legal, Topic 2.1), and overriding with a _different access level_.
- **Virtual functions in templates:** member function templates **cannot** be virtual.
- Java: field hiding (`Base.x` vs. `Derived.x`), static method "overriding" (it is really hiding), calling an overridable method in a constructor (Topic 3.3), and `@Override` as a compile-time check.

### Tricky questions and answers

#### Q1 \[SDE-2\]: Predict the output of `main()` above, then fix the class so `d.g(1.5)` and `d.h(5)` behave sensibly.

**Answer:**

```
Derived::f(1)
Base::g(double)
Derived::g(int)
Base::h(int)
```

- `p->f()`: virtual dispatch picks `Derived::f`, but default arguments are filled in using the **static type `Base`**, so `x == 1`, _not_ 2.
- `p->g(1.5)`: through `Base*`, lookup in `Base` finds `g(double)`, and `Derived::g(int)` is a _different_ function, not an override, so `Base::g(double)` runs.
- `d.g(1.5)`: lookup in `Derived` finds `g(int)` and **stops**, hiding `g(double)`. The `double` is implicitly narrowed to `1` and `Derived::g(int)` runs (`-Wconversion` warns).
- `d.h(5)` does not compile (shown commented out), and `d.Base::h(5)` calls the base overload.

Fix: mark overrides, un-hide intentionally, and do not redefine defaults:

```cpp
struct Derived : Base {
    void f(int x) const override { /* ... */ }        // no default: inherit the contract's default via
    void f() const { f(1); }                           // an overload, or use NVI
    void g(double) override { /* ... */ }              // `override` turns signature drift into a compile error
    using Base::h;                                      // un-hide base overloads
    void h(const char*) { /* ... */ }                   // now h(int) AND h(const char*) both visible
};
```

In an interview, also say you would compile with `-Wsuggest-override -Woverloaded-virtual` so the compiler flags both mistakes.

#### Q2 \[SDE-3\]: Why does `std::unique_ptr<Circle> clone() const override` not compile, and what is the idiom? Does `final` actually make calls faster?

```cpp
class Shape  { public: virtual std::unique_ptr<Shape>  clone() const = 0; virtual ~Shape() = default; };
class Circle : public Shape { public: std::unique_ptr<Circle> clone() const override; };   // error
```

**Answer:** Covariance applies only to **pointers or references to classes** whose pointee types are related by inheritance. `unique_ptr<Circle>` and `unique_ptr<Shape>` are **unrelated class types** (one is merely _convertible_ to the other), so the compiler rejects the override ("invalid covariant return type").

The idiom, shown in the lesson, is **NVI plus a private raw covariant virtual**. `doClone()` returns a raw `Circle*` (covariant), and public non-virtual `clone()` wrappers wrap it into `unique_ptr<Shape>` (in `Shape`) or `unique_ptr<Circle>` (in `Circle`). Ownership transfer happens in exactly one place per class, so a leak is impossible even if `doClone()` throws (nothing was allocated yet) or the wrapper is bypassed.

**Does `final` help performance?** Sometimes. If the compiler knows the dynamic type is a `final` class (the static type is the `final` class, or whole-program/LTO analysis proves it), it **devirtualizes** to a direct call and can inline it. That means no vtable load, no indirect-branch misprediction, and optimizations like constant propagation across the call. If you hold a `Shape*`, `final` on `Circle` gives nothing at that call site.

Measure: on predictable branches the vtable call costs only a few cycles. The big wins are the inlining-enabled optimizations in hot loops. The more important benefit is **design**: `final` forbids unplanned subclassing (Topic 3.4).

## Module 3 Cheat Sheet

| Concept                     | One-line interview answer                                                                                                                                                                      |
| --------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Is-A layout                 | Base subobject at offset 0, derived members after it. Upcast is a no-op for single inheritance. `sizeof` is ABI-dependent (tail-padding reuse).                                                |
| Polymorphic layout          | One vptr at offset 0, reused by derived classes. The derived vtable extends the base's slots in order.                                                                                         |
| Multiple inheritance        | Subobjects in declaration order, one vptr each. Upcast to a secondary base adds an offset, so use `static_cast` through the same type (never `void*` or `reinterpret_cast`).                   |
| Diamond                     | Two copies of A = ambiguity and divergent state. Fix with `virtual` inheritance on the _middle_ classes.                                                                                       |
| Virtual base cost           | Extra dependent load per access, bigger objects, no `static_cast` downcast, and the most-derived class initializes it.                                                                         |
| Final overrider / dominance | Both sides override → derived must override. One side overrides → that one dominates.                                                                                                          |
| Init order                  | Virtual bases → bases (declaration order) → members (_declaration_ order) → body. Destruction reverses.                                                                                        |
| vptr during ctor/dtor       | Dynamic type = the class currently running. Virtual calls do not reach derived code (Java does). Pure virtual call = UB.                                                                       |
| Throwing ctor               | Completed bases and members are destroyed, the object's own destructor never runs, raw `new` members leak. Use RAII.                                                                           |
| LSP                         | A subtype must not strengthen preconditions, weaken postconditions, or break invariants. Mutable Square ≠ Rectangle.                                                                           |
| Composition                 | Black-box reuse, narrow surface, run-time swappable, tests can inject fakes. Prefer it unless you need true subtyping.                                                                         |
| Fragile base class          | The subclass depends on the base's self-use. Fix with forwarding wrappers, NVI, `final`, or documented contracts.                                                                              |
| Overriding vs. hiding       | Always write `override`. A derived name hides all base overloads, so un-hide with `using`.                                                                                                     |
| Default arguments           | Bound to the static type. Never redefine on virtuals.                                                                                                                                          |
| Covariance                  | Raw pointers and references only. For smart pointers use the NVI `doClone()` idiom.                                                                                                            |
| `final`                     | Blocks overriding and derivation, and enables devirtualization. Java's JIT achieves similar speedups through class-hierarchy analysis.                                                         |
| Java contrast               | Single class inheritance + interface defaults (`X.super.f()`), virtual by default, a virtual call in a constructor reaches the subclass, fields and statics are hidden, `final` fields freeze. |
