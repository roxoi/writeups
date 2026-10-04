---
title: "Module 4: Polymorphism & Abstraction"
description: "See how virtual functions really work through the vtable and vptr, compare runtime and compile-time polymorphism, and learn abstract classes, interfaces and virtual destructors."
author: ["name": "Rajendra Pancholi", "email": "rpancholi522@gmail.com"]
thumbnail: "/images/oop-polymorphism-abstraction.png"
tags: [OOP, C++, Polymorphism, Virtual-Functions, Templates]
keywords: ["Polymorphism in C++", "VTABLE and VPTR explained", "Virtual destructor", "Abstract class vs interface", "CRTP and static polymorphism"]
---

# OOP Mastery for MAANG Interviews: Module 4 — Polymorphism & Abstraction

![Polymorphism & Abstraction](/images/oop-polymorphism-abstraction.png)

### How this module connects to Module 3

In Module 3 you saw that a polymorphic object carries a hidden pointer (the `vptr`), that constructors set it, and that `override` and `virtual` change which function runs. Module 4 opens that machinery completely.

"Polymorphism" just means **many forms**: one name, many behaviors. There are two big families:

- **Compile-time polymorphism:** the compiler decides which function to call while building your program (overloading, templates).
- **Run-time polymorphism:** the program decides while it is running, based on the object's real type (virtual functions).

This module covers both, explains exactly how each one works inside the machine, and shows you how to choose between them.

### Words you will meet in this module

| Word                           | Simple meaning                                                                                                   |
| ------------------------------ | ---------------------------------------------------------------------------------------------------------------- |
| **Overloading**                | Several functions with the same name but different parameters.                                                   |
| **Overload resolution**        | The compiler's process for picking the best one.                                                                 |
| **Name mangling**              | The compiler rewriting a function's name to include its parameter types, so the linker can tell overloads apart. |
| **Static type / dynamic type** | Static = the type written in the code. Dynamic = the real type of the object at run time.                        |
| **vptr / vtable**              | A hidden pointer inside each object, pointing to a per-class table of function pointers.                         |
| **Thunk**                      | A tiny helper function that adjusts the `this` pointer and then jumps to the real function.                      |
| **Key function**               | The first non-inline, non-pure virtual function of a class. It decides where the vtable is emitted.              |
| **RTTI**                       | Run-time type information: extra data that lets `dynamic_cast` and `typeid` find an object's real type.          |
| **Devirtualization**           | The compiler turning a virtual call into a direct call because it can prove the real type.                       |
| **ABI**                        | The binary-level agreement (sizes, layouts, call rules) that lets separately compiled code work together.        |
| **Type erasure**               | Hiding a concrete type behind a uniform interface (`function` is the famous example).                       |

## 4.1: Compile-Time Polymorphism (Overload Resolution & Name Mangling)

### The idea in plain words

You can give several functions the same name as long as their parameters differ. This is called **overloading**:

```cpp
void show(int);
void show(double);
void show(const char*);
```

When you write `show(3.5)`, the compiler looks at the **types of your arguments** (here, `double`) and picks the best match **right then, while compiling**. By the time your program runs, there is no decision left to make. The call goes straight to one specific function.

This is **compile-time polymorphism**: same name, different behavior, chosen early and for free at run time.

Two questions follow naturally, and this topic answers both:

1. How exactly does the compiler decide which function is the "best" one? (Overload resolution.)
2. The linker only understands flat names. How can two functions both be called `show`? (Name mangling.)

### How overload resolution works, step by step

**The pipeline:**

1. **Name lookup** (normal lookup plus **ADL**, argument-dependent lookup, which also searches the namespaces of the argument types) builds the candidate set.
2. **Template argument deduction** adds template candidates.
3. **Viability:** keep only candidates with the right number of arguments, where every argument can be converted to the parameter type.
4. **Best viable function:** rank each argument's conversion and pick the winner.
5. Only _then_ do **access** and `= delete` checks happen (Topic 2.1).

**Conversion ranks, from best to worst:**

1. **Exact match** (identity, lvalue-to-rvalue, array-to-pointer, adding `const`).
2. **Promotion** (`char`/`short`/`bool` → `int`, `float` → `double`).
3. **Conversion** (`int` → `double`, `long` → `int`, derived\* → base\*, `0` → `T*`).
4. **User-defined conversion.**
5. **Ellipsis** (`...`).

A function wins if it is **no worse on every argument and strictly better on at least one**. Otherwise the call is **ambiguous** (a compile error).

**Tie-breakers:**

- A **non-template beats a template** when the conversions are equal.
- A **more specialized template** beats a less specialized one.
- For reference binding: an rvalue (temporary) prefers `T&&` over `const T&`, and a non-const lvalue prefers `T&` over `const T&`.

**Other rules:**

- **Return type does not participate.** When the compiler chooses, it does not yet know how you will use the result. Default arguments do not create overloads.
- **Function templates:** only _primary_ templates take part in overload resolution. Explicit **specializations do not**, so a specialization can be silently bypassed by a better overload.
- **Cost:** zero at run time. The call is a direct `call` to a known symbol, and the optimizer can inline it.

### How name mangling works

The object file's symbol table is **flat**: it has no concept of overloads. So the compiler **encodes the namespace, class, name, cv-qualifiers and parameter types into the symbol** (Itanium: `_Z` + an encoding).

| Declaration           | Mangled symbol   |
| --------------------- | ---------------- |
| `void f(int)`         | `_Z1fi`          |
| `f(double)`           | `_Z1fd`          |
| `f(const char*)`      | `_Z1fPKc`        |
| `f(int&)`             | `_Z1fRi`         |
| `f(int&&)`            | `_Z1fOi`         |
| `ns::f(int)`          | `_ZN2ns1fEi`     |
| `Foo::bar(int) const` | `_ZNK3Foo3barEi` |

- Constructors and destructors get variants (`C1`/`C2`, `D0`/`D1`/`D2`, see 4.5).
- **Return types are _not_ mangled for ordinary functions** (but they are for template instantiations: `_Z1gIiEvT_`).
- **`extern "C"`** switches off mangling _and_ C++ overloading. The symbol is just `c_add`, so you get one function per name, usable from C.
- MSVC mangles differently (`?f@@YAXH@Z`), which is one reason C++ libraries are not ABI-portable across compilers.
- **See it yourself:** `g++ -c x.cpp && nm x.o` shows mangled names. `nm -C x.o` or `c++filt _ZNK3Foo3barEi` demangles them.

### Java comparison

Overloads are also chosen **at compile time by static types**, in phases: first without boxing or varargs, then with boxing, then with varargs. The JVM's version of mangling is the **method descriptor** (`foo(Ljava/lang/String;)V`), and unlike C++ it **includes the return type**. That is how covariant returns and generics work, using compiler-generated _bridge methods_. Generic overloads with the same erasure (`f(List<String>)` and `f(List<Integer>)`) are a compile error.

### Code: overload resolution in action

```cpp
#include <cstdio>

// Overload set: resolved on STATIC types at compile time
void show(int)         { puts("int"); }          // _Z4showi
void show(double)      { puts("double"); }       // _Z4showd
void show(const char*) { puts("const char*"); }  // _Z4showPKc

// Reference-qualified overloads (the basis of move semantics)
void take(int&)        { puts("int&");        }  // _Z4takeRi
void take(const int&)  { puts("const int&");  }  // _Z4takeRKi
void take(int&&)       { puts("int&&");       }  // _Z4takeOi

namespace ns { void f(int) {} }                       // _ZN2ns1fEi
struct Foo { void bar(int) const {} };                // _ZNK3Foo3barEi

extern "C" int c_add(int a, int b) { return a + b; }  // symbol: c_add (no mangling, no overloading)

int main() {
    show(0);        // int         (exact; 0 -> const char* would be a conversion)
    show('a');      // int         (char -> int is a PROMOTION; char -> double is a conversion)
    show(3.0f);     // double      (float -> double promotion)
    show(true);     // int         (bool -> int promotion)
    show(nullptr);  // const char* (only viable candidate)
    // show(3L);    // ERROR: ambiguous: long -> int and long -> double are both plain conversions

    int x = 1; const int c = 2;
    take(x);        // int&        (non-const lvalue: exact, and less cv-qualified than const int&)
    take(c);        // const int&
    take(3);        // int&&       (rvalue prefers && over const&)
}
```

(An **lvalue** is a named thing you can take the address of, like `x`. An **rvalue** is a temporary, like the literal `3`. "cv-qualified" means carrying `const` or `volatile`.)

Walk through `show('a')`: a `char` can become an `int` by _promotion_ (rank 2) or a `double` by _conversion_ (rank 3). Promotion beats conversion, so `show(int)` wins. In `show(3L)`, a `long` needs a plain conversion to reach either `int` or `double`. Both are rank 3, so neither wins, and the call is ambiguous.

### Real-world picture

Overload resolution is **API-gateway routing by request schema at deploy time**. Each endpoint is registered under a fully qualified signature (`/v1/orders/create?int,double`), and the router (the compiler) picks the best match once, statically.

Mangling is the **fully qualified service or endpoint ID in the service registry**. The registry (the linker) is flat, so identical short names would collide without the extra qualifiers.

`extern "C"` is publishing **a plain, unversioned REST path that any client language can call**, with no room for overloaded variants.

### Interview traps

- **"Can you overload on return type?"** No. The return type is not part of the signature, is not mangled, and the call context is not visible to overload resolution. (Workarounds: a templated conversion operator, or a proxy return object.)
- **Promotion vs. conversion ambiguities** (`f(long)` with `f(int)` and `f(double)`), and `f(0)` vs. `f(nullptr)` with pointer overloads (use `nullptr`).
- **Templates stealing calls:** a template with an _exact_ match beats a non-template that needs a promotion (see Q2).
- **Specializations do not overload** (and which primary template they attach to depends on declaration order). Prefer overloading to specializing function templates.
- **Hiding vs. overloading** (Topic 3.5): a derived-class name hides the base overloads. Overloading never crosses scopes.
- **Overload + virtual:** the overload is picked statically, and the override at run time (4.2).
- **ADL surprises:** an unqualified call finds functions in the namespaces of its argument types.
- **`extern "C"` cannot be applied to overloaded functions**, and a C caller linking a C++ function without `extern "C"` (or the other way around) gets `undefined reference` (Q1).
- Java: static-type overload resolution (`f(Object)` vs. `f(String)` with an `Object`-typed variable holding a `String`), the boxing and varargs phases, and erasure clashes.

### Tricky questions and answers

#### Q1 \[SDE-1/2\]: A C program links against your C++ library and fails with `undefined reference to 'compute'`, though `nm` shows the library defines `compute`. Why, and how do you fix it, including for an overloaded function?

```cpp
// lib.h                          // lib.cpp
int compute(int x);               int compute(int x)    { return x * 2; }
double compute(double x);         double compute(double x) { return x * 2; }
```

**Answer:** `nm` shows `_Z7computei` and `_Z7computed` (mangled), while the C compiler looks for the unmangled symbol `compute`. The linker's symbol table is flat, so the names never match.

Fix: give the C-visible API **C linkage** in a header shared by both languages, with _one_ function per C name:

```cpp
// lib.h  (valid C and C++)
#ifdef __cplusplus
extern "C" {
#endif
int    compute_i(int x);          // symbol: compute_i
double compute_d(double x);       // symbol: compute_d  (two C names, no overloading in C)
#ifdef __cplusplus
}
#endif

// lib.cpp
#include "lib.h"
extern "C" int    compute_i(int x)    { return x * 2; }
extern "C" double compute_d(double x) { return x * 2; }
```

Applying `extern "C"` to _both_ `compute` overloads is an error ("conflicting declaration of C function"), since they would need the same symbol. The return type is not mangled for plain functions, so `int f(); double f();` fails at the _declaration_ stage, not at link time. As a bonus, C linkage gives a stable ABI that does not depend on the compiler's mangling or on `` types.

#### Q2 \[SDE-2/3\]: Predict the output. Which calls surprise people, and why?

```cpp
void f(int)          { puts("f(int)"); }
void f(double)       { puts("f(double)"); }
void f(const char*)  { puts("f(const char*)"); }
void f(bool)         { puts("f(bool)"); }
template <class T> void f(T) { puts("f<T>"); }

f(1);          // (a)
f(1.0f);       // (b)
f('a');        // (c)
f("hi");       // (d)
f(nullptr);    // (e)
f(1L);         // (f)
f(true);       // (g)
```

**Answer:**

- **(a) `f(int)`**: exact match from both `f(int)` and `f<int>`, so the **non-template wins the tie**.
- **(b) `f<T>`**: the template deduces `float` (exact). `f(double)` needs a _promotion_, which ranks lower, so the **template wins**.
- **(c) `f<T>`**: `T = char` is exact, while `f(int)` needs a promotion.
- **(d) `f(const char*)`**: `"hi"` is a `const char[3]`. Array-to-pointer is an _exact-match_ conversion for both `f(const char*)` and `f<const char*>`, so the **non-template wins**.
- **(e) `f<T>`**: `T = nullptr_t` is exact, while `f(const char*)` needs a null-pointer conversion. (A trap: you would expect the pointer overload.)
- **(f) `f<T>`**: `T = long` is exact, and every non-template needs a conversion. (Without the template this call would be ambiguous.)
- **(g) `f(bool)`**: exact non-template.

Lesson: a catch-all `template<class T> void f(T)` **outbids every overload that needs a promotion or conversion**. Constrain it (SFINAE with `enable_if`, or C++20 concepts) or give it a distinct name. Interviewers use this to test whether you know the _rank order_, not just "closest type."

## 4.2: Runtime Polymorphism & Dynamic Binding

### The idea in plain words

Now the opposite situation. You have a pointer to a `Shape`, but you do not know at compile time whether the object behind it is a `Circle` or a `Square`. You still want `shape->area()` to run the _right_ version.

That is **runtime polymorphism**. The choice of function is made **while the program runs**, based on the object's **dynamic type** (the type it was actually created as), not the **static type** of the pointer or reference you used (the type written in the code). This is also called **dynamic binding** or _late binding_.

It needs three ingredients:

1. A **virtual** function.
2. A **pointer or reference** (not a copy).
3. An object whose real type is different from the pointer's type.

### What is static and what is dynamic

- The **static** type decides: which function _name and signature_ is chosen, the **access check**, and the **default arguments** (Topic 3.5).
- The **dynamic** type decides: **which override runs.**

Rules to remember:

- **Only through pointers or references.** A by-value `Base b = derived;` slices (Module 1), and its dynamic type _is_ `Base`.
- **Only virtual functions.** Non-virtual members are bound by the static type. A derived function with the same name merely _hides_ the base one (Topic 3.5).
- **Data members are never polymorphic.** `base->x` always reads `Base::x`.

### What a virtual call costs

The compiler emits: _load the vptr from the object → load the function pointer from the vtable → indirect call_ (Topic 4.3 gives the full walkthrough).

- That is an extra dependent load plus an **indirect branch**, which the CPU predicts with its branch target buffer.
- A well-predicted call site costs about as much as a normal call.
- A **randomly alternating target** causes mispredictions, on the order of 10–20 cycles each.
- The bigger cost is that **inlining is blocked**, which disables downstream optimizations (constant propagation, vectorization).

**Devirtualization** (the compiler turns the call into a direct call) happens when the dynamic type is provable: a `final` class, an object created in the same scope, link-time optimization (`-flto`), or profile-guided speculative devirtualization with a guard.

### Overloading is not dynamic

`visit(Shape&)` vs. `visit(Circle&)` is chosen from the _static_ type. So mixing overloads with virtual functions needs **double dispatch** (the Visitor pattern, Q2).

### `dynamic_cast` and RTTI

`dynamic_cast<D*>(b)` reads the type information that the vtable points to (4.3) and walks the hierarchy at run time. It is far slower than a virtual call, and a chain of `dynamic_cast`s signals a missing virtual function (an Open/Closed violation).

### Java comparison

Every instance method is virtual unless it is `private`, `static` or `final`. HotSpot's JIT profiles each call site:

- **Monomorphic** (one receiver type): inlined behind a type guard.
- **Bimorphic** (two types): two guarded inlines.
- **Megamorphic** (many types): falls back to a vtable call (`invokevirtual`) or an itable search (`invokeinterface`).

Class-hierarchy analysis can also devirtualize and later _deoptimize_ if a new subclass loads. Java binds overloads statically and has the same double-dispatch trap. Modern Java answers it with **sealed types plus pattern-matching `switch`** instead of Visitor. Fields are hidden, not overridden.

### Code: dynamic dispatch, and what is _not_ dynamic

```cpp
#include <cmath>
#include <iostream>
#include <memory>
#include <string>
#include <vector>

struct Shape {
    virtual ~Shape() = default;
    virtual double      area() const = 0;                        // dynamic dispatch
    virtual string name() const { return "Shape"; }
    // NON-virtual "template method": statically bound itself, but its INTERNAL calls to
    // area()/name() are dynamic, so derived classes customize behavior without re-implementing it.
    string describe() const { return name() + " area=" + to_string(area()); }
    int tag = 0;                                                  // data members are NOT polymorphic
};
struct Circle final : Shape {                                     // final => calls on Circle& can devirtualize
    double r;
    explicit Circle(double r) : r(r) {}
    double      area() const override { return M_PI * r * r; }
    string name() const override { return "Circle"; }
    int tag = 1;                                                  // HIDES Shape::tag (a second field!)
};
struct Square final : Shape {
    double s;
    explicit Square(double s) : s(s) {}
    double      area() const override { return s * s; }
    string name() const override { return "Square"; }
};

int main() {
    vector<unique_ptr<Shape>> shapes;                   // static type Shape, dynamic types differ
    shapes.push_back(make_unique<Circle>(1.0));
    shapes.push_back(make_unique<Square>(2.0));
    for (const auto& s : shapes)
        cout << s->describe() << '\n';                       // Circle area=3.14..., Square area=4.0...
                                                                  // each s->area(): vptr load, slot load, call
    Circle c(1.0);
    Shape* p = &c;
    cout << p->tag << ' ' << c.tag << '\n';                  // 0 1: field access uses the STATIC type
    c.area();                                                     // Circle is final: direct call, inlinable
}
```

Look at `describe()`. It is a normal non-virtual function, but inside it calls `name()` and `area()`, which _are_ virtual. So each shape customizes the result without re-writing `describe()`. This is the **Template Method** idea.

And look at `tag`: `Circle` declares its own `tag`, which does _not_ replace `Shape::tag`. It creates a second field. Reading it through a `Shape*` gets the base one (0), reading it through a `Circle` gets the derived one (1).

### Real-world picture

Dynamic binding is **service discovery with a load balancer**. Callers hold a _stable logical name_ (the base type and slot), and at request time the registry resolves it to whichever concrete instance backs it. It is flexible (hot-swap implementations, plugins), but every call pays a lookup, and the callee cannot be inlined into the caller (no cross-service optimization).

`final` is **pinning a service to a fixed address** so the call can be compiled in directly.

Overload-vs-override confusion is **routing on the request's declared schema (static) versus the actual backend that serves it (dynamic)**.

### Interview traps

- **Slicing:** by-value parameters and containers (`vector<Base>`) kill polymorphism.
- **Non-virtual hiding:** `a.stat()` through a `Base&` calls `Base::stat` even though `Derived::stat` exists.
- **Data members do not dispatch** (`p->tag`), and Java field hiding behaves the same.
- **Default arguments** use the static type (Topic 3.5).
- **Virtual calls in constructors and destructors** (Topic 3.3).
- **Overload vs. override mix-ups** (`draw(Shape&)` vs. `draw(Circle&)`): this needs double dispatch.
- **Performance questions:** "What does a virtual call cost?" Do not say "slow." Explain the extra load + indirect branch + lost inlining, _then_ say it depends on prediction and whether the call is hot. Mention devirtualization (`final`, LTO, PGO), sorting or batching by type, and `variant` or templates (4.6).
- **`dynamic_cast` chains** instead of virtual functions, and `dynamic_cast` cost and `-fno-rtti` environments.
- **`vector<unique_ptr<Base>>` pointer chasing** (Module 1): per-object cache misses.
- Java: JIT inline caches, why one rarely loaded subclass can _speed up or slow down_ a hot loop (megamorphic call site), and `final`/sealed as optimization hints.

### Tricky questions and answers

#### Q1 \[SDE-1/2\]: What does this print?

```cpp
struct A {
    virtual void who() const { puts("A"); }
    void stat() const        { puts("stat A"); }
    int v = 1;
};
struct B : A {
    void who() const override { puts("B"); }
    void stat() const         { puts("stat B"); }
    int v = 2;
};
void byVal(A a)        { a.who(); }
void byRef(const A& a) { a.who(); a.stat(); printf("%d\n", a.v); }
int main() { B b; byVal(b); byRef(b); }
```

**Answer:**

```
A        <- byVal: copy-initializes an A (slicing); the dynamic type of the copy is A
B        <- byRef: virtual dispatch through the reference reaches B::who
stat A   <- stat() is non-virtual: bound to the STATIC type A (B::stat merely hides it)
1        <- data members aren't polymorphic: A::v, not B::v
```

Fixes: pass polymorphic objects by reference, pointer or smart pointer. Make `stat` virtual if it should vary by dynamic type. Avoid same-named data members in hierarchies (`-Wshadow`).

#### Q2 \[SDE-3\]: `draw(Shape&)` and `draw(Circle&)` exist, but calling `draw(*shapePtr)` always picks `draw(Shape&)`. Why, and how do you get dispatch on **both** the shape type and the "renderer" type?

```cpp
struct Shape  { virtual ~Shape() = default; };
struct Circle : Shape {};
void draw(Shape&)  { puts("draw(Shape&)"); }
void draw(Circle&) { puts("draw(Circle&)"); }
Circle c; Shape& s = c;
draw(s);          // prints draw(Shape&), not draw(Circle&)
```

**Answer:** Overload resolution happens at **compile time on static types**. `s` is a `Shape&`, so `draw(Circle&)` is not even viable (there is no implicit downcast), and the dynamic type is never consulted. Virtual dispatch selects on the dynamic type of _one_ object (the receiver).

To dispatch on two, use **double dispatch (the Visitor pattern)**. The first virtual call (`accept`) resolves the shape's dynamic type. Inside it, `*this` has the _exact static type_, so the second call picks the right `visit` overload:

```cpp
struct Circle; struct Square;
struct Visitor {                                     // the "renderer"/operation side
    virtual ~Visitor() = default;
    virtual void visit(Circle&) = 0;
    virtual void visit(Square&) = 0;
};
struct Shape {
    virtual ~Shape() = default;
    #virtual void accept(Visitor&) = 0;               // dispatch #1: on the shape's dynamic type
};
struct Circle : Shape {
    double r = 1;
    void accept(Visitor& v) override { v.visit(*this); }   // *this is Circle& STATICALLY => visit(Circle&)
#};                                                         // dispatch #2: virtual call on the visitor
struct Square : Shape {
    double s = 2;
    void accept(Visitor& v) override { v.visit(*this); }
};
struct AreaVisitor : Visitor {
    double total = 0;
    void visit(Circle& c) override { total += 3.14159 * c.r * c.r; }
    void visit(Square& s) override { total += s.s * s.s; }
};
// Shape& any = ...;  AreaVisitor av; any.accept(av);   // both types resolved at runtime
```

Trade-offs: adding a **new operation** is easy (write a new `Visitor`), but adding a **new shape** forces changes to every visitor (this is the _expression problem_). For a closed set of types, `variant` plus `visit` (4.6) does the same without a hierarchy. In Java, sealed interfaces with pattern-matching `switch` give exhaustive checking and remove most Visitor boilerplate.

## 4.3: Under the Hood: VTABLE and VPTR

### The idea in plain words

How does a program "choose at run time"? With a simple trick:

- Each polymorphic **class** has one table of function pointers, called the **vtable**. It is created once and never changes.
- Each polymorphic **object** stores one hidden pointer, called the **vptr**, that points to its class's vtable.

To call `shape->area()`, the program follows the object's vptr to the vtable, finds the `area` entry at a fixed position, and jumps to whatever address is stored there. A `Circle` object's vptr points to the Circle table (whose `area` entry is `Circle::area`). A `Square`'s points to the Square table. Same position, different function. That is the whole trick.

You can think of the vtable as a **restaurant menu** and the vptr as the **menu the waiter is holding for your table**. #The order ("item #2") is always the same, but the kitchen behind each menu is different.

### How it works inside (Itanium ABI)

#### Where things live

- **One vtable per class** (in `.rodata` or `.data.rel.ro`, read-only memory), **one vptr per object** (8 bytes at offset 0 of the primary base).
- The vtable is emitted in the translation unit that defines the class's **key function**: the _first non-inline, non-pure virtual function_. If there is none, it is emitted (as a weak/COMDAT symbol) in every translation unit that needs it.

#### The layout

The vptr points to an _address point_ in the middle of the table:

```
Circle vtable                                  offset from vptr
  offset-to-top  (0)                             -16
  &typeinfo for Circle  (RTTI)                    -8
vptr ->  &Circle::~Circle()  (D1, complete)        0    slot 0
         &Circle::~Circle()  (D0, deleting)        8    slot 1
         &Circle::area() const                    16    slot 2
         &Circle::scale(double)                   24    slot 3
```

- Slots follow **declaration order, base class first**.
- A derived vtable is the base's slots with overridden entries replaced and new virtuals appended. So a given function always has the _same slot index_ in every class of the hierarchy. That is what makes dispatch through a `Shape*` work.
- A virtual destructor occupies **two slots** (D1 and D0, see 4.5).
- Pure virtuals point to `__cxa_pure_virtual`, which aborts the program.

#### A virtual call, step by step

For `p->area()` (`Shape* p`, and `area` is slot 2):

1. _Compile time:_ the static type `Shape` tells the compiler that `area` is slot 2, i.e. byte offset 16.
   #2. _Run time, load #1:_ read the vptr from the object (`[p + 0]`).
   #3. _Run time, load #2:_ read the function pointer at `vptr + 16`.
2. _Run time:_ indirect `call`, with `this` (the object address) already in `rdi`.

```asm
mov  rax, QWORD PTR [rdi]        ; rax = vptr            (rdi = p = this)
mov  rax, QWORD PTR [rax+16]     ; rax = vtable[slot 2] = &Circle::area
call rax                         ; return value (double) in xmm0
```

That is two **dependent loads** plus an indirect branch. The vtable is tiny and shared, so it is almost always in L1 or L2 cache, and the object's first cache line is usually needed anyway.

#### Who sets the vptr

Every constructor stores its own class's vtable address into the vptr **after its base subobjects finish and before its member initializers and body run**. Each destructor resets it on entry. This is _precisely_ why virtual calls in constructors and destructors resolve to the current class (Topic 3.3).

#### Multiple inheritance and virtual bases

- Each polymorphic base subobject has its own vptr and **secondary vtable**. When `C : A, B` overrides `B::g`, the secondary vtable's entry points at a **thunk** that adjusts `this` by the subobject offset and then jumps to `C::g` (Q2).
- `offset-to-top` is what `dynamic_cast<void*>` uses to find the _complete_ object.
- **Virtual bases** add negative-index entries (vbase offsets, vcall offsets) to the vtable (Topic 3.2).

#### RTTI and size

- **RTTI:** `typeid` and `dynamic_cast` read the `typeinfo` pointer at `vptr[-1]`. `-fno-rtti` drops those, but virtual dispatch still works.
- **Size cost:** +8 bytes per object. Total vtable memory is `slots × 8` per _class_, not per object.
- **MSVC** also keeps the vptr at offset 0, but its vtable has an RTTI "Complete Object Locator" at `[-1]` and no offset-to-top.
- **See it yourself:** `g++ -fdump-lang-class -c x.cpp` (older GCC: `-fdump-class-hierarchy`) or `clang++ -Xclang -fdump-vtable-layouts -c x.cpp` print the layouts.

#### Java comparison

The object header holds a **klass pointer** (not a vptr into a per-function table), and the `Klass`/`InstanceKlass` structure has the vtable embedded.

- `invokevirtual` loads the klass, indexes the vtable, and calls (in the interpreter). The JIT usually inlines it behind a guard instead.
- `invokeinterface` first locates the interface's **itable** (a scan, softened by inline caches).
- `invokespecial` (constructors, `super.f()`, private methods) and `invokestatic` are direct calls with no table at all.

### Code: build a vtable by hand, then peek at the real one

```cpp
#include <cstdio>
#include <cstring>
#include <typeinfo>

// Part 1: the compiler's job, done by hand in plain C-style code
struct ShapeVT {                                   // "vtable": one per class, constant
    double (*area)(const void* self);              // slot 0
    void   (*destroy)(void* self);                 // slot 1 (deleting destructor)
};
struct ShapeObj  { const ShapeVT* vt; };           // the vptr lives at offset 0
struct CircleObj { ShapeObj base; double r; };     // "derived": base subobject FIRST (Topic 3.1)

static double circle_area(const void* self) {
    auto* c = static_cast<const CircleObj*>(self); // downcast: same address (offset 0)
    return 3.14159 * c->r * c->r;
}
static void circle_destroy(void* self) { delete static_cast<CircleObj*>(self); }
static const ShapeVT kCircleVT = { circle_area, circle_destroy };   // constant, shared by all circles

CircleObj* make_circle(double r) { return new CircleObj{{&kCircleVT}, r}; }  // "constructor" stores the vptr
double area(const ShapeObj* s) { return s->vt->area(s); }   // THE virtual call: load vt, load fn, call
void   destroy(ShapeObj* s)    { s->vt->destroy(s); }       // virtual destruction

// Part 2: peek at the real thing (Itanium ABI only; diagnostic, NEVER ship this)
struct Shape  { virtual ~Shape() = default; virtual double area() const = 0; };
struct Circle : Shape { double r = 1; double area() const override { return r; } };
struct Square : Shape { double s = 2; double area() const override { return s; } };

static void* vptrOf(const void* obj) { void* vp; memcpy(&vp, obj, sizeof vp); return vp; }

int main() {
    ShapeObj* mine = &make_circle(2.0)->base;
    printf("hand-made vcall: %f\n", area(mine));            // 12.56636
    destroy(mine);

    Circle c1, c2; Square s;
    printf("same class => same vptr: %d\n", vptrOf(&c1) == vptrOf(&c2));   // 1
    printf("diff class => diff vptr: %d\n", vptrOf(&c1) != vptrOf(&s));     // 1
    void** vt = static_cast<void**>(vptrOf(&c1));
    auto* ti  = static_cast<const type_info*>(vt[-1]);      // RTTI pointer sits just BEFORE the vptr
    printf("typeinfo says: %s\n", ti->name());               // "6Circle" (mangled)
}
```

Part 1 is the best way to _really_ understand virtual calls. It builds exactly what the compiler builds: a constant table of function pointers (`kCircleVT`), an object whose first field points to that table, a "constructor" (`make_circle`) that stores the pointer, and a "virtual call" (`area`) that is just `s->vt->area(s)`. Nothing magic is left.

### Real-world picture

A vtable is a **service's routing table (an RPC method table)**, and the vptr is the **service handle stored in each client session**. Every instance of the "Circle service" shares one immutable routing table, and each session stores only a pointer to it. A call looks up the method _by index_ and dispatches.

Thunks are **adapter or proxy hops** that translate a request addressed to the secondary interface into the primary object's coordinates.

The "key function" rule means **one team owns publishing the routing table**. If that team never ships it, you get the link-time equivalent of a 404 (Q1).

### Interview traps

- **"Explain how a virtual call works at the instruction level."** Expected: vptr load, slot load, indirect call, with `this` in the first argument register. Add _where the vptr is set_ (constructors) and _what the cost is_ (2 loads + indirect branch + lost inlining).
- **`sizeof` questions:** a class with any virtual function gets +8, and two polymorphic bases give two vptrs.
- **`undefined reference to vtable for X`:** a declared-but-undefined key function (Q1).
- **Is the vtable per object or per class?** Per class. The vptr is per object.
- **Can you call a virtual function on a deleted, moved-from or uninitialized object?** The vptr is garbage or stale, and the call jumps to arbitrary memory. This is the classic exploit primitive (use-after-free vtable hijack). Know that **CFI and vtable verification** and `-fsanitize=cfi` exist.
- **Memory-copying polymorphic objects** (`memcpy`, or `memset(this, 0, ...)` in a constructor) overwrites the vptr, so do not.
- **Virtual functions in constructors and destructors** (the vptr changes).
- **Inline virtual functions** still go through the vtable unless devirtualized, and in a header-only class every translation unit may emit its own copy of the vtable.
- Java: "How does `invokeinterface` differ from `invokevirtual`?", "Why can adding a subclass slow a hot loop?", and "What is CHA/deoptimization?"

### Tricky questions and answers

#### Q1 \[SDE-2\]: The build fails with `undefined reference to 'vtable for Foo'` (and `Foo::~Foo()`). Nothing calls a function that isn't defined. Why?

```cpp
// foo.h
struct Foo {
    virtual ~Foo();            // declared, never defined anywhere
    virtual void run();
    int x;
};
// main.cpp
#include "foo.h"
int main() { Foo f; }
```

**Answer:** `Foo`'s constructor must store the vptr in `f`, which needs the **address of Foo's vtable**. The compiler emits the vtable only in the translation unit that defines the **key function** (the first non-inline, non-pure virtual function), which here is `~Foo()`. That translation unit does not exist, so the symbol is missing and the link fails. The vtable is also referenced when `f` is destroyed (`Foo::~Foo`).

Fixes:

```cpp
// foo.cpp: define the key function in exactly ONE translation unit
#include "foo.h"
Foo::~Foo() = default;
void Foo::run() {}
// OR make all virtuals inline/defaulted in the header: the vtable is then emitted (weakly) wherever it is used:
struct Foo { virtual ~Foo() = default; virtual void run() {} int x; };
```

Interview tip: mention the key-function rule by name. It is also why putting at least one non-inline virtual function (typically the destructor) in a `.cpp` is a good idea for big hierarchies. It emits **one** vtable and typeinfo instead of one per translation unit, which cuts binary size and link time.

#### Q2 \[SDE-3\]: `C : A, B`, and `C` overrides `B::g()`. Trace exactly what happens for `B* pb = &c; pb->g();`, then `delete pb;`. Where does the pointer adjustment happen?

```cpp
struct A { virtual ~A() = default; virtual void f() {} double a; };
struct B { virtual ~B() = default; virtual void g() {} double b; };
struct C : A, B { void g() override { /* uses this->a and this->b */ } double c; };
```

**Answer:** Layout (Topic 3.1): the `A` subobject is at 0 (vptr + `a`), the `B` subobject is at 16 (its own vptr + `b`), and `c` is at 32.

1. `B* pb = &c;` is an upcast to a **secondary base**. The compiler **adds 16**, so `pb` points at `B`'s subobject, whose vptr refers to **`C`'s secondary vtable for `B`**.
2. `pb->g()` is _slot 2_ of `B`'s vtable layout (after D1 and D0). Load `pb`'s vptr, load slot 2, and make the indirect call with `this = pb` (**the B-subobject address**).
3. But `C::g` expects `this` to be the **`C*`** (offset 0). So the secondary vtable's slot points at a **thunk** generated for `C::g`:

   ```asm
   non-virtual thunk to C::g():
       sub  rdi, 16          ; this = (C*)((char*)this - 16)
       jmp  C::g()           ; tail call, no extra frame
   ```

4. `delete pb;`: with a virtual destructor, the call goes through `B`'s secondary vtable to a **destructor thunk** that likewise subtracts 16 before running `C`'s deleting destructor. That way `operator delete` receives the _original_ allocation address (the `A` subobject address, `&c`).

   Without a virtual destructor, `delete pb` would pass `&c + 16` to `operator delete`, which is **heap corruption**, not just a leak.

   `offset-to-top` (stored at `vptr[-2]` in `B`'s secondary vtable) records that **−16** for `dynamic_cast<void*>`.

So the adjustment happens in two places: **at the upcast site** (+16, caller side) and **inside the thunk** (−16, callee side). `reinterpret_cast` and `void*` round-trips skip the first, which is why Topic 3.1's Q2 crashes.

## 4.4: Abstract Classes vs. Interfaces

### The idea in plain words

Sometimes a class is only a _promise_, not something you can build.

- An **abstract class** is a half-finished blueprint. It has at least one **pure virtual function** (written `= 0`), which means "children must provide this." You cannot create an object of an abstract class. But it may still hold data, constructors and finished methods.
- An **interface** is a _pure promise_: only abstract operations, no data. It says "anything that can do these things." C++ has no `interface` keyword (an interface is just an abstract class with no data, by convention). Java has a real `interface` keyword.

A quick way to choose: use an **interface** for a _capability_ ("can store," "can be closed") that many unrelated classes may offer. Use an **abstract class** for a _skeleton_ that already does the boring, shared part and leaves a few blanks to fill in.

### How they work inside

#### In C++

- An abstract class still gets a **vtable**. Pure virtual slots point at `__cxa_pure_virtual`, which prints "pure virtual method called" and aborts if it is ever reached (for example by a call from a constructor).
- Its constructors **do run**, as part of the derived object's chain (Topic 3.3).
- A pure virtual function **may have a body**, callable only through a qualified name (`Base::f()`).
- A **pure virtual destructor must have a definition** (4.5, Q1).
- Interface-style classes should have **no data, a virtual destructor, and a deleted copy** (anti-slicing).
- Multiple interface inheritance is cheap, but a shared interface base needs `virtual` inheritance (Topic 3.2).

#### In Java

- An **abstract class** has instance fields, constructors, any access level, and uses up the single-inheritance slot.
- An **interface** has `public static final` constants only, abstract methods, plus `default`, `static`, and (Java 9+) `private` methods. A class can implement many interfaces. In the class file an interface is `ACC_INTERFACE | ACC_ABSTRACT`. `default` methods are real bytecode bodies in the interface.
- Calls through an interface type use **`invokeinterface`** (an itable lookup, usually hidden by inline caching). Calls through an abstract class use `invokevirtual` (a vtable index).

#### Evolution and ABI

- Adding a pure virtual function to a published interface breaks **every implementer at compile time** (they become abstract) and changes the vtable layout (an **ABI break**, Topic 2.5).
- Adding a _non-pure_ virtual function is source-compatible but still shifts or extends vtables.
- Java's `default` methods exist for exactly this reason, at the price of the diamond conflicts in Topic 3.2.

#### Design guidance

- Use an **interface** to express a _capability or role_ with many unrelated implementers, and to apply Dependency Inversion and Interface Segregation (small, focused interfaces).
- Use an **abstract class** for a _skeleton_ that implements the invariant parts (the **Template Method / NVI** pattern, Topic 2.1) and leaves hooks, like Java's `List` plus `AbstractList` ("skeletal implementation").
- Combine them: **interface for the contract, abstract class for the reusable skeleton, concrete class for the specifics.**

#### Contract design checklist

Document: pre/post-conditions and invariants (LSP, Topic 3.4), **ownership** (who deletes? return `unique_ptr`), **lifetime** of returned references and views (Topic 2.4), **const-correctness**, **exception and `noexcept` guarantees**, **thread safety**, and a **versioning** strategy (Q2).

#### Compile-time interfaces

Templates are "duck-typed" interfaces (and C++20 concepts name and check them). They are covered in 4.6.

### Code: interface, skeleton, concrete class

```cpp
#include <cstddef>
#include <optional>
#include <stdexcept>
#include <string>
#include <type_traits>
#include <unordered_map>
#include <utility>

// INTERFACE: contract only. No data, no logic, deleted copy (anti-slicing).
class IStorage {
public:
    IStorage() = default;
    IStorage(const IStorage&)            = delete;
    IStorage& operator=(const IStorage&) = delete;
    virtual ~IStorage() = default;                                    // public virtual: deletable via IStorage*
    virtual void put(const string& key, string value) = 0;
    virtual optional<string> get(const string& key) const = 0;
};
static_assert(is_abstract_v<IStorage>, "cannot be instantiated");

// ABSTRACT CLASS: skeletal implementation shared state + invariant logic + hooks =====
class StorageBase : public IStorage {
    size_t maxKey_;                                              // state: not allowed in a Java interface
    size_t puts_ = 0;
protected:
    explicit StorageBase(size_t maxKey) : maxKey_(maxKey) {}     // runs in the ctor chain of the concrete class
    virtual void doPut(const string& key, string value) = 0;   // hook
public:
    void put(const string& key, string value) final {       // NVI + final: the contract is enforced ONCE
        if (key.empty() || key.size() > maxKey_) throw invalid_argument("bad key");
        ++puts_;
        doPut(key, move(value));
    }
    size_t putCount() const noexcept { return puts_; }
};

// CONCRETE
class MemStorage final : public StorageBase {
    unordered_map<string, string> m_;
    void doPut(const string& k, string v) override { m_[k] = move(v); }
public:
    MemStorage() : StorageBase(256) {}
    optional<string> get(const string& k) const override {
        auto it = m_.find(k);
        return it == m_.end() ? nullopt : optional<string>(it->second);
    }
};

// A pure virtual function CAN have a body: a default the subclass must opt into explicitly
struct Walker { virtual ~Walker() = default; virtual void walk() = 0; };
void Walker::walk() { /* shared default behavior */ }
struct Dog : Walker { void walk() override { Walker::walk(); /* then dog-specific */ } };
```

Read this from top to bottom as three layers. `IStorage` says _what_. `StorageBase` does the shared checking once (note `put` is `final`, so no subclass can skip the key validation) and leaves one blank, `doPut`. `MemStorage` fills in that blank.

### Real-world picture

An **interface** is an **OpenAPI or gRPC `.proto` contract**: a pure description of operations that any number of independent services can implement, with no behavior and no state.

An **abstract class** is a **framework's base handler or SDK scaffold**: it gives you request validation, metrics, retries and logging, and asks you to fill in just `doHandle()`.

Adding a method to the published contract is a **breaking API change** that needs a versioning plan. Adding a protected hook to the scaffold only affects the services that inherit it.

### Interview traps

- **"Abstract class vs. interface?"** A weak answer: "interfaces have no implementation." A strong answer covers state, constructors, access control, inheritance slots, evolution and ABI, typical use (capability vs. skeleton), and the JVM invocation difference (`invokeinterface`/itable vs. `invokevirtual`/vtable).
- **A pure virtual destructor without a body** gives a link error (Q1).
- **Calling a pure virtual function from a constructor or destructor** is UB ("pure virtual method called").
- **An abstract class with a public non-virtual destructor** deleted through a base pointer (4.5).
- **Interfaces with data members or constructors with logic:** it is now an abstract class, and the diamond duplicates state.
- **Fat interfaces** that violate ISP, and **adding a method** to a published interface (Q2).
- **Default-method conflicts in Java** (Topic 3.2), and the fact that interface `default` methods cannot use instance state.
- **Returning raw pointers from interface factories** (unclear ownership).
- Java: "Can an interface have a constructor?" (No.) "Can an abstract class implement an interface without implementing its methods?" (Yes.) "Why are interface fields `static final`?" (No instance state.) "`Comparable` vs. `Comparator` as contracts."

### Tricky questions and answers

#### Q1 \[SDE-2\]: Why does this fail to link? Why must a _pure_ virtual destructor have a body?

```cpp
struct Base { virtual ~Base() = 0; };       // pure virtual destructor: Base is abstract
struct D : Base {};
int main() { D d; }                          // undefined reference to `Base::~Base()'
```

**Answer:** `= 0` makes `Base` abstract (useful when the class has no other virtual function to make pure). But **destructors are not overridden, they are chained**: `D::~D()` ends by _calling_ `Base::~Base()` (Topic 3.3). That call needs an actual body, so the linker looks for one and finds none. Pure virtual destructors are the **only** pure virtual functions that _must_ be defined.

Fix:

```cpp
struct Base { virtual ~Base() = 0; };
inline Base::~Base() = default;              // definition outside the class (inline if in a header)
```

#### Q2 \[SDE-3\]: Plugins compiled against `IStorage` v1 are in production. Add `remove()` without breaking them. What are your options in C++, and what is the Java answer?

**Answer:** Adding `virtual bool remove(const string&) = 0;` to `IStorage` breaks **source** compatibility (every plugin becomes abstract) and **binary** compatibility (old binaries' vtables lack the new slot, so a call reads past the end of the table and jumps to garbage).

Options:

**1. Extension interface plus a capability query (preferred):** leave `IStorage` frozen, add a new interface, and let callers discover it.

```cpp
class IStorageV2 : public IStorage {                 // new contract; v1 plugins are unaffected
public:
    virtual bool remove(const string& key) = 0;
};
// caller:
bool tryRemove(IStorage& s, const string& k) {
    if (auto* v2 = dynamic_cast<IStorageV2*>(&s)) return v2->remove(k);   // new plugins
    return false;                                                          // graceful fallback for v1
}
```

Caveat: `dynamic_cast` across **shared-library boundaries** compares `typeinfo`, which can fail with hidden symbol visibility or duplicated RTTI. For plugin ABIs, use a COM or `QueryInterface`-style method **designed into v1** (`virtual void* queryInterface(const char* id) noexcept`), or a **C-ABI function table whose first field is `struct_size`** (the pattern Win32 and Vulkan use: new fields are appended, and old binaries pass a smaller size).

**2. Add a _non-pure_ virtual function with a default body:** source-compatible, but only safe if every implementer is **recompiled** (the vtable grows). So it is an internal-API tactic, not a plugin-ABI one.

**3. Version the namespace or interface name** (`inline namespace v2`), and keep a v1 adapter.

**Java:** add a `default` method to the interface (`default boolean remove(String k) { return false; }`). Old implementers keep working, because the JVM resolves interface method bodies at link time by name and descriptor, with no baked-in slot numbers. The remaining risks are _behavioral_ (the default may be wrong for them) and the diamond conflicts of Topic 3.2.

## 4.5: Virtual Destructors

### The idea in plain words

Suppose you have a `Base*` that really points to a `Derived` object, and you `delete` it. Which destructor should run? Obviously `Derived`'s, because only `Derived` knows how to clean up its own members.

But a destructor is a function like any other. If it is **not** `virtual`, then `delete base_ptr` uses the pointer's _static_ type and runs only `~Base`. The derived part is never cleaned up: its memory, files and locks leak. This is not just a leak. It is **undefined behavior**.

A **virtual destructor** fixes this. It makes `delete base_ptr` run the destructor of the object's _dynamic_ type, just like any other virtual call.

The simple rule: **if a class has virtual functions and anyone might delete it through a base pointer, give it a public virtual destructor.**

### How it works inside

#### What `delete p` compiles to

**With a non-virtual `~Base`:** a _direct, statically bound_ call to `Base::~Base(p)`, then `operator delete(p, sizeof(Base))`. The compiler uses the **static type**. `Derived`'s destructor body never runs, and its members (`vector`, `string`, `unique_ptr`) are **never destroyed**. Their heap buffers, file descriptors, locks and sockets **leak**.

**With a virtual `~Base`:** `delete p` becomes _load vptr → load the **deleting destructor** slot (D0) → call_. `Derived`'s D0 runs `Derived`'s D1 (body, then members in reverse, then bases) and finally calls `operator delete(this, sizeof(Derived))`, **with the right pointer and size**.

#### It is worse than a leak

- With **sized deallocation** (C++14, on by default in GCC), `operator delete` receives `sizeof(Base)`, but the allocation was `sizeof(Derived)`. Size-class allocators (tcmalloc, jemalloc and others) use the size to choose the bin, so a wrong size can **corrupt allocator metadata or abort**.
- With multiple inheritance, `delete` may also pass a **pointer not equal to the allocation's start** (Topic 4.3, Q2): heap corruption.
- **Leak scale:** a hierarchy deleted through the base with a 1 KiB buffer in each derived object leaks about 1 GB per million objects. It is a slow-burn out-of-memory that only appears in long-running services.

#### Three destructor variants per class (Itanium)

- **D0**, _deleting_: destroys and frees. This is the one `delete` calls through the vtable.
- **D1**, _complete-object_: destroys and also runs virtual-base destructors.
- **D2**, _base-object_: destroys without the virtual bases. Used when this class is a base subobject.

The vtable's two destructor slots are D1 and D0.

#### Rules

- **Public virtual destructor** if the class has virtual functions and objects may be deleted through a base pointer (including by `unique_ptr<Base>`).
- Otherwise a **protected non-virtual destructor**: `delete basePtr` then fails to compile (good for mixins, policies, CRTP bases).
- Or make the class `final`.
- Warnings: `-Wnon-virtual-dtor` and `-Wdelete-non-virtual-dtor` (the second is in `-Wall`). Sanitizers: ASan's `new-delete-type-mismatch`, and LeakSanitizer.

#### Costs and side effects

- A virtual destructor in a class with no other virtual functions adds a **vptr (+8 bytes)** and a vtable, changing `sizeof` and POD-ness.
- Declaring any destructor suppresses implicit moves (Module 1), so interface bases write `= default` for the special members they need.

#### The `shared_ptr` accident

`shared_ptr` **captures the deleter at creation**, from the _original_ pointer type. So `shared_ptr<Base>(new Derived)` and `make_shared<Derived>()` destroy correctly even with a non-virtual destructor. This is a fragile accident that disappears with `unique_ptr<Base>` or a raw `delete` (Q1).

#### Java comparison

There are no destructors, so this bug class does not exist for memory (the GC frees by the object's real size and type). The virtual-destructor _role_ is played by `AutoCloseable.close()`, which is dispatched virtually and invoked by try-with-resources. If a subclass does not override `close()` to release its own resource, you **leak the resource** (file descriptors, sockets) instead of memory.

### Code: correct, broken, and guarded

```cpp
#include <cstdio>
#include <memory>
#include <vector>

static int g_live = 0;                                      // counts live Tracker objects (proves destruction)
struct Tracker { Tracker() { ++g_live; } Tracker(const Tracker&) { ++g_live; } ~Tracker() { --g_live; } };

// CORRECT: virtual destructor
struct Base {
    virtual ~Base() = default;                              // slot 0 (D1) + slot 1 (D0) in the vtable
    virtual void run() = 0;
};
struct Derived : Base {
    vector<char> buf = vector<char>(1 << 20);     // 1 MiB owned by the derived part
    Tracker           t;
    void run() override {}
};

// BROKEN (do not ship): virtual function + NON-virtual destructor
struct BadBase {
    ~BadBase() = default;                                   // -Wnon-virtual-dtor warns here
    virtual void run() = 0;
};
struct BadDerived : BadBase {
    vector<char> buf = vector<char>(1 << 20);
    Tracker           t;
    void run() override {}
};

// GUARDED: protected non-virtual dtor => deleting through the base does not COMPILE
class PolicyBase {
protected:
    ~PolicyBase() = default;                                // only derived classes may destroy it
};
struct Policy : PolicyBase {};

int main() {
    { unique_ptr<Base> p = make_unique<Derived>(); }      // ~Derived runs via the vtable
    printf("live after good: %d\n", g_live);                    // 0

    // BadBase* b = new BadDerived; delete b;     // UB: typically only ~BadBase runs => 1 MiB and one
    // printf("live after bad: %d\n", g_live);  // Tracker leak (live == 1), and operator delete gets
    //                                              // sizeof(BadBase), not sizeof(BadDerived). Run under
    //                                              // ASan/LSan to see the report.

    // PolicyBase* pb = new Policy; delete pb;    // COMPILE ERROR: ~PolicyBase is protected
    Policy ok;  (void)ok;                                              // fine: destroyed as Policy
}
```

The `Tracker` is a tiny trick to _prove_ destruction happened: it counts up when created and down when destroyed. After the good case, the count is back to 0. In the broken case (commented out), it would stay at 1.

### Real-world picture

A virtual destructor is the **teardown hook registered with the orchestrator**. When the platform deprovisions a "generic service" handle, it must call _that service's_ shutdown routine, not a generic one. Without it, Kubernetes deletes the pod object but **never runs the app's `preStop` cleanup**. Connections, temp files and cloud resources stay allocated, invisible until the bill or the OOM-killer arrives.

A protected non-virtual destructor is **RBAC**: only the owning subclass may tear the object down, and the generic handle simply cannot issue the delete.

### Interview traps

- **"Why must destructors be virtual in base classes?"** Expect to be asked what _exactly_ happens otherwise: UB, only the base destructor runs, derived members leak, plus the sized-delete and pointer-adjustment dangers.
- **`unique_ptr<Base>` vs. `shared_ptr<Base>`** with a non-virtual destructor (Q1).
- **The virtual destructor must be in the _topmost_ class that is deleted through.** Once `Base` declares it virtual, all derived destructors are virtual automatically, even without the keyword.
- **A pure virtual destructor without a body** gives a link error (4.4).
- **Throwing from a destructor** calls `terminate` (destructors are implicitly `noexcept`).
- **`virtual ~Base() = default;` suppresses implicit moves**, so declare the special members you need.
- **Virtual calls inside destructors** resolve to the current class (Topic 3.3).
- **Non-polymorphic standard classes** (`string`, `vector`, `unordered_map`) **have non-virtual destructors**, so never inherit from them to delete polymorphically (Topic 3.4).
- **Making the destructor virtual "just in case"** costs +8 bytes per object and a vtable. For billion-object arrays that matters.
- Java: "Is there a destructor?" (No; `finalize` is deprecated for removal.) "What plays the role of a virtual destructor?" (`AutoCloseable.close()` plus try-with-resources.) "How can Java leak resources through inheritance?" (A subclass that does not override `close()`.)

### Tricky questions and answers

#### Q1 \[SDE-1/2\]: `Base` has a non-virtual destructor. Which of these are safe, and why?

```cpp
struct Base { ~Base() { puts("~Base"); } };
struct Derived : Base {
    ~Derived() { puts("~Derived"); }
    vector<int> v = vector<int>(1000);
};
shared_ptr<Base> a = make_shared<Derived>();    // (1)
shared_ptr<Base> b(new Derived);                     // (2)
unique_ptr<Base> c = make_unique<Derived>();    // (3)
Base* d = new Derived;  delete d;                         // (4)
```

**Answer:**

- **(1) and (2) are safe in practice.** `shared_ptr` creates its control block from the _original_ pointer type (`Derived*`), and type-erases a deleter that does `delete static_cast<Derived*>(p)`. So it prints `~Derived` then `~Base`, even though `~Base` is not virtual.
- **(3) is UB.** `unique_ptr<Base>` has the deleter `default_delete<Base>`, which does `delete (Base*)p`: a statically bound `~Base`, `v` leaks, and the sized `operator delete` receives the wrong size.
- **(4) is UB** for the same reason.

This is a _safety net_, not a design. The first time someone passes a raw `Base*`, stores a `unique_ptr<Base>`, or switches smart-pointer types, it breaks silently. Fix by writing `virtual ~Base() = default;` (or `protected` and non-virtual if the class is not meant for polymorphic deletion), and compile with `-Wall -Wnon-virtual-dtor`.

#### Q2 \[SDE-3\]: Describe exactly what the compiler emits for `delete p` with and without a virtual destructor, what can go wrong beyond the leak, and give two designs that make the mistake impossible.

**Answer:**

```
// Non-virtual ~Base, p is Base* (dynamic type Derived):
call  Base::~Base(p)                      ; statically bound: Derived's members are never destroyed
call  operator delete(p, sizeof(Base))    ; WRONG size (and, with multiple inheritance, possibly a
                                          ;  pointer that is not the start of the allocation)

// Virtual ~Base:
mov   rax, [p]                            ; vptr
call  [rax + 8]                           ; slot 1 = D0 "deleting destructor" of the DYNAMIC type
//   Derived::D0(p):
//        call Derived::D1(p)             ; body -> members (reverse) -> Base::D2 (base-object dtor)
//        call operator delete(p, sizeof(Derived))   ; right pointer, right size
```

Beyond the leak:

1. A **sized-delete mismatch** corrupts size-class allocators or trips their checks.
2. With multiple inheritance, `operator delete` may receive a **non-allocation pointer** (see Topic 4.3, Q2), which means heap corruption.
3. **Non-memory resources** (locks, file descriptors, transactions, RAII guards) are never released.
4. **Destructor side effects** (flush, deregister, log) are skipped, giving silent data loss.

Designs that make the mistake impossible:

```cpp
// Design A: polymorphic base => public virtual destructor + deleted copy (anti-slicing)
struct IService { virtual ~IService() = default; /* ... */ };

// Design B: NOT polymorphic => protected non-virtual destructor: `delete basePtr` fails to COMPILE
struct Mixin { protected: ~Mixin() = default; };

// Design C: type-erased owner that always knows the dynamic type
template <class T, class... A> shared_ptr<IService> make(A&&... a) {
    return make_shared<T>(forward<A>(a)...);       // deleter bound to T at creation
}
```

Add `final` on leaf classes, `-Wnon-virtual-dtor` and `-Wdelete-non-virtual-dtor`, ASan plus LSan in CI, and a code-review rule: _"any class with a virtual function gets a virtual destructor or a protected one."_

## 4.6 Static Polymorphism — Templates, CRTP, `variant` & Type Erasure

### The idea in plain words

You have now seen runtime polymorphism (virtual functions). It is flexible, but it costs a pointer hop per call, and it requires every type to inherit from a common base.

There is another way to get "many forms": let the **compiler** write a separate, specialized version of your code for each type, at compile time. That is what **templates** do. This is called **static** (compile-time, or parametric) polymorphism.

```cpp
template <class T>
double total(const vector<T>& v) { /* calls x.area() on each element */ }
```

`total<Circle>` and `total<Square>` become two different, fully optimized functions. There is no vptr, no vtable, no indirect call. The "interface" is implicit: any type that has an `area()` method works (this is called **duck typing**). In C++20 you can name and check that interface with _concepts_.

The trade-off: the set of types must be known when you compile. You cannot load a new type from a plugin at run time.

This topic covers four tools for static or "no-base-class" polymorphism: templates, CRTP, `variant`, and type erasure. It ends with a table that tells you when to use each.

### How each tool works inside

#### Templates

- `totalArea<Circle>` and `totalArea<Square>` are _different functions_, each fully inlinable and vectorizable, with **no vptr, no vtable, no indirect call**.
- Costs: **code bloat** (one copy per type), longer compile times, and a **closed set at compile time** (you cannot load a type from a plugin).

#### CRTP (the Curiously Recurring Template Pattern)

`struct Sq : ShapeBase<Sq>`: the base is a template parameterized by the _derived_ class. Then `static_cast<Derived*>(this)` lets the base call derived code with **no virtual call**.

- The object has **no vptr**, so `sizeof` does not grow.
- Hazards: nothing verifies that `D` really derives from `Base<D>` (`struct Y : Base<X>` compiles and is UB), and the derived part is not yet constructed while `Base`'s constructor runs (Q1).

#### `variant<A, B, C>`

Holds exactly one of a **closed set** of types **inline** (size = largest member + an index, no heap, no pointer chasing, contiguous in a `vector`).

- `visit` dispatches via a **jump table on the index** (one fairly predictable indirect jump, but the visited lambda can be inlined per alternative).

#### Type erasure

`function`, `any`, and the `Shape` wrapper in the code below. A _value-semantic_ wrapper owns a `unique_ptr<Concept>` with a templated `Model<T>`.

- The interface is internal and **virtual** (one indirect call per operation, plus a heap allocation unless there is a small-buffer optimization).
- But user types **do not need to inherit from anything**: it is non-intrusive ("external polymorphism").
- This is also how `shared_ptr`'s deleter works.

#### Choosing between them

| Need                                                              | Best fit                                 |
| ----------------------------------------------------------------- | ---------------------------------------- |
| Open set of types, plugins, runtime-loaded code, stable ABI       | **Virtual** interface (or C-ABI table)   |
| Hot loop over homogeneous data, max inlining and vectorization    | **Templates / type-bucketed containers** |
| Closed, small set of types in one container                       | **`variant`** + `visit`             |
| Value semantics for unrelated types, no common base               | **Type erasure**                         |
| Compile-time policy or strategy injection, no per-object overhead | **CRTP / policy templates (EBO)**        |

#### Java comparison

Generics are **erased**: one compiled body, `Object`-typed internally, with boxing for primitives. So Java has _no_ #compile-time specialization. The **JIT** performs the equivalent at run time by inlining monomorphic call sites. C# generics are reified over value types, and Rust monomorphizes like C++ templates. Java's closed-set analogue is **sealed interfaces plus records plus pattern-matching `switch`**.

### Code: five ways to compute areas

```cpp
#include <cmath>
#include <memory>
#include <utility>
#include <variant>
#include <vector>

struct Circle { double r; double area() const { return 3.14159 * r * r; } };   // no base class at all
struct Square { double s; double area() const { return s * s; } };

// (1) Templates: duck typing, fully inlined per type
template <class T>
double totalArea(const vector<T>& v) { double s = 0; for (const auto& x : v) s += x.area(); return s; }

// (2) CRTP: static "inheritance", no vptr. Constructor is private + friend D, so only D can derive.
template <class D>
class ShapeBase {
    ShapeBase() = default;
    friend D;                                              // `struct Y : ShapeBase<X>` will NOT compile
public:
    double doubledArea() const { return 2 * static_cast<const D*>(this)->area(); }   // resolved at compile time
};
struct Tile : ShapeBase<Tile> { double s = 1; double area() const { return s * s; } };

// (3) Closed set: variant + visit (inline storage, jump table on the index)
using AnyShape = variant<Circle, Square>;
double area(const AnyShape& s) { return visit([](const auto& x) { return x.area(); }, s); }

// (4) Type erasure: value semantics, non-intrusive, open set, one virtual call inside
class Shape {
    struct Concept {
        virtual ~Concept() = default;
        virtual double area() const = 0;
        virtual unique_ptr<Concept> clone() const = 0;
    };
    template <class T> struct Model final : Concept {
        T v;
        explicit Model(T x) : v(move(x)) {}
        double area() const override { return v.area(); }
        unique_ptr<Concept> clone() const override { return make_unique<Model>(*this); }
    };
    unique_ptr<Concept> p_;
public:
    template <class T> Shape(T x) : p_(make_unique<Model<T>>(move(x))) {}   // any type with area()
    Shape(const Shape& o) : p_(o.p_->clone()) {}                // deep copy through the hidden interface
    Shape(Shape&&) noexcept = default;
    Shape& operator=(Shape o) noexcept { p_.swap(o.p_); return *this; }   // copy-and-swap
    double area() const { return p_->area(); }
};

// (5) Type-bucketed scene: each loop is monomorphic, inlinable, vectorizable
struct Scene {
    vector<Circle> circles;
    vector<Square> squares;
    double total() const { return totalArea(circles) + totalArea(squares); }
};

int main() {
    vector<Shape> mixed{Circle{1}, Square{2}};              // heterogeneous, value semantics, no base class
    double s = 0; for (const auto& x : mixed) s += x.area();
    AnyShape v = Square{3};  (void)area(v);
    Tile t; (void)t.doubledArea();
    return s > 0 ? 0 : 1;
}
```

Notice that `Circle` and `Square` have **no base class** and no `virtual` anywhere. They simply both happen to have an `area()` method. Templates, `variant` and type erasure all work with them as they are. That is the big freedom static polymorphism gives you.

The type-erased `Shape` class (4) is the most subtle. From the outside it looks like one ordinary value type. Inside, it hides a small virtual interface (`Concept`) and a template (`Model<T>`) that adapts any `T` with an `area()` method to that interface.

### Real-world picture

Templates are **compile-time code generation per tenant**: each tenant gets a build specialized for its schema, fast and inlined, but you rebuild for each new tenant and cannot add one at run time.

Virtual dispatch is a **multi-tenant service with a shared handler** and a run-time lookup.

`variant` is a **sealed, typed envelope** (a Protobuf `oneof`): one of a fixed set, stored inline in the message.

Type erasure is a **sidecar or adapter**: services that never agreed on a base contract are wrapped behind a uniform interface at the edge, at the price of one hop.

### Interview traps

- **"Virtual functions vs. templates?"** Answer in terms of _open vs. closed type sets, binary-boundary stability, inlining, code size and compile times._ Do not say "templates are faster" without conditions.
- **CRTP pitfalls:** the wrong derived type in `Base<X>`, using derived members in `Base`'s constructor, and no common base type (you cannot put them in one `vector<Base*>`).
- **Template code bloat and error messages**, and constraints (SFINAE with `enable_if`, or C++20 `requires`).
- **`variant` costs:** `sizeof` is the largest alternative (padding waste), `valueless_by_exception`, and adding an alternative recompiles every `visit`.
- **Type-erasure costs:** heap allocation, one virtual call, copy semantics via `clone()`, and the generic constructor hijacking copies (constrain it with `enable_if` so `Shape(Shape&)` is not intercepted in real code).
- **`function` overhead** vs. templated callables (a virtual-like call plus a possible allocation, vs. an inlined lambda).
- **Specializations vs. overloads** (4.1) in generic code.
- Java: "Why can't you do `new T()` or overload on `List<String>` vs. `List<Integer>`?" (erasure), and "What does the JIT do that C++ templates do at compile time?"

### Tricky questions and answers

#### Q1 \[SDE-2\]: What are the two bugs in this CRTP code? Fix them.

```cpp
template <class D> struct Base {
    Base() { static_cast<D*>(this)->init(); }
};
struct X : Base<X> { int v = 42; void init() { printf("%d\n", v); } };
struct Y : Base<X> {};
```

**Answer:**

1. **Calling derived code from the base constructor.** While `Base<X>::Base()` runs, `X`'s members have not been initialized (Topic 3.3: bases first, then members), so `init()` reads an **indeterminate `v`** (UB, typically garbage). A virtual call would have dispatched to the base version. CRTP silently reaches into the _unconstructed_ derived part, which is worse.
2. **`struct Y : Base<X>` compiles.** Nothing ensures `D` is really the derived class, so `static_cast<X*>(this)` on a `Y` is **UB** (a wrong-type downcast).

Fix: make the constructor private and befriend `D` (only `D` can inherit), and call the derived hook _after_ construction:

```cpp
template <class D> class Base {
    Base() = default;
    friend D;                                   // `struct Y : Base<X>` now fails to compile (Y isn't a friend)
public:
    void run() { static_cast<D*>(this)->init(); }   // called by the user once the object is fully built
};
struct X : Base<X> { int v = 42; void init() { printf("%d\n", v); } };
// X x; x.run();   // prints 42
```

#### Q2 \[SDE-3\]: You draw 100k shapes per frame from an open-ended set that designers extend via plugins, plus a closed core set. Compare virtual, `variant`, and templates, pick a design, and say how you would validate it.

**Answer:** Split by _who owns the type set_:

- **`vector<unique_ptr<Shape>>` + virtual:** simplest, and open to plugins. But each element is a separate allocation (**pointer chasing**, Module 1), each `draw()` is an indirect call whose target alternates unpredictably across shape types (branch mispredictions), and nothing inlines.
- **`vector<variant<...>>`:** objects are **contiguous and inline** (cache friendly), dispatch is an index jump table, and it is exhaustive at compile time. But it is a **closed** set: plugin types cannot join, and every `visit` recompiles when you add an alternative. `sizeof` is the largest alternative.
- **Type-bucketed containers + templates** (the `Scene` above): one `vector<T>` per type, and a templated loop per bucket. Each loop is **monomorphic** (perfect prediction, inlined, auto-vectorizable), with no dispatch at all. This is typically the fastest for the _core_ types.
- **Recommended: a hybrid.** **Closed core types in type-bucketed (or `variant`) storage** for the hot path, plus **one virtual `IPluginShape`** bucket for plugin types (that bucket pays the dispatch cost, but it is small). Sorting or batching by type also helps any virtual path, because it makes the indirect branch predictable.
- **Validate, do not guess:** write a microbenchmark with realistic type mixes and sizes, and compare `perf stat -e branch-misses,cache-misses,instructions` (or Instruments/VTune). Check devirtualization in the assembly (`-O2 -S`) and flip LTO and PGO on and off. Report the _measured_ cost per shape, plus binary size and compile time (templates have costs there).

The principle to state: **virtual = open set + stable boundary, templates/variant = closed set + speed, and measure before choosing.**

## Module 4 Cheat Sheet

| Concept                       | One-line interview answer                                                                                                                                                                                          |
| ----------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Overload resolution           | Compile-time, on static types. Ranks: exact > promotion > conversion > user-defined > ellipsis. A non-template beats a template on a tie, but an exact-match template beats a non-template that needs a promotion. |
| Return type                   | Not part of the signature or the mangled name (except in template instantiations), so you cannot overload on it.                                                                                                   |
| Name mangling                 | Encodes namespace, class, name, cv-qualifiers and parameters into a flat linker symbol (`_ZNK3Foo3barEi`). `extern "C"` disables it, with no overloading.                                                          |
| Static vs. dynamic type       | Static: overload choice, access, default args, data members, non-virtual calls. Dynamic: which virtual override runs.                                                                                              |
| Double dispatch               | Overloads do not see dynamic types, so use Visitor (`accept` → `visit(*this)`), `variant`, or Java sealed + pattern `switch`.                                                                                 |
| Virtual call                  | Load vptr → load slot → indirect call (`this` in `rdi`). +8 bytes per object, one vtable per class. Costs an extra load, an indirect branch, and lost inlining.                                                    |
| Vtable layout                 | `[offset-to-top][typeinfo] ← vptr → [D1][D0][virtual fns in declaration order]`. Derived tables keep the base slot order.                                                                                          |
| Who sets vptr                 | Each constructor, after bases and before members. That is why virtual calls in ctors and dtors hit the current class.                                                                                              |
| Key function                  | The first non-inline, non-pure virtual: it emits the vtable. Undefined → `undefined reference to vtable for X`.                                                                                                    |
| Multiple inheritance dispatch | Secondary vtable + `this`-adjusting thunk. The upcast adds the offset, and the thunk subtracts it.                                                                                                                 |
| Abstract class vs. interface  | Abstract: state, ctors, skeleton/NVI. Interface: pure contract or capability. Java adds `invokeinterface`/itable vs. `invokevirtual`/vtable, and `default` methods for evolution.                                  |
| Pure virtual destructor       | Makes the class abstract, but **must have a body** (the chain calls it).                                                                                                                                           |
| Virtual destructor            | Without it, `delete base*` is UB: derived members leak, wrong size to `operator delete`, possible heap corruption. Public virtual, or protected non-virtual, or `final`.                                           |
| `shared_ptr` vs. `unique_ptr` | `shared_ptr` captures the deleter by the original type (safe by accident), while `unique_ptr<Base>` does not.                                                                                                      |
| Static polymorphism           | Templates, CRTP, variant: no vptr, inlinable, closed set, code bloat. Type erasure: open set, value semantics, one virtual call inside.                                                                            |
| Choosing                      | Open set + stable boundary → virtual. Closed set + speed → templates, variant, type buckets. Measure with `perf`.                                                                                                  |
| Java contrast                 | Overloads static, overrides dynamic, all instance methods virtual, JIT inline caches and CHA, generics erased, no destructors (use `AutoCloseable`).                                                               |
