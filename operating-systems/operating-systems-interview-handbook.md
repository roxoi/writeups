---
title: "Operating Systems Interview Handbook: From Fundamentals to Whiteboard-Ready Answers"
description: "A ground-up, interview-focused guide to Operating Systems for SDE roles — covering processes, scheduling, synchronization, deadlocks, memory management, and file systems with worked examples."
author: [{"name": "Rajendra Pancholi", "email": "rpancholi522@gmail.com"}]
thumbnail: "/images/os-interview-handbook.png"
tags: [Operating-Systems, SDE-Interview, Computer-Science, System-Design, CS-Fundamentals]
keywords: ["Operating systems interview questions", "Process vs thread interview", "CPU scheduling algorithms", "Deadlock Banker's algorithm", "Paging vs segmentation", "Page replacement LRU FIFO", "OS interview cheat sheet SDE"]
---

# Operating Systems Interview Handbook: From Fundamentals to Whiteboard-Ready Answers

![Operating Systems Interview Handbook](/images/os-interview-handbook.png)

*A ground-up, interview-focused guide to Operating Systems for SDE roles at product-based companies (Google, Microsoft, Amazon, TCS Digital, etc.)*

### How to Use This Guide

Each concept follows a 4-part format:
- **Definition** - the crisp, interview-ready answer
- **Analogy** - a real-world picture so it sticks
- **Why It Matters / How It Works** - the mechanics
- **Interview Angle** - the exact questions/scenarios you'll be asked

## MODULE 1: Introduction & Basics

### 1.1 What is an Operating System?

**Definition:** An OS is a system software that sits between the user/applications and the computer hardware. It manages hardware resources (CPU, memory, disk, I/O devices) and provides a convenient, consistent interface for programs to run.

**Analogy:** Think of the OS as the **manager of a hotel**. Guests (programs) don't talk to the electricity board, water supply, or housekeeping staff (hardware) directly. They ask the manager (OS) for a room (memory), room service (I/O), or use of the gym (CPU time), and the manager coordinates everything so guests don't collide with each other.

**Why It Matters:** Without an OS, every application would need to know how to control the disk controller, the keyboard, the network card, etc. The OS abstracts this away and also **multiplexes** shared resources safely among many programs.

**Two core jobs of an OS:**
1. **Resource Manager** - allocates CPU, memory, disk, I/O fairly and efficiently.
2. **Abstraction Provider** - gives programs a clean interface (files, processes, sockets) instead of raw hardware.

**Interview Angle:**
- Q: "What are the main functions of an OS?" → Process management, memory management, file management, I/O management, security, and providing a user interface.
- Q: "Why do we need an OS at all?" → To manage hardware efficiently, provide abstraction, enable multitasking, and enforce protection/security between programs.

### 1.2 Dual-Mode Operation: User Mode vs. Kernel Mode

**Definition:** Modern CPUs support at least two privilege levels. **Kernel mode (supervisor mode)** allows unrestricted access to hardware and all instructions. **User mode** is restricted - a program cannot directly touch hardware or execute privileged instructions.

**Analogy:** Kernel mode is like being **airport security staff** with a master key to every door. User mode is like being a **passenger** - you can walk around the terminal (your own memory space) but you can't walk into the control tower (hardware) without going through security (a system call).

**Why It Matters:** This separation is the foundation of **system protection and stability**. If user programs could directly manipulate hardware or other processes' memory, one buggy or malicious app could crash the entire system. A mode bit in the CPU tracks the current mode; when a trap/interrupt/system call happens, the mode bit flips to kernel mode, the OS handles it, then flips back.

**How it works step by step:**
1. Program runs in user mode.
2. Program needs a privileged operation (e.g., read a file) → executes a **system call**.
3. CPU traps into kernel mode, jumps to a fixed OS entry point.
4. OS performs the operation with full hardware access.
5. Control returns to user mode, mode bit resets.

**Interview Angle:**
- Q: "What happens if a user-mode program tries to execute a privileged instruction?" → The CPU generates a **trap/exception** (e.g., "illegal instruction" or "privileged instruction fault"), and the OS typically terminates the offending process.
- Q: "Why can't user programs directly access hardware?" → Protection: prevents one process from crashing the system or corrupting another process's data/hardware state.

### 1.3 System Calls

**Definition:** A system call is the programmatic interface through which a user-mode program requests a service from the kernel (e.g., file I/O, process creation, memory allocation).

**Analogy:** A system call is like **filling out a request form** to the hotel manager instead of doing the task yourself - "please open this door for me" - because you don't have the master key.

**Why It Matters:** System calls are the *only* legitimate gateway between user mode and kernel mode. Libraries like the C standard library (`printf`, `malloc` at a lower level, `fopen`) eventually funnel down into system calls like `write()`, `brk()`/`mmap()`, `open()`.

**Categories of system calls:**
| Category | Examples |
|---|---|
| Process control | `fork()`, `exec()`, `exit()`, `wait()` |
| File management | `open()`, `read()`, `write()`, `close()` |
| Device management | `ioctl()`, `read()`, `write()` |
| Information maintenance | `getpid()`, `alarm()`, `sleep()` |
| Communication | `pipe()`, `shmget()`, `send()`, `recv()` |

**How it works:** A system call typically uses a **software interrupt/trap instruction** (e.g., `int 0x80` historically, or `syscall` instruction on x86-64) that transfers control to a kernel trap handler, which looks up the requested service in a **system call table** using a syscall number.

**Interview Angle:**
- Q: "Difference between `fork()` and `exec()`?" → `fork()` creates a near-identical **copy** of the calling process (child gets a copy of the parent's address space, same code, new PID). `exec()` **replaces** the current process image with a new program (same PID, new code/data/stack). They're often used together: fork a child, then exec a new program in it (this is how shells launch commands).
- Q: "What happens internally during a system call?" → Trap instruction → CPU switches to kernel mode → OS uses syscall number to index into a table of function pointers → executes the requested kernel function → returns result → switches back to user mode.
- Q: "Is a system call the same as a function call?" → No - a function call stays in the same privilege level and address space; a system call causes a **mode switch** with extra overhead (saving registers, context, mode transition).

## MODULE 2: Process & Thread Management

### 2.1 Process vs. Program vs. Thread

**Definition:**
- **Program:** Passive entity - code sitting on disk (an executable file).
- **Process:** Active entity - a program in execution, with its own memory space (code, data, heap, stack), state, and resources.
- **Thread:** The smallest unit of CPU execution *within* a process. A process can have multiple threads that share the same memory/resources but have their own stack, registers, and program counter.

**Analogy:** A **program** is a recipe in a cookbook. A **process** is a chef actively cooking that recipe in their own kitchen (with their own ingredients/utensils = memory). **Threads** are multiple hands of the *same* chef working simultaneously in that *same* kitchen - they share the fridge and counter (memory) but each hand is doing its own task (execution).

**Why It Matters:** Threads are "lightweight processes" - creating and switching threads is much cheaper than processes because threads share memory (no need to duplicate address space).

**Key comparison table (classic interview table - memorize this):**

| Aspect | Process | Thread |
|---|---|---|
| Memory | Separate address space | Shares address space with sibling threads |
| Creation cost | Heavy (OS allocates new memory, PCB, etc.) | Lightweight |
| Communication | Needs IPC (pipes, sockets, shared memory) | Direct via shared memory (needs sync) |
| Crash impact | One process crashing doesn't affect others | One thread crashing can crash the whole process |
| Context switch | Expensive (flush TLB, switch page tables) | Cheaper (same address space) |

**Interview Angle:**
- Q: "Why are threads faster to create than processes?" → No new address space/page tables need to be set up; the OS just allocates a new stack, registers, and TCB (thread control block), reusing the parent process's memory map.
- Q: "If one thread crashes, does the whole process die?" → Yes, typically, because they share the same address space - a bad memory access (segfault) affects the entire process.

### 2.2 Process Control Block (PCB)

**Definition:** The PCB (a.k.a. Task Control Block) is a data structure maintained by the OS **for every process**, containing all the information needed to manage and resume it.

**Analogy:** The PCB is like a **patient's medical file** at a hospital. Even when the patient isn't in the room (process isn't running), the file (PCB) keeps everything needed to pick up their treatment exactly where it left off - vitals, history, current medication, etc.

**What's inside a PCB (know this list cold):**
1. **Process ID (PID)**
2. **Process State** (running, waiting, ready, etc.)
3. **Program Counter** (address of the next instruction)
4. **CPU Registers** (saved values when preempted)
5. **CPU Scheduling info** (priority, scheduling queue pointers)
6. **Memory management info** (page tables, base/limit registers)
7. **Accounting info** (CPU used, time limits)
8. **I/O status info** (open files, allocated devices)

**Why It Matters:** The PCB is what allows **context switching** to work - it's the OS's snapshot of a process, stored so the process can be paused and later resumed with zero loss of state.

**Interview Angle:**
- Q: "Where is the PCB stored?" → In kernel memory (protected, not accessible from user space).
- Q: "What's the relation between PCB and context switching?" → Context switch = save current process's CPU state *into* its PCB, then load the next process's state *from* its PCB into the CPU registers.

### 2.3 Process States & the Process Lifecycle

**Definition:** A process moves through a defined set of states during its life.

**The standard 5-state model:**
```
        admitted           interrupt
  New ─────────────► Ready ◄────────────┐
                       │                 │
                       │ scheduler       │
                       ▼ dispatch        │
                    Running ─────────────┘
                     │    │
       I/O or event  │    │ exit
          wait       ▼    ▼
                   Waiting  Terminated
                     │
                     │ I/O or event completion
                     ▼
                   Ready
```

- **New:** Process is being created.
- **Ready:** Process is loaded in memory, waiting for CPU allocation.
- **Running:** Instructions are being executed by the CPU.
- **Waiting/Blocked:** Process is waiting for an event (I/O completion, signal).
- **Terminated:** Process has finished execution.

**Analogy:** Think of a **queue at a bank**. *New* = you just walked in and are filling a form. *Ready* = you're sitting in the waiting area, token in hand. *Running* = you're at the counter being served. *Waiting* = the clerk says "wait, I need to verify something," so you step aside (I/O wait) - but you're not back in the general queue, you're in a special "on-hold" spot. *Terminated* = your work is done, you leave.

**Why It Matters:** Understanding state transitions is essential for understanding **scheduling** (Module 3) - the scheduler is the entity that moves processes between Ready and Running.

**Interview Angle:**
- Q: "Can a process go directly from Waiting to Running?" → No. It must go through Ready first - the scheduler decides when it actually gets the CPU.
- Q: "What causes a Running → Waiting transition?" → An I/O request, waiting on a lock/semaphore, or waiting for a signal.
- Q: "What causes Running → Ready (not Waiting)?" → Preemption - e.g., a timer interrupt expires the process's time slice (only in preemptive scheduling).

### 2.4 Context Switching

**Definition:** The mechanism by which the CPU switches from executing one process/thread to another, saving the current state and restoring the state of the next one.

**Analogy:** It's like a **surgeon leaving mid-operation** to attend to an emergency case. Before leaving, they carefully note down exactly where they stopped, what instruments were in use, vitals, etc. (save state into PCB). They then scrub in for the new patient using *their* saved notes (load new PCB). Later, they can return to the first patient and resume exactly where they left off.

**Why It Matters:** Context switching is what enables multitasking - but it's **pure overhead**; no useful work is done during the switch itself. This is why excessive context switching hurts performance (this ties into thrashing in Module 7).

**What happens during a context switch:**
1. Save the current process's CPU state (registers, PC, stack pointer) into its PCB.
2. Update the PCB's state field (e.g., Running → Ready/Waiting).
3. Move the PCB to the appropriate queue.
4. Select the next process via the scheduler.
5. Load the new process's state from its PCB into CPU registers.
6. Update memory management registers (e.g., page table base register) if switching processes.
7. Jump to the new process's program counter.

**Interview Angle:**
- Q: "Why is context switching between threads faster than between processes?" → Threads of the same process share the address space, so there's no need to switch page tables or flush the TLB - a very expensive part of process context switches.
- Q: "Is context switching triggered only by the timer interrupt?" → No - also by I/O interrupts, system calls that block, or higher-priority process arrivals in preemptive systems.

## MODULE 3: CPU Scheduling

### 3.1 Preemptive vs. Non-Preemptive Scheduling

**Definition:**
- **Non-preemptive:** Once a process gets the CPU, it keeps it until it voluntarily releases it (finishes or blocks on I/O).
- **Preemptive:** The OS can forcibly take the CPU away from a running process (e.g., a higher-priority process arrives, or a time slice expires).

**Analogy:** Non-preemptive is like a **single-lane toll booth with no cutting** - once a car starts being served, it finishes completely before the next one begins, no matter how long it takes. Preemptive is like an **ER triage system** - if a more critical patient arrives, the doctor may pause the current (less urgent) patient to attend to the emergency.

**Why It Matters:** Preemptive scheduling gives better responsiveness (important for interactive systems) but has overhead from context switching and needs careful synchronization to avoid race conditions (Module 4). Non-preemptive is simpler but can cause **starvation** for shorter jobs stuck behind long ones (convoy effect).

**Interview Angle:**
- Q: "Which scheduling algorithms are inherently non-preemptive?" → FCFS, and SJF/Priority *in their pure form* (non-preemptive variants exist for both).
- Q: "What's the 'convoy effect'?" → Short processes get stuck waiting behind one long process holding the CPU, drastically increasing average waiting time - a classic FCFS weakness.

### 3.2 Key Scheduling Metrics (memorize these formulas)

- **Arrival Time (AT):** When the process enters the ready queue.
- **Burst Time (BT):** Total CPU time the process needs.
- **Completion Time (CT):** When the process finishes execution.
- **Turnaround Time (TAT)** = CT − AT (total time from arrival to completion)
- **Waiting Time (WT)** = TAT − BT (time spent waiting in ready queue)
- **Response Time (RT)** = Time of first CPU allocation − AT (important for interactive systems)

**Goal of scheduling algorithms:** minimize average waiting time, minimize turnaround time, maximize CPU utilization and throughput, and be fair (avoid starvation).

### 3.3 First-Come, First-Served (FCFS)

**Definition:** Processes are executed strictly in order of arrival. Non-preemptive.

**Analogy:** A single queue at a grocery store checkout - first in line, first served, no exceptions.

**How to solve a problem:**
1. Sort processes by arrival time.
2. Each process starts right after the previous one finishes (or at its own arrival time, whichever is later).
3. Compute CT, TAT, WT for each.

**Worked example:**
| Process | AT | BT |
|---|---|---|
| P1 | 0 | 5 |
| P2 | 1 | 3 |
| P3 | 2 | 8 |

Gantt chart: `P1(0-5) → P2(5-8) → P3(8-16)`
- P1: CT=5, TAT=5-0=5, WT=5-5=0
- P2: CT=8, TAT=8-1=7, WT=7-3=4
- P3: CT=16, TAT=16-2=14, WT=14-8=6
- Avg WT = (0+4+6)/3 = 3.33

**Why It Matters / Downsides:** Simple but causes the **convoy effect** - a short process arriving right after a long one has started must wait a long time.

**Interview Angle:**
- Q: "Is FCFS preemptive or non-preemptive?" → Non-preemptive.
- Q: "What's its biggest weakness?" → Convoy effect / poor average waiting time when burst times vary widely.

### 3.4 Shortest Job First (SJF) - Non-preemptive & Preemptive (SRTF)

**Definition:** SJF picks the process with the **smallest burst time** next. The preemptive version is called **Shortest Remaining Time First (SRTF)** - if a new process arrives with a shorter remaining burst than the currently running one, it preempts it.

**Analogy:** A supermarket express lane that always calls up whoever has the **fewest items**, even if they arrived later than someone with a full cart - this minimizes the *total* time everyone spends waiting.

**Why It Matters:** SJF is **provably optimal** for minimizing average waiting time (given all burst times are known in advance) - this is a classic interview fact. But it requires knowing burst times ahead of time (often estimated via **exponential averaging** of past bursts in real systems), and can cause **starvation** of long processes if short ones keep arriving.

**Worked example (non-preemptive):**
| Process | AT | BT |
|---|---|---|
| P1 | 0 | 8 |
| P2 | 1 | 4 |
| P3 | 2 | 2 |
| P4 | 3 | 1 |

At t=0, only P1 available → runs P1 fully? No - non-preemptive SJF still only picks among *arrived* processes at the decision point. Since P1 is the only one available at t=0, it must run (0-8), even though shorter jobs arrive later. Gantt: `P1(0-8) → P4(8-9) → P3(9-11) → P2(11-15)` - after P1 finishes at 8, all of P2, P3, P4 have arrived, so pick smallest BT first (P4=1, then P3=2, then P2=4).

**Interview Angle:**
- Q: "Why is SJF optimal?" → Mathematically, among all possible orderings, scheduling the shortest job first minimizes the sum (and thus average) of waiting times - provable by an exchange argument (swapping any longer job ahead of a shorter one only increases total wait).
- Q: "Main drawback of SJF?" → (1) Requires predicting future burst time (impossible to know exactly, so it's estimated), and (2) can starve long processes.
- Q: "How is future CPU burst estimated in real OS schedulers?" → Exponential averaging: `τ(n+1) = α·t(n) + (1-α)·τ(n)`, where t(n) is the actual last burst and τ(n) is the previous estimate.

### 3.5 Round Robin (RR)

**Definition:** Each process gets a fixed **time quantum**; if it doesn't finish, it's preempted and put at the back of the ready queue. Purely preemptive.

**Analogy:** A **table tennis club with a strict 5-minute timer per player** - everyone rotates and gets a turn regardless of how much they've completed; if they're not done in their slot, they go to the back of the line and wait for their next turn.

**Why It Matters:** Excellent for time-sharing/interactive systems - every process gets fair, regular CPU access, giving good **response time**. But if the time quantum is:
- **Too large** → degenerates into FCFS.
- **Too small** → excessive context-switching overhead dominates (bad throughput).

**Worked example:** Quantum = 2
| Process | AT | BT |
|---|---|---|
| P1 | 0 | 5 |
| P2 | 0 | 3 |
| P3 | 0 | 1 |

Gantt: `P1(0-2) → P2(2-4) → P3(4-5) → P1(5-7) → P2(7-8) → P1(8-9)`
(P3 finishes in one quantum since BT=1 < quantum. P1 needs three rounds: 2+2+1=5, P2 needs two rounds: 2+1=3.)

**Interview Angle:**
- Q: "How do you choose a good time quantum?" → Rule of thumb: 80% of CPU bursts should be shorter than the time quantum. Too small → overhead-heavy; too large → behaves like FCFS.
- Q: "Does RR guarantee low waiting time?" → Not necessarily low, but **fair and bounded** - no process waits more than (n-1) × quantum for its next turn, given n processes.

### 3.6 Priority Scheduling

**Definition:** Each process is assigned a priority; the CPU is given to the highest-priority process. Can be preemptive or non-preemptive.

**Analogy:** An **airport security fast-track line** for business/first-class passengers - they get served ahead of economy passengers regardless of arrival order.

**Why It Matters - Starvation & Aging:** Low-priority processes can wait indefinitely if high-priority ones keep arriving - this is called **starvation** (a.k.a. "indefinite blocking"). The standard fix is **aging** - gradually increase the priority of processes that have waited a long time, guaranteeing they eventually run.

**Interview Angle:**
- Q: "What is starvation and how do you solve it?" → Starvation: a process never gets scheduled because lower-numbered/higher priority processes keep jumping the queue. Fix: **aging** - increase priority the longer a process waits.
- Q: "Does a lower priority *number* mean higher or lower priority?" → Convention-dependent, but in most textbooks/OS (e.g., Linux), **lower number = higher priority**. Always clarify this in an interview if asked to solve a numeric example.

### 3.7 Multilevel Queue & Multilevel Feedback Queue Scheduling

**Definition:**
- **Multilevel Queue:** Ready queue is split into multiple separate queues based on process type (e.g., system processes, interactive processes, batch processes), each with its own scheduling algorithm, and queues themselves are scheduled with fixed priority or time-slicing between queues.
- **Multilevel Feedback Queue (MLFQ):** Like multilevel queue, but processes **can move between queues** based on behavior (e.g., a CPU-bound process that keeps using its full quantum gets demoted to a lower-priority queue; an I/O-bound process that yields quickly stays in a high-priority queue).

**Analogy:** A **university with separate tracks** - undergrad, grad, faculty - each with different rules (multilevel queue). MLFQ is like a **video game matchmaking system** that automatically promotes/demotes you between skill brackets based on your recent performance, rather than a fixed track.

**Why It Matters:** MLFQ is the most general and widely used in real operating systems (Linux's CFS is conceptually related, Windows uses MLFQ-like scheduling) because it **adapts dynamically** without needing to know process behavior in advance.

**Interview Angle:**
- Q: "What's the key difference between Multilevel Queue and MLFQ?" → Multilevel Queue has fixed queue assignment (a process never changes queues); MLFQ allows dynamic movement between queues based on observed behavior.
- Q: "How does MLFQ prevent starvation of low-priority queues?" → Aging - periodically boost all processes to the topmost queue, or dedicate a certain fraction of CPU time to lower queues.

### 3.8 Scheduling Quick-Reference Table

| Algorithm | Preemptive? | Best for | Weakness |
|---|---|---|---|
| FCFS | No | Simplicity | Convoy effect |
| SJF/SRTF | Optional | Minimizing avg wait | Starvation, needs burst prediction |
| Round Robin | Yes | Interactive/time-sharing | Bad if quantum mis-tuned |
| Priority | Optional | Critical task urgency | Starvation (fix: aging) |
| MLFQ | Yes | General-purpose real OS | Complex to tune |

## MODULE 4: Process Synchronization

### 4.1 Race Conditions

**Definition:** A race condition occurs when multiple processes/threads access and manipulate shared data **concurrently**, and the final outcome depends on the unpredictable timing/order of execution.

**Analogy:** Two people simultaneously updating the **same shared bank balance** by reading it, adding money, and writing it back - without coordination. If both read "$100" before either writes back, one deposit can get silently lost.

**Why It Matters:** Race conditions are the root cause of countless subtle, hard-to-reproduce bugs. The **classic example**: `counter++` looks atomic but is actually three machine steps: (1) load counter into a register, (2) increment register, (3) store register back to counter. If two threads interleave these steps, updates can be lost.

**Interview Angle:**
- Q: "Give a classic example of a race condition." → Two threads incrementing a shared counter variable without synchronization; the classic three-step (load, increment, store) breakdown of `count++` shows how updates get lost.
- Q: "How do you detect/prevent race conditions?" → Use synchronization primitives (locks, semaphores) to enforce mutual exclusion on the shared resource - this is the **critical section problem**, discussed next.

### 4.2 The Critical Section Problem

**Definition:** A **critical section** is a code segment where a process accesses shared resources (variables, files, data structures) that must not be concurrently accessed by more than one process. The critical section problem is designing a protocol so that when one process is executing in its critical section, no other process is allowed to execute in its own.

**Analogy:** A **single-occupancy restroom with a lock** - only one person (process) can be inside (critical section) at a time; others must wait outside (in the "entry section") until the door is unlocked.

**Three requirements a valid solution must satisfy (interview gold - memorize):**
1. **Mutual Exclusion:** Only one process can be in its critical section at a time.
2. **Progress:** If no process is in the critical section, one of the processes waiting to enter must be able to do so - the decision cannot be postponed indefinitely, and it can't be made by a process that's not trying to enter.
3. **Bounded Waiting:** There must be a limit on how many times other processes can enter their critical sections before a waiting process gets its turn (no indefinite starvation).

**Interview Angle:**
- Q: "What three conditions must a critical section solution satisfy?" → Mutual exclusion, progress, bounded waiting (as above).
- Q: "What's the general structure of a process trying to use a critical section?" → Entry section (request access) → Critical section → Exit section (release access) → Remainder section.

### 4.3 Peterson's Solution

**Definition:** A classic software-only solution to the critical section problem for **two processes**, using two shared variables: a `turn` variable and a boolean `flag[]` array.

**Analogy:** Two people trying to use one doorway politely: each raises a flag saying "I want to go" (`flag[i] = true`), then politely says "but you go first if you also want to" (`turn = other`). Whoever's flag is down, or who lost the "you go first" toss, waits.

**Pseudocode (know this cold):**
```
// Process i (0 or 1); j is the other process
flag[i] = true;
turn = j;
while (flag[j] && turn == j) {
    // busy wait
}
// --- critical section ---
flag[i] = false;
// --- remainder section ---
```

**Why It Matters:** It's a foundational theoretical construct showing mutual exclusion is achievable with just shared memory (no special hardware instructions) - but it only works for 2 processes and relies on assumptions (like atomic reads/writes and instruction reordering not happening) that **modern compilers/CPUs with out-of-order execution can violate**, so it's rarely used in real production systems (hardware-based locks are preferred).

**Interview Angle:**
- Q: "Does Peterson's solution satisfy all three critical section requirements?" → Yes, theoretically: mutual exclusion (via the turn+flag combo), progress, and bounded waiting (each process waits at most one turn).
- Q: "Why isn't Peterson's Solution used in real modern systems?" → It doesn't scale beyond 2 processes easily, and it can fail on modern hardware due to instruction reordering/compiler optimizations unless memory barriers are added; real systems use hardware atomic instructions instead.

### 4.4 Hardware Support: Test-and-Set / Compare-and-Swap

**Definition:** Modern CPUs provide **atomic instructions** that read-modify-write a memory location in a single, uninterruptible step - the actual foundation of real-world locks.
- **Test-and-Set(lock):** Atomically sets `lock = true` and returns the *old* value.
- **Compare-and-Swap (CAS)(value, expected, new):** Atomically checks if `value == expected`; if so, sets `value = new`. Returns success/failure.

**Analogy:** It's like a **vending machine that either dispenses the item and takes your money in one indivisible motion, or does nothing at all** - no other customer can sneak a transaction in the middle.

**Why It Matters:** These atomic instructions are the actual building block used to implement mutexes, spinlocks, and semaphores in real operating systems and lock-free data structures.

**Interview Angle:**
- Q: "What is Compare-and-Swap used for?" → Implementing lock-free/wait-free data structures and building higher-level synchronization primitives like mutexes without needing OS-level blocking.

### 4.5 Mutex vs. Semaphore (very frequently asked - get this exact)

**Definition:**
- **Mutex (Mutual Exclusion Lock):** A binary lock, owned by whoever locks it. Only the thread that locked it can unlock it. Purpose: protect a critical section (ensures **exclusive access**).
- **Semaphore:** An integer variable accessed only via two atomic operations, `wait()` (a.k.a. `P()`, decrements) and `signal()` (a.k.a. `V()`, increments). Can be:
  - **Binary semaphore:** value is 0 or 1 (similar to mutex, but no ownership).
  - **Counting semaphore:** value can range over any integer, used to control access to a resource pool with multiple identical instances (e.g., N available database connections).

**Analogy:** A **mutex** is like a **single bathroom key** shared by roommates - only whoever took the key can return it. A **counting semaphore** is like a **parking garage with N spots and a counter at the entrance** - the counter (semaphore value) tracks how many spots are free; any car (thread) can decrement it on entry and any car leaving increments it, no "ownership" of the counter itself.

**Key distinction table (memorize):**

| Aspect | Mutex | Semaphore |
|---|---|---|
| Purpose | Mutual exclusion (lock) | Signaling AND resource counting |
| Ownership | Yes - only the locker can unlock | No ownership - any thread can signal |
| Value range | Binary (locked/unlocked) | Binary or counting (integer) |
| Use case | Protect a critical section | Manage a pool of N resources; coordinate between threads (e.g., producer-consumer) |

**Semaphore pseudocode:**
```
wait(S) {          // P operation
    while (S <= 0) ; // busy-wait (or block, in real impl)
    S--;
}
signal(S) {         // V operation
    S++;
}
```

**Interview Angle:**
- Q: "Can a semaphore be signaled by a different thread than the one that waited on it?" → Yes! This is the key difference from a mutex - a semaphore has no ownership, which makes it usable for **signaling** between threads (e.g., a consumer thread can `wait()` and a producer thread can `signal()`), not just mutual exclusion.
- Q: "Give an example where you'd use a semaphore but not a mutex." → Producer-Consumer with a bounded buffer of size N - a counting semaphore tracks "empty slots" and "full slots"; this coordination pattern (one thread waits, a *different* thread signals) can't be done cleanly with a mutex.

### 4.6 Classical Synchronization Problems

#### 4.6.1 Producer-Consumer Problem (Bounded Buffer)

**Definition:** Producers generate data items and put them into a shared, fixed-size buffer; consumers remove items from the buffer. Must ensure producers don't add to a full buffer and consumers don't remove from an empty buffer, with mutual exclusion on buffer access.

**Analogy:** A **bakery conveyor belt with limited shelf space** - bakers (producers) can't put bread on the belt if it's full; customers (consumers) can't take bread if the belt is empty.

**Solution using semaphores:**
```
semaphore empty = N;   // counts empty slots
semaphore full = 0;    // counts filled slots
semaphore mutex = 1;   // binary, protects buffer access

// Producer
wait(empty);
wait(mutex);
   add_item_to_buffer();
signal(mutex);
signal(full);

// Consumer
wait(full);
wait(mutex);
   remove_item_from_buffer();
signal(mutex);
signal(empty);
```

**Interview Angle:**
- Q: "Why do we need both `empty`/`full` counting semaphores AND a `mutex`?" → `empty`/`full` handle the **capacity constraint** (blocking when buffer is full/empty), while `mutex` handles **mutual exclusion** on the actual buffer data structure so two producers/consumers don't corrupt it simultaneously.
- Q: "What happens if you swap the order of `wait(mutex)` and `wait(empty)` in the producer?" → **Deadlock risk!** If mutex is acquired first and then the buffer turns out to be full, the producer blocks on `wait(empty)` while still holding the mutex - the consumer can never acquire the mutex to free up space, causing a deadlock.

#### 4.6.2 Readers-Writers Problem

**Definition:** A shared resource (e.g., a database) is read by multiple "reader" threads and modified by "writer" threads. Multiple readers can read simultaneously (no conflict), but a writer needs **exclusive** access (no readers or other writers at the same time).

**Analogy:** A **shared Google Doc in "view mode" vs "edit mode"** - any number of people can view simultaneously, but when someone wants to edit, everyone else must be locked out until they're done.

**Why It Matters:** There are two classic variants:
- **Readers-preference:** readers never wait if other readers are already reading (can starve writers).
- **Writers-preference:** once a writer is waiting, no new readers are allowed to jump the queue (can starve readers).

**Interview Angle:**
- Q: "How would you prevent writer starvation in a readers-preference solution?" → Use a fair queuing mechanism/aging, or switch to a writers-preference or strictly fair (FIFO) design where a waiting writer blocks new readers from entering.

#### 4.6.3 Dining Philosophers Problem

**Definition:** N philosophers sit at a round table, alternating between thinking and eating. Between each pair of adjacent philosophers is one shared fork, and eating requires picking up **both** the left and right forks. This models resource allocation / potential deadlock among competing processes.

**Analogy:** Exactly as described - 5 philosophers, 5 forks, each needing 2 forks to eat. If everyone picks up their left fork at the same time, everyone will be stuck waiting forever for their right fork - a **deadlock**.

**Classic solutions:**
1. **Resource ordering:** Number the forks; each philosopher picks up the lower-numbered fork first. This breaks the circular wait condition needed for deadlock.
2. **Limit concurrent diners:** Allow at most N-1 philosophers to attempt to pick up forks simultaneously (using a semaphore), guaranteeing at least one can always get both forks.
3. **Arbitrator/waiter:** A central semaphore/monitor grants permission before a philosopher may pick up any fork at all.

**Interview Angle:**
- Q: "What does the Dining Philosophers problem model, and why is it famous?" → It models **deadlock and starvation in concurrent resource allocation** - a classic scenario used to test understanding of how naive locking can lead to circular waits.
- Q: "How does 'pick up lower-numbered fork first' prevent deadlock?" → It breaks the **circular wait** condition (Module 5) - since forks are always acquired in a globally consistent order, you cannot form a cycle of philosophers each waiting on the next.

## MODULE 5: Deadlocks

### 5.1 What is a Deadlock?

**Definition:** A deadlock is a state where a set of processes are each waiting for a resource held by another process in the same set, so **none of them can ever proceed**.

**Analogy:** Two cars in a **single-lane bridge, facing each other**, each waiting for the other to reverse - neither will ever move unless something external intervenes.

### 5.2 Four Necessary Conditions for Deadlock (Coffman Conditions)

#**All four must hold simultaneously for deadlock to occur - this is the #1 most-asked deadlock question:**

1. **Mutual Exclusion:** At least one resource must be held in a non-shareable mode (only one process can use it at a time).
2. **Hold and Wait:** A process holding at least one resource is waiting to acquire additional resources currently held by other processes.
3. **No Preemption:** Resources cannot be forcibly taken away from a process; they must be released voluntarily.
4. **Circular Wait:** There exists a set of processes {P1, P2, ..., Pn} such that P1 waits for a resource held by P2, P2 waits for P3, ..., and Pn waits for P1 - forming a cycle.

**Interview Angle:**
- Q: "Name the four necessary conditions for deadlock." → Mutual exclusion, hold-and-wait, no preemption, circular wait (as above) - **removing any ONE breaks the possibility of deadlock**, which is the entire basis for prevention strategies.

### 5.3 Resource Allocation Graph (RAG)

**Definition:** A directed graph where processes and resources are nodes. A **request edge** (P → R) means the process is requesting the resource; an **assignment edge** (R → P) means the resource is currently allocated to that process.

**Analogy:** Think of it as a **flowchart of "who owes what to whom"** - if you can trace a full circle (a cycle) in the graph, you've found a group stuck waiting on each other.

**Rule:**
- If the graph has **no cycle** → no deadlock.
- If the graph has a cycle **and each resource type has only one instance** → deadlock is guaranteed.
- If a resource type has **multiple instances**, a cycle is **necessary but not sufficient** for deadlock (it might still resolve).

**Interview Angle:**
- Q: "If a Resource Allocation Graph has a cycle, is the system definitely deadlocked?" → Only if every resource in the cycle has a **single instance**. With multiple instances of a resource, a cycle indicates *possible* deadlock, not certain deadlock.

### 5.4 Handling Deadlocks: Four Strategies

#### 5.4.1 Deadlock Prevention
**Definition:** Design the system so that at least one of the four necessary conditions can *never* hold.

| Condition Attacked | Strategy |
|---|---|
| Mutual Exclusion | Make resources shareable where possible (not always feasible, e.g., printers) |
| Hold and Wait | Require processes to request all resources upfront, or release held resources before requesting new ones |
| No Preemption | Allow the OS to forcibly take resources away (preempt) if a process requests something unavailable |
| Circular Wait | Impose a total ordering on resource types; processes must request resources in increasing order |

#### 5.4.2 Deadlock Avoidance - Banker's Algorithm

**Definition:** The system dynamically examines each resource request and only grants it if the resulting state remains a **"safe state"** (a state from which there's a sequence of allocations that lets every process finish). Requires the OS to know in advance the **maximum resource demand** of each process.

**Analogy:** A **bank manager who never approves a loan** if doing so risks leaving the bank without enough cash to satisfy at least one customer's maximum claim, even in a worst-case scenario. The bank only lends when it's confident it can eventually satisfy everyone in *some* order.

**Core idea for the interview:**
- A state is **safe** if there exists an order to run all processes to completion such that each process's remaining need can be satisfied by (currently available resources + resources that will be freed by processes that finish before it).
- The algorithm checks: "if I grant this request, is the system still in a safe state?" If yes, grant it. If no, the process must wait.

**Data structures used:** `Available[]` (free resources), `Max[][]` (max demand per process per resource), `Allocation[][]` (currently allocated), `Need[][] = Max - Allocation`.

**Interview Angle:**
- Q: "What's the difference between a safe state and an unsafe state?" → A safe state guarantees deadlock can be avoided (there's *some* order to satisfy everyone). An unsafe state doesn't guarantee deadlock will happen, but it means deadlock *could* happen depending on future requests - so the algorithm refuses to enter an unsafe state at all.
- Q: "What's the biggest practical limitation of Banker's Algorithm?" → It requires knowing the **maximum resource need of every process in advance**, which is rarely realistic in general-purpose systems - hence it's more of a theoretical/embedded-systems tool than a real desktop-OS mechanism.

#### 5.4.3 Deadlock Detection & Recovery

**Definition:** Instead of preventing/avoiding deadlock, let it happen, but periodically run a **detection algorithm** (similar to Banker's, using a wait-for graph or resource allocation graph) to check if a deadlock exists, then **recover**.

**Recovery methods:**
1. **Process termination:** Kill one process (or all deadlocked processes) to break the cycle.
2. **Resource preemption:** Forcibly take a resource from one process and give it to another, then roll that process back.

**Interview Angle:**
- Q: "If detection finds a deadlock, how do you choose which process to kill?" → Consider factors like process priority, how long it's run, how many resources it holds, how many more it needs, and whether it's interactive or batch - kill the one with the least "cost" to abort.

#### 5.4.4 Deadlock Ignorance (Ostrich Algorithm)

**Definition:** Simply ignore the problem - assume deadlocks are rare enough that the cost of prevention/detection outweighs the cost of occasionally rebooting. **This is what most general-purpose OSes (Linux, Windows) actually do** for general deadlocks, because prevention has heavy performance costs.

**Interview Angle:**
- Q: "Why don't real operating systems implement Banker's Algorithm system-wide?" → The overhead of tracking every process's maximum resource claims is impractical for a general-purpose OS with dynamic, unpredictable workloads - most OSes use the "ostrich algorithm" and let higher-level software (databases, etc.) handle their own deadlock detection where it truly matters.

## MODULE 6: Memory Management

### 6.1 Logical (Virtual) Address vs. Physical Address

**Definition:**
- **Logical/Virtual Address:** The address generated by the CPU during program execution - what the program "sees."
- **Physical Address:** The actual address in the RAM hardware.

**Analogy:** It's like a **hotel room number on your key card (virtual) vs. the actual GPS coordinates of that room in the building (physical)**. You use the room number to navigate; the hotel's internal system maps it to real coordinates. If you're moved to a different room during your stay, your key card experience is the same, but the physical mapping changed.

**Why It Matters:** This separation, done via the **Memory Management Unit (MMU)**, is what allows: (1) processes to run without knowing their actual physical memory location, (2) multiple processes to have overlapping virtual address ranges without collision, and (3) memory protection (a process can't accidentally address another's physical memory).

**Interview Angle:**
- Q: "What hardware component translates logical to physical addresses?" → The MMU (Memory Management Unit).
- Q: "Why do we even need virtual addresses instead of just using physical addresses directly?" → Protection (isolate processes), flexibility (processes don't need to know their physical location, easing relocation), and enabling techniques like paging/virtual memory (allowing a process to use more memory than physically available).

### 6.2 Contiguous Memory Allocation

**Definition:** Each process is allocated one single contiguous block of physical memory.

**Fixed Partitioning vs. Dynamic Partitioning:**
- **Fixed:** Memory divided into fixed-size partitions ahead of time; simple but causes **internal fragmentation** (wasted space *inside* a partition if the process is smaller than the partition).
- **Dynamic:** Partitions created exactly the size needed; avoids internal fragmentation but causes **external fragmentation** (free memory scattered in small chunks between allocated blocks, none big enough for a new request).

**Allocation strategies for dynamic partitioning:**
- **First Fit:** Allocate the first free block big enough.
- **Best Fit:** Allocate the smallest free block that's big enough (minimizes leftover, but leaves tiny unusable slivers - worsens fragmentation over time).
- **Worst Fit:** Allocate the largest free block (leaves a large usable remainder, but often performs poorly in practice).

**Analogy:** Imagine **parking cars (processes) in a single row of parking spots (memory)**. If a car leaves from the middle, you get a gap - if no new car exactly fits that gap, it stays wasted (external fragmentation) until a "compaction" (defragmentation) crunches all cars together.

**Interview Angle:**
- Q: "Internal vs. external fragmentation - define both clearly." → Internal: wasted space *within* an allocated block because it's larger than needed. External: wasted space *between* allocated blocks - total free memory may be sufficient, but it's too fragmented to satisfy a request.
- Q: "How can you fix external fragmentation?" → **Compaction** (shuffle memory contents to consolidate free space) - expensive; or switch to **non-contiguous allocation** (paging) which eliminates the problem structurally.

### 6.3 Paging

**Definition:** A non-contiguous memory allocation scheme where physical memory is divided into fixed-size blocks called **frames**, and logical memory is divided into same-sized blocks called **pages**. Any page can be placed in any free frame - no need for contiguity.

**Analogy:** Think of a **book split into equal-sized pages, and a bookshelf with equal-sized slots** - pages of the book don't need to sit in physically adjacent slots on the shelf; a **table of contents (page table)** tells you exactly which shelf slot holds each page.

**How address translation works:**
- A logical address is split into a **page number (p)** and a **page offset (d)**.
- The **page table** (one per process) maps page number → frame number.
- Physical address = (frame number × frame size) + offset.

```
Logical Address = [ Page Number | Offset ]
                         │
                         ▼ (lookup in Page Table)
                   Frame Number
                         │
                         ▼
Physical Address = [ Frame Number | Offset ]
```

**Why It Matters:** Paging completely **eliminates external fragmentation** (any free frame works for any page) - though it introduces **internal fragmentation** in the last page of a process (if the process size isn't an exact multiple of page size).

**Interview Angle:**
- Q: "Does paging suffer from external or internal fragmentation?" → Internal only (in the last page); external fragmentation is eliminated because any free frame can hold any page.
- Q: "Where is the page table stored, and what's the performance concern?" → In main memory. The concern: every memory access now requires **two** memory accesses (one to read the page table, one for the actual data) - this is why the TLB (next section) exists.

### 6.4 Translation Lookaside Buffer (TLB)

**Definition:** A small, fast, hardware **cache** inside the CPU that stores recent virtual-to-physical page translations, avoiding a full page table lookup in memory for every access.

**Analogy:** If the page table is a **giant phone book you'd have to flip through every time**, the TLB is like **remembering the last few numbers you dialed** - if it's a repeat call, you skip the phone book entirely and dial instantly from memory.

**How it works:**
1. CPU generates a virtual address, extracts the page number.
2. Check the TLB first.
3. **TLB Hit:** Frame number found immediately → fast physical address computed.
4. **TLB Miss:** Must access the page table in memory (slow), then load that mapping into the TLB for next time (may evict an old entry).

**Why It Matters:** TLB hit rate is critical for performance - a high hit rate (typical: 98%+ in real workloads) means most memory accesses skip the expensive page table walk.

**Interview Angle:**
- Q: "What happens on a context switch regarding the TLB?" → Since each process has its own page table, TLB entries from the old process are no longer valid - the TLB is typically **flushed** (or, in modern CPUs, entries are tagged with an **Address Space ID / ASID** so multiple processes' entries can coexist without flushing, improving switch performance).
- Q: "What's 'Effective Access Time (EAT)' and how do you calculate it?" → `EAT = hit_ratio × (TLB access time + memory access time) + (1 − hit_ratio) × (TLB access time + 2 × memory access time)` - a classic numeric interview problem. E.g., TLB access = 20ns, memory access = 100ns, hit ratio = 80%: EAT = 0.8×(20+100) + 0.2×(20+200) = 96 + 44 = 140ns.

### 6.5 Segmentation

**Definition:** A memory management scheme that divides a process's memory into **variable-sized logical segments** based on the program's actual structure - e.g., code segment, data segment, stack segment, heap segment - rather than fixed-size pages.

**Analogy:** Instead of chopping a book into arbitrary equal pages (paging), segmentation is like organizing it by **meaningful chapters of different lengths** - "Introduction," "Methods," "Results" - each is a logical unit that can grow/shrink independently.

**Why It Matters:** Segmentation matches how programmers actually think about a program (functions, arrays, stacks as distinct logical units), and allows fine-grained protection (e.g., mark the code segment read-only/executable, stack read-write). But being variable-sized, it **reintroduces external fragmentation**, which is why most modern systems use **paged segmentation** (a hybrid: segments divided further into pages) to get the best of both.

**Interview Angle:**
- Q: "Paging vs. Segmentation - key difference?" → Paging divides memory into **fixed-size, physically-motivated** blocks invisible to the programmer; segmentation divides memory into **variable-sized, logically-motivated** units that match the program's structure. Paging avoids external fragmentation but causes internal fragmentation; segmentation is the reverse.

### 6.6 Inverted Page Table

**Definition:** Instead of one page table **per process** (which can consume huge memory for large address spaces), an inverted page table keeps **one single system-wide table with one entry per physical frame**, recording which process and which page currently occupies that frame.

**Analogy:** A traditional page table is like **every hotel guest carrying their own full map of the entire hotel** (wasteful if the hotel is huge). An inverted page table is like the **hotel keeping one master registry, indexed by room number, listing which guest currently occupies each room** - one compact list instead of thousands of guest-specific maps.

**Why It Matters:** It saves massive memory (table size is proportional to *physical* memory size, not virtual address space × number of processes), but lookups become slower because you must search by physical frame using process ID + virtual page as the search key (usually mitigated with a hash table).

**Interview Angle:**
- Q: "What's the main trade-off of an inverted page table?" → Saves memory (one entry per physical frame, not per virtual page per process), but lookup is slower since you now need to *search* the table (typically via hashing) rather than directly indexing it.

## MODULE 7: Virtual Memory

### 7.1 Demand Paging

**Definition:** Pages are loaded into physical memory **only when they are actually needed/referenced** (i.e., "lazily"), rather than loading an entire process into memory upfront.

**Analogy:** It's like **streaming a movie instead of downloading the whole file first** - you only pull in the chunk of data (page) you're about to watch (access), not the entire content upfront.

**Why It Matters:** This is the core mechanism that allows **virtual memory** - a process can have a virtual address space *larger* than physical RAM, because not all of it needs to be resident at once.

### 7.2 Page Faults

**Definition:** A page fault is a trap/interrupt generated when a program accesses a page that is **marked invalid** (not currently in physical memory).

**How it's handled step by step (a very common interview walkthrough question):**
1. CPU generates a virtual address; MMU checks the page table.
2. Page table entry's **valid/invalid bit** shows the page isn't in memory → trap to OS (page fault).
3. OS checks if the reference was a legal memory access (if illegal → terminate process/segfault).
4. If legal, OS finds a **free frame** (or evicts a page - see replacement algorithms below).
5. OS schedules a disk I/O to read the required page into that frame.
6. Page table is updated (valid bit set, frame number filled in).
7. The instruction that caused the fault is **restarted**.

**Analogy:** You reach for a book on your desk (memory), but it's not there - it's in the library (disk). You send someone to fetch it (disk I/O), and once it arrives, you resume reading exactly where you left off.

**Interview Angle:**
- Q: "Walk me through what happens on a page fault." → (Use the 7 steps above.)
- Q: "Is a page fault an error?" → Not necessarily - it's a normal, expected part of demand paging (though *excessive* page faults indicate a performance problem - thrashing).

### 7.3 Page Replacement Algorithms

**Definition:** When a page fault occurs and there's **no free frame**, the OS must choose an existing page to evict ("victim") to make room.

#### 7.3.1 FIFO (First-In-First-Out)
**Definition:** Evict the page that has been in memory the **longest**, regardless of usage.
**Analogy:** A queue at a cafeteria - whoever's been sitting there longest gets asked to leave first, even if they just started eating.
**Weakness - Belady's Anomaly:** Increasing the number of frames can, counter-intuitively, **increase** the number of page faults under FIFO (this does NOT happen with LRU or Optimal).

#### 7.3.2 Optimal (OPT / MIN)
**Definition:** Evict the page that **won't be used for the longest time in the future**.
**Analogy:** A psychic librarian who knows exactly which book won't be needed again for the longest time, and removes that one.
**Why It Matters:** This is the theoretical **best possible** algorithm (lowest possible fault rate) - but it's **impossible to implement in practice** because it requires knowing the future reference string. It's used only as a benchmark to evaluate other algorithms.

#### 7.3.3 LRU (Least Recently Used)
**Definition:** Evict the page that hasn't been used for the **longest time in the past** (approximates Optimal by assuming past behavior predicts the future).
**Analogy:** Cleaning out your closet by donating the clothes you haven't worn in the longest time - a reasonable, practical proxy for "won't need it again soon."
**Implementation approaches:** Counters (timestamp every access - expensive), or a **stack** (move accessed page to top; least recently used is at the bottom).

**Worked Example - Reference String: `7, 0, 1, 2, 0, 3, 0, 4, 2, 3`, 3 frames:**

| Algorithm | Page Faults |
|---|---|
| FIFO | 9 faults (with 3 frames) |
| LRU | 8 faults |
| Optimal | 7 faults |

(This is the exact classic example from Silberschatz's OS textbook - highly likely to appear verbatim or with small changes in interviews. Practice tracing this by hand.)

**Interview Angle:**
- Q: "What is Belady's Anomaly, and which algorithm(s) suffer from it?" → The counter-intuitive phenomenon where adding *more* frames leads to *more* page faults. FIFO suffers from it; LRU and Optimal are "stack algorithms" and provably never suffer from it.
- Q: "Why is LRU generally preferred over FIFO in practice?" → LRU better approximates real program behavior via the principle of **locality of reference** (recently used pages are likely to be used again soon), giving fewer faults on average, and it doesn't suffer from Belady's Anomaly.
- Q: "How would you approximate true LRU efficiently in hardware?" → Using a **reference bit** per page (set on access, periodically cleared) - algorithms like **Second-Chance (Clock)** use this to approximate LRU cheaply without full timestamping.

### 7.4 Thrashing

**Definition:** A state where the system spends **more time paging (swapping pages in/out) than executing actual instructions**, because processes don't have enough frames to hold their "working set" of actively used pages.

**Analogy:** Imagine a student trying to study from **one single library book slot** but needing to reference 5 different books constantly - they spend all their time walking back and forth to the library returning/fetching books, and almost no time actually reading.

**Why It Matters (the causal chain - a favorite interview trace):**
1. CPU utilization is low.
2. OS's scheduler sees low CPU utilization and (mistakenly) thinks it should increase multiprogramming - admits **more** processes.
3. More processes compete for the same limited physical frames → each gets even fewer frames.
4. More page faults occur → even more time spent on I/O, even less actual CPU work.
5. CPU utilization drops further → OS admits still more processes → **vicious cycle** = thrashing.

**How to fix / prevent thrashing:**
- **Working Set Model:** Track each process's actively used pages over a recent time window; ensure enough frames are allocated to hold this working set.
- **Page Fault Frequency (PFF) control:** Directly monitor each process's fault rate; if too high, give it more frames; if too low, take some away.
- Reduce the degree of multiprogramming (suspend/swap out some processes entirely).

**Interview Angle:**
- Q: "Explain thrashing and how the OS can detect and resolve it." → (Use the causal chain + working set/PFF fixes above.)
- Q: "What's the 'working set' of a process?" → The set of pages a process has referenced in the most recent Δ time units - a practical approximation of "the pages it needs right now" to avoid faulting.

## MODULE 8: Storage & File Systems

### 8.1 Disk Scheduling Algorithms

**Definition:** Algorithms that determine the **order** in which pending disk I/O requests (at various cylinder/track positions) are serviced, to minimize the total seek time (the head movement) - a major hard-disk performance bottleneck.

#### 8.1.1 FCFS (Disk)
Service requests strictly in arrival order. Simple, but can cause the disk head to "wildly swing" across the disk if requests are scattered.

#### 8.1.2 SSTF (Shortest Seek Time First)
**Definition:** Always service the request **closest** to the current head position.
**Analogy:** A bus that always picks up whichever waiting passenger is physically nearest, regardless of who's been waiting longest.
**Weakness:** Can cause **starvation** for requests far from the "hot zone" of activity, similar to SJF's starvation issue.

#### 8.1.3 SCAN ("Elevator Algorithm")
**Definition:** The disk head moves in one direction, servicing all requests along the way, until it reaches the end of the disk, then **reverses direction** and services requests on the way back.
**Analogy:** Exactly like a **building elevator** - it doesn't jump around; it continues in one direction, serving all floor requests along the way, before reversing.

#### 8.1.4 C-SCAN (Circular SCAN)
**Definition:** Like SCAN, but instead of reversing direction at the end, the head jumps back to the **beginning** of the disk without servicing requests on the return trip, then starts scanning forward again.
**Why It Matters vs SCAN:** Provides more **uniform wait times** - SCAN can make requests just behind the head wait almost a full sweep, while requests just missed at the far end get serviced almost immediately after the reversal (uneven). C-SCAN treats the disk as a circular list, giving more consistent (fair) response times.

#### 8.1.5 LOOK / C-LOOK
**Definition:** Same as SCAN/C-SCAN, but the head only travels as far as the **last request** in each direction (instead of going all the way to the physical end of the disk), then reverses/jumps - saving unnecessary travel.

**Quick comparison:**

| Algorithm | Behavior | Weakness |
|---|---|---|
| FCFS | Service in arrival order | Poor performance, erratic head movement |
| SSTF | Closest request first | Starvation of far requests |
| SCAN | Sweep back and forth, serve all | Uneven wait time at extremes |
| C-SCAN | Sweep one direction, jump back | Slightly more overhead (return jump) but very fair |
| LOOK/C-LOOK | Like SCAN/C-SCAN but only to last request | More efficient than SCAN/C-SCAN (no wasted travel) |

**Interview Angle:**
- Q: "Why is C-SCAN often preferred over SCAN?" → It provides more **uniform/fair wait times** across all disk regions, because it treats the disk as circular rather than doubling back and re-favoring the middle region.
- Q: "Numeric problem type to expect:" Given a disk with cylinders 0–199, head starting at, say, cylinder 53, and a request queue like `98, 183, 37, 122, 14, 124, 65, 67`, calculate total head movement for each algorithm. **Practice tracing these by hand** - this is an extremely common whiteboard question.

### 8.2 File Allocation Methods

**Definition:** How the OS keeps track of which disk blocks belong to which file.

#### 8.2.1 Contiguous Allocation
Each file occupies a set of **contiguous** blocks on disk.
- **Pro:** Excellent read performance (sequential access is fast, minimal seek); simple to implement.
- **Con:** External fragmentation; hard to grow a file if adjacent blocks are taken.

#### 8.2.2 Linked Allocation
Each file is a **linked list** of disk blocks, scattered anywhere, each block holding a pointer to the next.
- **Pro:** No external fragmentation; files can grow easily.
- **Con:** No efficient random access (must traverse the list from the start); pointers waste some space; reliability risk (a single corrupted pointer breaks the chain).
- **Variant - FAT (File Allocation Table):** Keeps all the "next block" pointers in one table in a reserved area of the disk instead of inside each block, enabling faster traversal and easier random access lookups.

#### 8.2.3 Indexed Allocation
Each file has a dedicated **index block** containing pointers to all of its data blocks (like a table of contents).
- **Pro:** Supports efficient direct/random access (look up the index, jump straight to the block) without external fragmentation.
- **Con:** Overhead of maintaining the index block; for very large files, may need **multi-level or linked index blocks** (this is exactly how Unix inodes work, with direct blocks, single/double/triple indirect blocks).

**Analogy:**
- **Contiguous** = a novel printed as one unbroken block of consecutive pages.
- **Linked** = a treasure hunt where each clue (block) tells you where to find the next one - no map, but no wasted space either.
- **Indexed** = a book with a table of contents (index block) pointing directly to the page number of every chapter (data block) - fast lookup of any section without reading sequentially.

**Interview Angle:**
- Q: "Which allocation method does a typical Unix/Linux file system (ext-family) use, conceptually?" → Indexed allocation via **inodes**, with direct pointers for small files and single/double/triple indirect pointers to scale to very large files without a huge fixed-size index.
- Q: "Why is indexed allocation generally preferred over linked allocation for large files needing random access?" → Because linked allocation requires sequential traversal to reach any given block (O(n) for random access), whereas indexed allocation gives near O(1) direct access via the index block.

## FINAL CHEAT SHEET: Top 10 High-Yield OS Interview Questions

**1. Q: What's the difference between a process and a thread?**
A: A process is an independent execution unit with its own memory space; a thread is a lightweight execution unit *within* a process that shares memory/resources with sibling threads but has its own stack and registers.

**2. Q: What are the four necessary conditions for deadlock?**
A: Mutual exclusion, hold-and-wait, no preemption, circular wait - all four must hold simultaneously; breaking any one prevents deadlock.

**3. Q: Mutex vs. Semaphore?**
A: A mutex is a locking mechanism with ownership (only the locker can unlock) used purely for mutual exclusion. A semaphore is an integer counter (binary or counting) with no ownership, used for both mutual exclusion and signaling/coordination between different threads.

**4. Q: What happens during a page fault?**
A: CPU accesses a page marked invalid → trap to OS → OS validates the access, finds/evicts a frame, schedules disk I/O to load the page, updates the page table, and restarts the faulting instruction.

**5. Q: What is thrashing and what causes it?**
A: Thrashing is when the system spends more time page-faulting/swapping than doing actual work, caused by too many processes competing for too few physical frames - fixed via the working set model or reducing the degree of multiprogramming.

**6. Q: Explain Belady's Anomaly.**
A: The counter-intuitive case where *adding more* physical frames leads to *more* page faults - it affects FIFO but never LRU or Optimal (which are "stack algorithms").

**7. Q: Paging vs. Segmentation?**
A: Paging splits memory into fixed-size, physically-driven units (no external fragmentation, some internal fragmentation). Segmentation splits memory into variable-sized, logically-driven units matching program structure (no internal fragmentation, but external fragmentation possible).

**8. Q: What are the 3 requirements of a valid Critical Section solution?**
A: Mutual exclusion, progress, and bounded waiting.

**9. Q: fork() vs exec()?**
A: `fork()` creates a near-duplicate child process (copy of the parent's memory/code, new PID). `exec()` replaces the calling process's own memory image with a new program (same PID). Shells typically fork, then exec in the child.

**10. Q: How does the Banker's Algorithm avoid deadlock?**
A: Before granting any resource request, it simulates the allocation and checks whether the resulting state is "safe" (i.e., there's still some order in which all processes can finish); it only grants requests that keep the system in a safe state.

#### Final Prep Tip
For coding-round interviewers at product companies, expect **numeric tracing problems** far more than pure definitions - practice by hand: Gantt charts for scheduling (Module 3), LRU/FIFO/Optimal fault counting (Module 7), and disk head movement totals (Module 8). These show up in 70%+ of OS interview rounds at companies like Amazon and Microsoft. Good luck!