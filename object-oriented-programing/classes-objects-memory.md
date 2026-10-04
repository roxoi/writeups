---
title: "Module 1: Classes, Objects & Memory"
description: "Learn what an object really is - a block of memory with a layout, lifetime and address - and how it is created, copied, moved and destroyed in C++17."
author: ["name": "Rajendra Pancholi", "email": "rpancholi522@gmail.com"]
thumbnail: "/images/oop-classes-objects-memory.png"
tags: [OOP, C++, Memory-Management, RAII, Move-Semantics]
keywords: ["Classes and objects in C++", "Stack vs heap allocation", "Copy constructor shallow vs deep copy", "Rule of 0/3/5 and move semantics", "RAII and destructors"]
---

# OOP Mastery for MAANG Interviews: Module 1 — Classes, Objects & Memory

![Classes, Objects & Memory](/images/oop-classes-objects-memory.png)

### How this module is built

Each topic starts with a plain-language idea, then goes step by step deeper into how the machine really works, then shows code, a real-world picture, interview traps, and tricky questions with full answers. You never have to jump levels. Just keep reading downward and the topic gets deeper on its own.

### Words you will meet (read once, come back any time)

| Word                        | Simple meaning                                                                                                  |
| --------------------------- | --------------------------------------------------------------------------------------------------------------- |
| **Memory address**          | The "house number" of a byte in memory.                                                                         |
| **Pointer**                 | A variable that stores a memory address.                                                                        |
| **Compile time / run time** | Compile time = while the compiler builds your program. Run time = while the program is running.                 |
| **Undefined behavior (UB)** | Code the language gives no promise for. It may crash, print garbage, or appear to work today and fail tomorrow. |
| **Stack / heap**            | The two main places where a program keeps data. Topic 1.2 explains both.                                        |
| **ABI**                     | The binary-level agreement (sizes, layouts, call rules) that lets separately compiled code work together.       |
| **Cache**                   | A small, very fast memory inside the CPU. Data that is close together loads faster.                             |
| **RAII**                    | "The object that owns a resource also cleans it up." Topic 1.4.                                                 |

## 1.1: Classes vs. Objects

### The idea in plain words

A **class** is a plan, like the drawing of a house. An **object** is a real house built from that plan. One plan can build many houses. Each house has its own furniture, but they all follow the same drawing.

```cpp
#include <string>

class Dog {
public:
    std::string name;       // data: every Dog gets its own copy
    int age;
    void birthday() { ++age; }   // behavior: the code exists only once
};

// Dog a;  Dog b;   <- two objects, each with its own name and age
```

Notice the split:

- **Data** (`name`, `age`) lives _inside each object_.
- **Code** (`birthday()`) lives _once_, outside the objects, and every Dog shares it.

That one idea explains most of the memory facts below.

### What the compiler really does

A class itself takes **no data memory**. It is only a description. Here is what actually exists in memory:

- The member functions are compiled once into the program's `.text` section (the part of the program file that holds machine code). All objects share them.
- Only **non-static data members** live inside each object. If the class has virtual functions, there is also a hidden pointer called the **`vptr`** (see below).
- **Static members** are not inside the object. They live in the `.data` or `.bss` sections (global storage: `.data` for values that start non-zero, `.bss` for zero-start values).

#### How members are laid out (padding and alignment)

The compiler puts members in memory **in the order you declare them**. But CPUs like to read a value from an address that is a multiple of its size. This rule is called **alignment**. An `int` (4 bytes) wants an address divisible by 4. A `double` (8 bytes) wants an address divisible by 8. To satisfy this, the compiler adds empty gap bytes called **padding**.

The full rules:

- Each member is placed at the next address that fits its alignment.
- The object's own alignment equals its largest member's alignment.
- `sizeof` (the object's size) is rounded up to a multiple of that alignment. The extra bytes at the end are called **tail padding**.
- C++ **never reorders** your members. You control the order.

#### Two small surprises

- An **empty class** has `sizeof == 1`, not 0. Two different objects must have two different addresses, so even an empty one needs one byte. When an empty class is used as a _base class_, the compiler can remove that byte. This is called **Empty Base Optimization (EBO)**.
- A class with **virtual functions** (functions that choose their version at run time, covered fully in Module 4) gets a hidden pointer called the **vptr** (8 bytes on a 64-bit system). It points to a per-class table called the **vtable**, stored in read-only memory. The constructor sets the vptr.

#### Java comparison

Every Java object has a header (a "mark word" plus a "klass pointer"), usually 12–16 bytes with compressed pointers. The JVM may **reorder fields** to reduce padding. C++ never reorders.

### Code: seeing padding with your own eyes

```cpp
#include <cstddef>   // offsetof
#include <cstdint>
#include <iostream>

static_assert(sizeof(void*) == 8, "Layout numbers below assume LP64");

// Members laid out in declaration order; compiler inserts padding.
struct Wasteful {
    char    a;   // offset 0
                 // 7 bytes padding so 'b' is 8-aligned
    double  b;   // offset 8
    char    c;   // offset 16
                 // 3 bytes padding so 'd' is 4-aligned
    int32_t d;   // offset 20
};               // sizeof == 24 (alignof == 8)

// Same data, sorted by descending alignment => minimal padding.
struct Compact {
    double  b;   // 0
    int32_t d;   // 8
    char    a;   // 12
    char    c;   // 13
                 // 2 bytes tail padding to round up to multiple of 8
};               // sizeof == 16

struct Empty {};                                  // sizeof == 1
struct Poly { virtual ~Poly() = default; int x; };// vptr(8) + int(4) + pad(4) == 16
struct WithStatic { static int counter; int v; }; // sizeof == 4 (static not in object)

static_assert(sizeof(Wasteful) == 24 && alignof(Wasteful) == 8, "");
static_assert(sizeof(Compact)  == 16, "");
static_assert(sizeof(Empty)    == 1,  "");
static_assert(sizeof(Poly)     == 16, "");
static_assert(sizeof(WithStatic) == 4, "");

int main() {
    std::cout << "offset of d in Wasteful: " << offsetof(Wasteful, d) << '\n'; // 20
}
```

(`static_assert` checks a fact at compile time. If the number is wrong, the program does not even compile. `LP64` means a common 64-bit setup where pointers and `long` are 8 bytes.)

The lesson: **sort members from largest alignment to smallest** and `Wasteful` (24 bytes) becomes `Compact` (16 bytes) with the same data.

### Real-world picture

A class is a **container image** and an object is a **running container**. The image (code and metadata) is stored once and shared. Each running container only pays for its own writable state. That is why a million `Order` objects cost `1,000,000 × sizeof(Order)` and not a million copies of the methods.

Padding is like **protobuf or Parquet schema design**: the order of fields changes the memory footprint. At 500 million rows, saving 8 bytes per row saves 4 GB of RAM and causes fewer cache misses.

### Interview traps

- "What is `sizeof` an empty class?" The answer is **1, not 0**. "What if it is a base class?" EBO makes it cost 0 extra bytes.
- Reordering members changes `sizeof`, and the _source order_ is part of the ABI. Reordering members in a public header breaks binary compatibility.
- Adding a single `virtual` function adds 8 bytes to **every** instance. That matters for billion-object workloads.
- **Object slicing:** copying a derived object into a base-typed _value_ throws away the derived members and the derived vptr.
- **False sharing:** two hot atomic variables in one object, used by two threads, fight over the same cache line (a 64-byte block of CPU cache). The fix is `alignas(64)` to push them onto separate lines.
- Java: "How much memory does `new Object()` take?" (about 16 bytes). "`Integer` vs `int` in a `List`?" (the `Integer` is a separate object, so you get pointer-chasing plus object headers).

### Tricky questions and answers

#### Q1 \[SDE-1/2\]: What is `sizeof(Msg)` on a 64-bit platform, and how do you shrink it without changing meaning?

```cpp
struct Msg {
    bool   urgent;     // 1
    double timestamp;  // 8
    bool   encrypted;  // 1
    int    length;     // 4
    short  version;    // 2
};
```

**Answer:**

- `urgent` is at offset 0 (then 7 bytes of padding). `timestamp` is at 8. `encrypted` is at 16 (then 3 bytes of padding). `length` is at 20. `version` is at 24 (then 6 bytes of tail padding). **sizeof = 32.**
- Sort by descending alignment: `double(8) int(4) short(2) bool bool` gives 8+4+2+1+1 = 16, so **sizeof = 16**. That is a 50% reduction.
- Bitfields (`bool urgent:1`) shrink it even more, but they make access slower and non-atomic. Mention this trade-off.

```cpp
struct MsgCompact { double timestamp; int length; short version; bool urgent; bool encrypted; };
static_assert(sizeof(MsgCompact) == 16, "");
```

#### Q2 \[SDE-2\]: What does this print, and why is it a bug?

```cpp
#include <iostream>
#include <array>
#include <string>
struct Base {
    int id = 1;
    virtual std::string name() const { return "Base"; }
    virtual ~Base() = default;
};
struct Derived : Base {
    std::array<char, 64> payload{};
    std::string name() const override { return "Derived"; }
};
void byValue(Base b)        { std::cout << b.name() << '\n'; }
void byRef(const Base& b)   { std::cout << b.name() << '\n'; }
int main() { Derived d; byValue(d); byRef(d); }
```

**Answer:** It prints `Base` and then `Derived`.

`byValue` takes a `Base` _by value_, so it runs `Base`'s copy constructor. That copies **only the `Base` part** of `d`. The 64-byte payload is cut off (this is called **slicing**), and the new object's vptr points to **Base's vtable**. So the virtual call can never reach `Derived`.

Fix: pass polymorphic types by `const&`, by pointer, or by smart pointer. Even better, forbid slicing at the root of the hierarchy:

```cpp
struct Base {
    Base() = default;
    Base(const Base&) = delete;             // forbid accidental slicing
    Base& operator=(const Base&) = delete;
    virtual std::unique_ptr<Base> clone() const = 0;  // explicit polymorphic copy
    virtual ~Base() = default;
};
```

## 1.2: Stack vs. Heap Allocation

### The idea in plain words

A program keeps its data in a few places. The two most important are:

- **The stack** is like a pile of plates, or a scratchpad. When a function starts, it puts its local variables on top. When the function ends, everything it put there is removed automatically. It is very fast and you never clean it up yourself.
- **The heap** is like a big warehouse. You ask for space (`new` or `malloc`), you get an address, and you must give the space back (`delete` or `free`) when you are done. It is flexible and long-lived, but slower, and it is your job to release it.

Formally: _automatic (stack) storage_ is tied to a block of code and freed automatically. _Dynamic (heap, also called free-store) storage_ is requested explicitly and lives until released.

### How each one works inside

#### The stack

- Each thread has **one contiguous region** of stack (about 8 MB on Linux, 1 MB on Windows by default).
- Allocating is **one CPU instruction**: `sub rsp, N` (move the stack pointer down). Freeing is `add rsp, N`.
- A function's **frame** holds the return address, the saved frame pointer, local variables, spilled registers, and padding.
- It is always "hot" in the L1 cache (the fastest CPU cache).

#### The heap

- An **allocator** (glibc's ptmalloc, jemalloc, tcmalloc) manages lists of free chunks ("bins") and large regions ("arenas").
- Each chunk carries bookkeeping data (about 8–16 bytes).
- To grow, the allocator asks the operating system with `brk` or `mmap` calls.
- Costs: **lock contention** between threads, **fragmentation** (free memory split into unusable pieces), and **unpredictable latency**.

#### Global storage

- Globals and `static` locals live in the **`.data`** section (initialized values) and the **`.bss`** section (zero-initialized). Both are set up when the program loads.

#### Java comparison

- All objects are on the heap. References live in stack frames or fields.
- HotSpot's **escape analysis** can notice an object that never "escapes" a method and replace it with plain registers or stack values.
- **TLABs** (thread-local allocation buffers) make Java allocation a simple pointer bump, which is very fast.

#### Why layout matters: cache locality

`std::vector<T>` stores its elements **side by side**, so the CPU's prefetcher can stream them in. `vector<T*>` or `list<T>` forces **pointer chasing**: every element may be somewhere else, and each jump can be a cache miss (about 100 ns). Scans can be roughly **10–100× slower**.

### Code: where does each thing live?

```cpp
#include <iostream>
#include <memory>
#include <string>
#include <vector>

int g_initialized = 42;   // .data segment
int g_zeroed;             // .bss segment

struct Particle { double x, y, z, mass; };  // 32 bytes: 2 per 64-byte cache line

void where_does_it_live() {
    int local = 1;                      // STACK
    static int calls = 0;               // .data/.bss (initialized once, not per call)
    ++calls;
    auto* heap_int = new int(7);        // pointer 'heap_int' on STACK, pointee on HEAP
    std::vector<int> v(4);              // vector control block (3 pointers, 24 bytes) on STACK,
                                        // its 16-byte element buffer on HEAP
    std::string small = "hi";           // SSO: chars stored INSIDE the string object (stack)
    std::string big(100, 'x');          // too large for SSO: buffer on HEAP

    std::cout << "stack local : " << &local    << '\n'
              << "static      : " << &calls    << '\n'
              << "global .data: " << &g_initialized << '\n'
              << "heap int    : " << heap_int  << '\n'
              << "vector buf  : " << v.data()  << '\n';
    delete heap_int;                    // every new needs exactly one matching delete
}

// Contiguous: hardware prefetcher friendly.
double sum_contiguous(const std::vector<Particle>& v) {
    double s = 0;
    for (const auto& p : v) s += p.mass;
    return s;
}
// Pointer chasing: each element is a separate heap allocation. After churn/fragmentation
// these are scattered; every dereference can be a cache miss.
double sum_pointer_chasing(const std::vector<std::unique_ptr<Particle>>& v) {
    double s = 0;
    for (const auto& p : v) s += p->mass;
    return s;
}
```

(**SSO** means _small string optimization_: short strings keep their characters inside the string object itself, so no heap is used.)

### Real-world picture

The stack is a **request-scoped scratch space**: free to take, and thrown away in one step when the request ends. The heap is a **shared database or cache cluster**: flexible and long-lived, but every access needs coordination and has variable latency.

High-frequency trading systems and game engines deliberately avoid heap allocation on the hot path. They use arenas, pools and pre-reserved vectors for exactly this reason.

### Interview traps

- Returning a pointer or reference to a stack local (**dangling**, UB). The compiler may warn, but through aliasing it often cannot.
- "Is `std::vector<int> v;` on the heap?" Neither fully. The small control block is wherever `v` lives (here, the stack). The element buffer is on the heap.
- **Stack overflow:** `int buf[10'000'000];` inside a function, or unbounded recursion. Fix: use `std::vector`, or an iterative algorithm.
- `new` in a hot loop becomes a latency bottleneck and a lock-contention point. Fix: reuse buffers, call `reserve()`, use object pools or `std::pmr`.
- "Does `new` always mean heap?" No. Placement `new`, custom `operator new`, and arenas exist. Also, `make_unique` is still a heap allocation.
- Linked list vs. vector for iteration-heavy work: Big-O says list inserts are O(1), but cache behavior makes the vector win up to surprisingly large sizes.
- `malloc(0)`, mismatching `delete` with `delete[]`, or calling `free` on memory from `new` are all UB.

### Tricky questions and answers

#### Q1 \[SDE-1\]: Find all the bugs.

```cpp
int& make_counter() { int c = 0; return c; }

char* greeting() {
    char buf[32] = "hello";
    return buf;
}

void leak() {
    int* p = new int[100];
    if (rand() % 2) return;     // early return
    delete p;
}
```

**Answer:**

1. `make_counter` returns a reference to a local variable. The stack frame is removed when the function ends, so the reference **dangles** (UB).
2. `greeting` returns the address of a stack array that no longer exists (same bug).
3. `leak`: the early return leaks 400 bytes, and `delete p` should be `delete[] p` (UB). All of this disappears with RAII.

```cpp
#include <string>
#include <vector>
int  make_counter()   { return 0; }                 // return by value
std::string greeting() { return "hello"; }          // std::string owns its storage
void no_leak() {
    std::vector<int> p(100);                        // freed on every exit path
    if (rand() % 2) return;
}
```

#### Q2 \[SDE-2/3\]: Why can looping over a `std::vector<Particle>` be 10–50× faster than over a `std::vector<std::unique_ptr<Particle>>` when both are O(n)?

**Answer:** Both are O(n), but the constants are very different:

- **Contiguous layout gives spatial locality.** One 64-byte cache line holds 2 Particles. The hardware prefetcher sees the straight-line pattern and loads ahead, so most reads hit L1.
- **The pointer vector** first loads a pointer and then **follows it to a separate allocation**. Each allocation also carries about 16 bytes of allocator header, which wastes cache space. After allocator churn, the objects are scattered, so each access can miss L1 and L2 and go all the way to DRAM (about 100 ns instead of about 1 ns). These loads also depend on each other, so the CPU cannot overlap them.
- Heap allocation also costs N allocator calls when building and N frees when destroying.

**What to do:** prefer value semantics and contiguous storage. If you need polymorphism, store `std::variant<A,B,C>` in a vector, or group objects by type (structure-of-arrays). **Measure before changing**, and mention performance counters such as `perf stat -e cache-misses`.

## 1.3: Constructors Deep-Dive

### The idea in plain words

A **constructor** is a special function that runs when an object is created. Its job is to put the object into a good starting state, and to make sure the object's **class invariants** (rules that must always be true, like "size is never negative") hold from the very first moment.

The kinds of constructor:

| Kind          | Looks like      | Purpose                                             |
| ------------- | --------------- | --------------------------------------------------- |
| Default       | `T()`           | Create with no arguments                            |
| Parameterized | `T(int x)`      | Create from values                                  |
| Copy          | `T(const T&)`   | Create a copy of another object                     |
| Move          | `T(T&&)`        | Take over another object's resources (Topic 1.6)    |
| Delegating    | `T() : T(0) {}` | One constructor calls another                       |
| Converting    | single-argument | Allows implicit conversion. **Mark it `explicit`.** |

### How constructors really work

- A constructor is just a function with a hidden `this` pointer aimed at **already-allocated raw storage**. `new T(args)` does two steps: `operator new(sizeof(T))` to get memory, then the constructor call. If the constructor throws, the matching `operator delete` frees the memory.
- **Initialization order is fixed:**
  1. virtual bases (Module 3),
  2. direct base classes, in declaration order,
  3. **non-static members, in the order they are _declared in the class_, not the order written in the initializer list**,
  4. the constructor body.
- An **initializer list** (`: size_(n), data_(...)`) _initializes_ members directly. Assigning inside the body _re-assigns_ after the members were already default-constructed. Initializer lists are **required** for `const` members, reference members, and members with no default constructor.
- If you declare any constructor yourself, the compiler **removes** the automatic default constructor.
- The compiler's copy constructor does a **memberwise copy**. For a raw pointer, that copies the _address_, not the data. This is a **shallow copy**. Two objects now own one resource, and a **double free** follows. A **deep copy** allocates new storage and copies the pointee.
- **Copy elision:** since C++17, `T t = T(...)` and `return T(...)` are _guaranteed_ to build the object directly in its final place, with no copy. **Named Return Value Optimization (NRVO)** (returning a named local) is allowed but not guaranteed.
- If a constructor throws halfway, the already-built members and bases are destroyed in reverse order, but the object's own destructor does **not** run. Raw owning pointers acquired in the body therefore **leak**.
- **Java comparison:** there are no value-copy semantics. Assignment copies the _reference_. `clone()` and copy-constructor patterns are manual. A Java `final` field is the equivalent of a C++ `const` member.

### Code: shallow copy vs. a correct deep-copy class

```cpp
#include <algorithm>
#include <cstddef>
#include <iostream>
#include <utility>

// BAD: implicit copy duplicates the pointer, so the destructor runs twice on one block.
class ShallowBuffer {
    int* data_;
public:
    explicit ShallowBuffer(std::size_t n) : data_(new int[n]()) {}
    ~ShallowBuffer() { delete[] data_; }
    // implicit copy ctor: data_ copied by value => DOUBLE FREE on copy
};

class Buffer {
    std::size_t size_;   // declared FIRST => initialized FIRST
    int*        data_;   // declared SECOND

public:
    // Parameterized + converting ctor guard: 'explicit' blocks `Buffer b = 5;`.
    explicit Buffer(std::size_t n)
        : size_(n),                       // initializer list: direct initialization
          data_(new int[n]()) {}          // () => value-initialize (zero-fill)

    Buffer() : Buffer(0) {}               // delegating ctor: single source of truth

    // Deep-copy constructor: takes const& (by value would recurse infinitely).
    Buffer(const Buffer& other)
        : size_(other.size_), data_(new int[other.size_]) {
        std::copy(other.data_, other.data_ + size_, data_);
    }

    // Copy assignment via copy-and-swap: strong exception safety + self-assignment safe.
    Buffer& operator=(Buffer other) noexcept {   // 'other' is already a deep copy
        swap(other);
        return *this;                            // 'other' (old state) is destroyed here
    }

    void swap(Buffer& o) noexcept {
        std::swap(size_, o.size_);
        std::swap(data_, o.data_);
    }

    ~Buffer() { delete[] data_; }
};
```

How the **copy-and-swap** trick works: the assignment operator takes its argument _by value_, so the caller makes a deep copy for you. You then swap your guts with the copy. When the function ends, the copy (which now holds your old data) is destroyed and cleans it up. The result is exception-safe and handles `a = a` correctly.

### Real-world picture

Copying a service object that holds a **database connection handle**: a shallow copy gives two services sharing one connection. When one closes it, the other fails in a confusing way. A deep copy opens an independent connection. It is the difference between copying a symlink (copies the pointer) and `cp -r` (copies the data).

Initializer-list order bugs are the config version of a service reading an environment variable _before_ the bootstrap script has set it.

### Interview traps

- **Initializer-list order trap:** members start in _declaration_ order. Using one member to set up another compiles fine and reads garbage if the order is wrong. `-Wreorder` catches the mismatch.
- **Most vexing parse:** `Widget w();` does not create an object. It declares a function! Write `Widget w;` or `Widget w{};`.
- Missing `explicit` allows silent conversions: with `void f(Buffer); f(1000000);` a megabyte is allocated without you noticing.
- A copy constructor that takes its argument **by value** is infinite recursion (compile error).
- **Self-assignment** (`a = a`) with a "delete first, then copy" implementation destroys the data before copying it.
- An exception inside a constructor means the destructor does not run, so raw `new` members leak.
- `std::vector<T>` growth uses the move constructor **only if it is `noexcept`**. Otherwise it copies every element.
- A **virtual function called inside a constructor** goes to the class being built right now, not the most-derived class.
- Java: leaking `this` from a constructor (registering a listener, starting a thread) publishes a half-built object.

### Tricky questions and answers

#### Q1 \[SDE-2\]: What is the bug? What are the fixes?

```cpp
class Window {
    std::size_t area_;
    std::size_t w_;
    std::size_t h_;
public:
    Window(std::size_t w, std::size_t h) : h_(h), w_(w), area_(w_ * h_) {}
};
```

**Answer:** Members start in **declaration order**: `area_`, then `w_`, then `h_`, no matter how the list is written. So `area_(w_ * h_)` reads two members that are **not yet initialized** (indeterminate value, UB). Fixes:

```cpp
// Fix A: use the constructor parameters, which are always initialized.
Window(std::size_t w, std::size_t h) : area_(w * h), w_(w), h_(h) {}

// Fix B: declare dependencies first (and keep the list in the same order).
//   std::size_t w_; std::size_t h_; std::size_t area_;
// Fix C (best): don't store derived data. Compute it.
std::size_t area() const { return w_ * h_; }
```

#### Q2 \[SDE-2/3\]: How many copies or moves does each function do in C++17?

```cpp
struct Tracer {
    Tracer()                  { std::puts("ctor"); }
    Tracer(const Tracer&)     { std::puts("copy"); }
    Tracer(Tracer&&) noexcept { std::puts("move"); }
    ~Tracer()                 { std::puts("dtor"); }
};
Tracer make1()        { return Tracer(); }
Tracer make2()        { Tracer t; return t; }
Tracer make3(bool f)  { Tracer a, b; return f ? a : b; }
int main() { Tracer x = make1(); Tracer y = make2(); Tracer z = make3(true); }
```

**Answer:**

- `make1`: returns a temporary (a _prvalue_). C++17 **guarantees elision**: one `ctor`, zero copies or moves, built directly in `x`.
- `make2`: returns a named local. **NRVO** is allowed, not required. GCC, Clang and MSVC will normally elide it (one `ctor`). If the compiler does not elide, it falls back to the **move** constructor, because a returned local is first treated as an rvalue.
- `make3`: two possible locals, and the result is a conditional expression (not a plain variable name), so NRVO is impossible and the implicit move does not apply. The compiler must **copy**. Expect `ctor ctor copy dtor dtor`, with the copied object destroyed later in `main`.
- Interview takeaway: **do not write `return std::move(local);`**. It blocks NRVO and triggers a "pessimizing move" warning.

## 1.4: Destructors & Garbage Collection

### The idea in plain words

A **destructor** is a function that runs automatically when an object's life ends. It gives back whatever the object was holding: memory, files, locks, network connections.

**Garbage collection (GC)** is a different approach used by Java, Python and Go. A background system finds objects nobody can reach any more and reclaims their **memory** at some later time.

Two big differences to remember:

- C++ destructors run **at a known moment** (deterministic).
- GC runs **when the system decides** (non-deterministic), and it only handles _memory_, not files or sockets.

### How C++ destruction works

- Destructors run at the end of a scope (even when an **exception** is flying out of it, called **stack unwinding**), at `delete`, or at program shutdown for static objects (reverse order of construction).
- **Destruction order:** destructor body first, then members in **reverse declaration order**, then base classes in reverse construction order.
- **`virtual` destructor:** calling `delete base_ptr` on a derived object when the base destructor is _not_ virtual is **UB**. Usually only `~Base` runs and the derived part leaks. With `virtual`, the call is dispatched through the vtable to the right destructor. Rule: any class meant to be deleted through a base pointer needs a public virtual destructor (or a protected non-virtual one).
- Destructors are implicitly `noexcept`. Throwing out of one during unwinding calls `std::terminate` (the program ends).
- **RAII** (Resource Acquisition Is Initialization): tie a resource to an object's life. Acquire in the constructor, release in the destructor. It works for memory, file descriptors, locks, sockets and transactions.
- `std::shared_ptr` keeps a heap **control block** with strong and weak counts and a deleter. Reference counting **cannot reclaim cycles** (A holds B, B holds A).

### How garbage collection works (Java)

- A tracing GC starts from **GC roots** (thread stacks, static fields, JNI handles), marks everything reachable, and reclaims the rest. Cycles are collected naturally.
- **Generational hypothesis:** most objects die young. The young generation (Eden plus survivor spaces) uses a fast _copying_ collector. Survivors are promoted to the old generation, collected by G1, ZGC or Shenandoah (concurrent, region-based, with ZGC pauses mostly under a millisecond).
- Cleanup is **non-deterministic**, so GC manages _memory only_. `finalize()` is deprecated for removal (JEP 421). Use `try-with-resources` (`AutoCloseable`) for predictable release, and `Cleaner` only as a safety net. Python uses `with`, Go uses `defer`.
- GC "leaks" are really _reachability leaks_: static caches, listeners nobody removed, `ThreadLocal`s in thread pools, and inner classes that hold their outer object.

### Code: an RAII file wrapper and a polymorphic base

```cpp
#include <cstdio>
#include <memory>
#include <stdexcept>
#include <string>
#include <string_view>
#include <utility>

// RAII wrapper: exclusive ownership of a FILE* (like unique_ptr, but explicit for teaching).
class File {
    std::FILE* fp_{nullptr};

    void close() noexcept {                     // never throws: destructors must not
        if (fp_) { std::fclose(fp_); fp_ = nullptr; }
    }
public:
    File(const char* path, const char* mode) : fp_(std::fopen(path, mode)) {
        // If we throw here, ~File does NOT run, but nothing has been acquired yet. Safe.
        if (!fp_) throw std::runtime_error(std::string("open failed: ") + path);
    }
    File(const File&)            = delete;      // exclusive ownership => no copies
    File& operator=(const File&) = delete;
    File(File&& o) noexcept : fp_(std::exchange(o.fp_, nullptr)) {}   // transfer ownership
    File& operator=(File&& o) noexcept {
        if (this != &o) { close(); fp_ = std::exchange(o.fp_, nullptr); }
        return *this;
    }
    ~File() { close(); }                        // runs on normal exit AND during unwinding

    void write(std::string_view s) {
        if (std::fwrite(s.data(), 1, s.size(), fp_) != s.size())
            throw std::runtime_error("write failed");   // file still closed by ~File
    }
};

void process() {
    File f("out.log", "w");
    f.write("start\n");
    throw std::runtime_error("boom");           // stack unwinding calls ~File => fd released
}

// Polymorphic base: virtual destructor is mandatory for delete-through-base.
struct Shape {
    virtual ~Shape() = default;
    virtual double area() const = 0;
};
struct Circle : Shape {
    std::unique_ptr<double[]> samples = std::make_unique<double[]>(1000);  // Rule of Zero
    double area() const override { return 3.14159; }
};
// std::unique_ptr<Shape> s = std::make_unique<Circle>();  // correct destruction
```

Notice `process()`: it throws an exception, yet the file still gets closed. That is the power of RAII. You never write a `close()` call on the error path.

### Real-world picture

RAII is a **connection-pool lease** that is returned automatically when the request handler's scope ends, even if the handler crashes. GC is a **janitor who sweeps the building at night**: great for reclaiming memory, useless when you need the meeting room unlocked _right now_.

A real incident pattern: a Java service relied on `finalize()` to close sockets. It worked in tests but hit "Too many open files" in production, because the GC runs when the _heap_ is under pressure, not when _file descriptors_ are.

### Interview traps

- **Missing virtual destructor** on a base class, then `delete` through a base pointer (UB and a partial leak).
- **Throwing from a destructor** ends the program during unwinding. Swallow and log the error, or offer an explicit `close()` that is allowed to throw.
- **Virtual calls inside constructors and destructors** resolve to the current class's version, never a more-derived one.
- `delete` vs `delete[]`, double delete, and `delete this` (legal but fragile).
- **`shared_ptr` cycles** (parent ↔ child) leak. Break them with `weak_ptr`. Also, `shared_ptr` is not free: atomic reference-count operations, a control-block allocation, and cache-line contention.
- Prefer `make_shared` and `make_unique` (one allocation, and exception safe).
- Java: "Is `finalize` reliable?" No. "Can Java leak memory?" Yes, through lingering references. "Why use try-with-resources?"
- SDE-3 interviews also ask about GC pause behavior (stop-the-world, tuning `-Xmx`, G1 vs ZGC).

### Tricky questions and answers

#### Q1 \[SDE-2\]: What happens here? Fix it properly.

```cpp
struct Base { ~Base() { std::puts("~Base"); } };
struct Derived : Base {
    int* p = new int[1000];
    ~Derived() { delete[] p; std::puts("~Derived"); }
};
int main() { Base* b = new Derived; delete b; }
```

**Answer:** Deleting a `Derived` through a `Base*` when `Base`'s destructor is non-virtual is **undefined behavior**. In practice only `~Base` runs, so `~Derived` is skipped and the 4000-byte array **leaks**. Sanitizers may also report a size mismatch. Fix:

```cpp
struct Base { virtual ~Base() = default; };
struct Derived : Base {
    std::unique_ptr<int[]> p = std::make_unique<int[]>(1000);  // Rule of Zero: no manual dtor
};
int main() { std::unique_ptr<Base> b = std::make_unique<Derived>(); }  // no raw delete
```

#### Q2 \[SDE-2/3\]: Why does `~Node` never print, and how is this different in Java?

```cpp
struct Node {
    std::shared_ptr<Node> next, prev;
    ~Node() { std::puts("~Node"); }
};
int main() {
    auto a = std::make_shared<Node>();
    auto b = std::make_shared<Node>();
    a->next = b;
    b->prev = a;
}   // end of main
```

**Answer:** Node `a` is owned by the local variable `a` and by `b->prev` (count 2). Node `b` is owned by the local `b` and by `a->next` (count 2). At the end of `main`, each local drops one reference, leaving both counts at 1, held by each other. Neither reaches zero, so **both leak**. Reference counting has no idea about "reachable from a root".

Fix: make the back-pointer non-owning.

```cpp
struct Node {
    std::shared_ptr<Node> next;
    std::weak_ptr<Node>   prev;   // does not contribute to the strong count
    ~Node() { std::puts("~Node"); }
};
```

In Java, the same graph is collected, because neither node is reachable from a GC root once the locals die. The trade-off: Java accepts non-determinism and GC pauses. C++ gives deterministic cost but you must watch for cycles yourself.

## 1.5: The `this` Pointer / Reference

### The idea in plain words

When you write `dog.birthday()`, how does the single shared `birthday()` function know _which_ dog to change? The answer: every non-static member function secretly receives one extra argument, the address of the object it was called on. Inside the function that address is called **`this`**.

So `this` means "the object I am working on right now."

### What the compiler really does

- Methods are **not stored per object**. There is **one** compiled function, and the object's address is passed as a **hidden first parameter**. `obj.f(x)` compiles roughly to `T::f(&obj, x)`. On x86-64 System V, `this` arrives in the `rdi` register, and the other arguments follow in `rsi`, `rdx`, and so on.
- Its type is `T* const` in normal methods and `const T* const` in `const` methods. That is why a `const` method cannot change members, and why `const` and non-`const` overloads count as different functions.
- `this` is a **prvalue** (a temporary value), so you cannot take its address. **Static** member functions have no `this`.
- **Virtual dispatch:** the call loads the vptr from `*this` (offset 0 in typical ABIs), indexes the vtable, and calls with `this`. With multiple inheritance, small **thunks** adjust `this` to point at the right sub-part (Module 3 and 4).
- A member name like `count_` is shorthand for `this->count_`, which compiles to `[this + offset]`.
- Calling a method through a **null pointer** is UB, even if the method never touches a member. The optimizer is allowed to assume `this != nullptr`.
- Lambdas capture `this` **by pointer**. `[*this]` (C++17) copies the whole object instead.
- C++23 adds an **explicit object parameter** ("deducing this"): `void f(this Self&& self)`.
- **Java comparison:** `this` is local variable slot 0 in every instance method's bytecode (`aload_0`). It is a managed reference and cannot be null inside an instance method.

### Code: chaining and the hidden parameter made visible

```cpp
#include <iostream>
#include <string>

class Counter {
    int count_ = 0;
public:
    // Returns *this by REFERENCE to enable chaining without copying.
    Counter& add(int n) {
        this->count_ += n;     // explicit this-> (identical to 'count_ += n')
        return *this;          // dereference this => the object itself (lvalue ref)
    }
    int get() const {          // 'this' has type const Counter* const here
        // this->count_ = 0;   // compile error: modifying through pointer-to-const
        return count_;
    }
    // Self-assignment guard pattern: compare addresses via this.
    Counter& operator=(const Counter& o) {
        if (this != &o) count_ = o.count_;
        return *this;
    }
};

// What the compiler conceptually generates (name mangling and ABI aside):
struct Counter_c { int count_; };
Counter_c& Counter_add(Counter_c* const self, int n) {   // 'this' made explicit
    self->count_ += n;
    return *self;
}
int Counter_get(const Counter_c* const self) { return self->count_; }

int main() {
    Counter c;
    c.add(1).add(2).add(3);      // all three calls receive the SAME address
    std::cout << c.get() << '\n'; // 6
}
```

### Real-world picture

A stateless microservice handler is deployed **once**, but every request carries a **tenant or session context** that tells it whose data to touch. Methods are the handler (one copy). `this` is the context passed in with each call.

A dangling `this` in a callback is like the handler holding on to a session ID after the session store has already thrown that session away.

### Interview traps

- **Chaining that returns by value** (`Builder add()` instead of `Builder& add()`): it changes a temporary copy, so the original is unchanged.
- **Capturing `this` in async callbacks, timers or threads:** the object may be destroyed before the callback fires (use-after-free).
- **Letting `this` escape during construction** (Java listener registration, C++ thread start inside a constructor): other code sees a half-built object.
- `delete this` is legal only if the object was created with `new`, nothing touches members afterwards, and nobody else holds pointers to it. It is used in reference counting and COM.
- "Can `this` be null?" Technically UB if it is, and compilers can optimize away `if (this == nullptr)`.
- **Const-overload pairs** (`T& at()` and `const T& at() const`), and why `mutable` exists for caches.
- **Name shadowing:** `void setX(int x) { x = x; }` assigns the parameter to itself. Use `this->x = x`.
- "Is a member function's cost per-object?" No. Non-virtual methods cost zero extra bytes per instance.

### Tricky questions and answers

#### Q1 \[SDE-1/2\]: Why is `b.str()` not `"ab"`?

```cpp
class Builder {
    std::string s_;
public:
    Builder add(const std::string& x) { s_ += x; return *this; }
    std::string str() const { return s_; }
};
int main() { Builder b; b.add("a").add("b"); std::cout << b.str(); }
```

**Answer:** It prints `a`. `add` returns `Builder` **by value**, so `return *this` runs the copy constructor. The first `add("a")` changes `b` and returns a temporary copy. Then `.add("b")` changes that **temporary**, which is thrown away. Return a reference instead:

```cpp
Builder& add(const std::string& x) { s_ += x; return *this; }
```

For one-shot builders, add ref-qualified overloads to avoid copies on temporaries: `Builder&& add(...) &&`.

#### Q2 \[SDE-3\]: What is wrong with this async code, and what are the safe options?

```cpp
class Session {
    std::string id_;
public:
    explicit Session(std::string id) : id_(std::move(id)) {}
    std::function<void()> makeLogger() {
        return [this] { std::cout << id_ << '\n'; };
    }
};
int main() {
    std::function<void()> logger;
    { Session s{"abc"}; logger = s.makeLogger(); }   // s destroyed here
    logger();                                        // ???
}
```

**Answer:** The lambda stored a raw `this` pointer. After `s` is destroyed, `logger()` follows a dangling pointer. That is **use-after-free** (UB). It may print garbage, crash, or even seem to work. ASan reports `stack-use-after-scope`. Safe options:

```cpp
// 1. Capture the needed data by value (best when it's cheap).
return [id = id_] { std::cout << id << '\n'; };

// 2. Capture a copy of the entire object (C++17).
return [*this] { std::cout << id_ << '\n'; };

// 3. Shared ownership: weak_ptr guard for async lifetimes.
class Session : public std::enable_shared_from_this<Session> {
    // ...
    std::function<void()> makeLogger() {
        return [weak = weak_from_this()] {
            if (auto self = weak.lock()) std::cout << self->id_ << '\n';  // runs only if alive
        };
    }
};
// Requires: auto s = std::make_shared<Session>("abc");
```

## 1.6 Rule of 0/3/5 and Move Semantics

### The idea in plain words

By now you have seen three questions every resource-owning class must answer:

1. What happens when I **copy** it?
2. What happens when I **move** it?
3. What happens when I **destroy** it?

If a class manages a resource, you must answer all three **consistently**. The "rules" are just a checklist for that:

- **Rule of Three:** if you write a destructor, copy constructor, or copy assignment, you almost certainly need all three.
- **Rule of Five:** C++11 adds two more: the move constructor and the move assignment.
- **Rule of Zero:** the best option. Use members that manage themselves (`vector`, `string`, `unique_ptr`), and then you write **none** of the five.

**What is a move?** Copying a 500 GB database makes a second 500 GB database. _Moving_ it just hands over the keys: the new owner points at the same data, and the old owner is left empty. A move is a few pointer copies. A copy allocates and copies O(n) bytes.

### How move semantics work inside

- A **move** transfers ownership by stealing the internal pointers and leaves the source in a **valid but unspecified** state (usually null or empty). A **copy** allocates and `memcpy`s, which costs O(n).
- An **rvalue reference** (`T&&`) binds to temporaries, or to expressions wrapped in `std::move`. **`std::move` is only a cast** to `T&&`. It moves nothing by itself. The move constructor that overload resolution picks does the real work.
- **`noexcept` on move operations is critical.** When `std::vector` grows, it uses `std::move_if_noexcept`. Without `noexcept`, it **copies** every element, to keep its "strong exception guarantee" (if something throws, the old data is still intact).
- Declaring a destructor or a copy operation **suppresses implicit move generation**. A "move" then silently becomes a copy.

### Code: all five special members

```cpp
#include <algorithm>
#include <cstddef>
#include <utility>

class Buffer {
    std::size_t size_ = 0;
    int*        data_ = nullptr;
public:
    explicit Buffer(std::size_t n) : size_(n), data_(n ? new int[n]() : nullptr) {}
    ~Buffer() { delete[] data_; }                                   // (1) destructor

    Buffer(const Buffer& o) : size_(o.size_), data_(o.size_ ? new int[o.size_] : nullptr) {
        std::copy(o.data_, o.data_ + size_, data_);                 // (2) deep copy: O(n)
    }
    Buffer& operator=(const Buffer& o) {                            // (3) copy assign
        if (this != &o) { Buffer tmp(o); swap(tmp); }               // copy then swap
        return *this;
    }
    Buffer(Buffer&& o) noexcept                                     // (4) move ctor: O(1)
        : size_(std::exchange(o.size_, 0)), data_(std::exchange(o.data_, nullptr)) {}

    Buffer& operator=(Buffer&& o) noexcept {                        // (5) move assign
        if (this != &o) {
            delete[] data_;                                         // release our resource first
            size_ = std::exchange(o.size_, 0);
            data_ = std::exchange(o.data_, nullptr);                // leave source empty-but-valid
        }
        return *this;
    }
    void swap(Buffer& o) noexcept { std::swap(size_, o.size_); std::swap(data_, o.data_); }
};

// Rule of Zero equivalent: same behavior, no special members written.
// class Buffer0 { std::vector<int> data_; public: explicit Buffer0(size_t n) : data_(n) {} };
```

(`std::exchange(a, b)` sets `a` to `b` and returns the old value of `a`. It is a neat way to "take the value and leave a replacement behind.")

Look at the last two lines: the `Buffer0` version with a `vector` gives the same behavior with **no** hand-written special members. That is why Rule of Zero is the goal.

### Real-world picture

Copy is **replicating a 500 GB database**. Move is **re-pointing the DNS record** to the existing database. The old name is left empty but is still safe to use or delete. Marking the move `noexcept` is the operational promise that the DNS cutover cannot fail halfway, so the orchestrator (`vector`) is willing to rely on it.

### Interview traps

- A move constructor **not marked `noexcept`**: `vector::push_back` growth silently copies every element (a performance cliff).
- Writing `return std::move(local);` blocks copy elision.
- **Use-after-move:** using a moved-from object for anything beyond assigning to it or destroying it.
- Moving a `const` object silently falls back to copying (`const T&&` cannot bind to `T&&`).
- A **named rvalue-reference parameter is itself an lvalue**. You need `std::move` (or `std::forward`) again to pass it on.
- Defining only a destructor means implicit moves are not generated, so code that "uses move" actually copies.

### Tricky questions and answers

#### Q1 \[SDE-2\]: Why does this vector reallocation print "copy" for every element, and how do you fix it?

```cpp
struct Item {
    Item() = default;
    Item(const Item&)     { std::puts("copy"); }
    Item(Item&&)          { std::puts("move"); }   // no noexcept
};
std::vector<Item> v;
v.reserve(1);
v.emplace_back();
v.emplace_back();   // forces reallocation
```

**Answer:** When it reallocates, `std::vector` must keep the strong exception guarantee. If a move threw halfway, the old buffer would already be damaged, with no way to roll back. So it uses `std::move_if_noexcept`: if the move constructor is not `noexcept` and a copy constructor exists, it **copies**.

Fix: `Item(Item&&) noexcept { ... }`. You should also call `reserve()` when the size is known, which avoids reallocation completely.

#### Q2 \[SDE-2/3\]: Is this safe? What is the state of `a` afterwards?

```cpp
Buffer a(1000);
Buffer b = std::move(a);
a = Buffer(10);              // (A)
// vs.
Buffer c = std::move(b);
std::cout << b.size();       // (B)
```

**Answer:**

- **(A) is safe.** The move left `a` with `size_ == 0, data_ == nullptr`, which is a valid state, and assigning to a moved-from object is explicitly allowed. The move assignment runs `delete[] nullptr` (a no-op), then takes the temporary's pointer.
- **(B) compiles and, for _this_ class, prints `0`**, because `Buffer`'s move constructor resets its source. But the standard only promises a "valid but unspecified" state for library types, and relying on it is a code smell. Only destroy or reassign a moved-from object.

The senior-level point: **`std::move` itself did nothing.** The move constructor chosen by overload resolution did the work, and its author decides what the source looks like afterward.

## Module 1 Cheat Sheet

| Concept          | One-line interview answer                                                                                   |
| ---------------- | ----------------------------------------------------------------------------------------------------------- |
| Class vs. object | Class is type metadata, object is storage. Methods are shared and data is per instance.                     |
| `sizeof`         | Members in order, padded to alignment. Sort by descending alignment to shrink.                              |
| Stack vs. heap   | Stack is pointer-bump and cache-hot. Heap is allocator-managed and potentially scattered.                   |
| Init order       | Bases, then members in **declaration** order, then the body.                                                |
| Copy             | Default is shallow. Owning raw pointers require a deep copy or a deleted copy.                              |
| Destructor       | Virtual for polymorphic bases, `noexcept`, and RAII beats manual cleanup.                                   |
| GC               | Reclaims memory only, is non-deterministic, and handles cycles. Use try-with-resources for everything else. |
| `this`           | Hidden first parameter. Chaining returns `*this` by reference.                                              |
| Rule of 5/0      | Prefer Zero. If writing five, make moves `noexcept`.                                                        |

