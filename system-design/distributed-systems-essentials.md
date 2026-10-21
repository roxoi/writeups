---
title: "Distributed Systems Essentials — Transactions, Consensus, Leader Election, Locks, Clocks, and IDs"
description: "The distributed systems concepts every system design interview expects: why distribution is hard, two-phase commit vs Saga, consensus and Raft, leader election, distributed locks and fencing tokens, clocks and ordering, unique ID generation, Bloom filters and HyperLogLog."
author: ["name": "Rajendra Pancholi", "email": "rpancholi522@gmail.com"]
created: "2026-10-17"
updated: "2026-10-17"
thumbnail: "/images/system-design-arch.png"
tags: [distributed-systems, saga, two-phase-commit, raft, consensus, leader-election, snowflake, bloom-filter, system-design]
keywords: ["Distributed systems basics", "Saga pattern vs two-phase commit", "Raft consensus explained", "Leader election and fencing token", "Distributed lock", "Snowflake ID generation", "Bloom filter", "Lamport clock vector clock"]
---

# Distributed Systems Essentials — Transactions, Consensus, Leader Election, Locks, Clocks, and IDs

Once a system runs on more than one machine, new problems appear that don't exist on a single server. This guide covers the handful of ideas that come up again and again in system design interviews and in real architectures: coordinating transactions across services, agreeing on a single leader, locking safely, ordering events, generating unique IDs, and answering questions approximately but cheaply.

Related topics are covered elsewhere: consistent hashing, replication, quorums, and CAP are in [Databases](system-design/databases); idempotency, Kafka, and the outbox pattern are in [Messaging and Communication](system-design/messaging-and-communication) and [Event-Driven Architecture](system-design/event-driven).

## Why Distributed Systems Are Hard

On one machine, a function call either returns or the program crashes. Across machines, much more can go wrong:

- **The network is unreliable.** Messages can be lost, delayed, duplicated, or reordered.
- **Partial failure.** Some machines fail while others keep running, and you often can't tell which.
- **No shared clock.** Each machine has its own clock, and they drift apart.
- **No shared memory.** Machines only know what messages tell them, and the information is always slightly stale.

### You Can't Tell Slow from Dead

If you send a request and get no reply, any of these could be true: the request was lost, the server crashed, the server is just slow, or the reply was lost. A **timeout** is only a guess.

This single fact explains most of the rest of this guide:

- Retries can cause **duplicates**, so operations must be idempotent.
- A node declared dead by a timeout may actually still be alive, which leads to **split-brain** (two leaders).
- Coordination protocols must work correctly even when their failure detection is wrong.

## Distributed Transactions

In a microservices system, one business operation (placing an order) touches several services, each with its own database. A normal ACID transaction can't span them. Two approaches exist.

### Two-Phase Commit (2PC)

A **coordinator** drives all participants to either all commit or all abort.

```
Phase 1 (Prepare):  Coordinator ──► "Can you commit?" ──► all participants
                    Each participant locks resources, writes to its log, replies YES or NO
Phase 2 (Commit):   If all said YES:  Coordinator ──► "COMMIT" to everyone
                    If any said NO:   Coordinator ──► "ABORT" to everyone
```

**Strengths:** strong atomicity, all-or-nothing across participants.

**Problems:**

- **Blocking:** if the coordinator crashes after participants voted YES, they must wait, holding locks, until it recovers, because they can't decide alone.
- **Latency:** several network round trips, and locks are held for the whole protocol.
- **Availability:** every participant must be up for a commit to happen.
- Not supported by many NoSQL databases and message brokers, and a poor fit for services that must stay independent.

2PC is used **inside** some databases (for example across shards or replicas in distributed SQL systems), but is rarely used **across microservices**.

### Saga Pattern

A **saga** is a sequence of **local transactions**, one per service. Each step commits on its own. If a later step fails, the saga runs **compensating transactions** to undo the effects of the earlier steps, in reverse order.

**Example: place an order**

```
1. Order Service      : create order (PENDING)          ◄── compensate: cancel order
2. Payment Service    : charge the customer             ◄── compensate: refund
3. Inventory Service  : reserve stock                   ◄── compensate: release stock
4. Shipping Service   : schedule delivery               ◄── compensate: cancel shipment
5. Order Service      : mark order CONFIRMED

If step 3 fails (out of stock):
   run compensations in reverse: refund payment → cancel order (CANCELLED)
```

**Two ways to coordinate a saga** (also described in [Event-Driven Architecture](system-design/event-driven)):

|            | Choreography                                           | Orchestration                                                         |
| ---------- | ------------------------------------------------------ | --------------------------------------------------------------------- |
| How        | Each service reacts to events and emits the next event | A central orchestrator tells each service what to do and tracks state |
| Coupling   | Very loose                                             | Services depend on the orchestrator                                   |
| Visibility | Hard to see the whole flow                             | Whole flow is in one place                                            |
| Best for   | Short, simple sagas                                    | Long or complex flows with branching                                  |

**Rules for building a saga correctly:**

- **Every step and every compensation must be idempotent and retryable,** because messages may be delivered more than once and steps may be retried after failures.
- Compensations are **semantic undo**, not a rollback: you can't un-send an email. Design a business-level reversal (send a cancellation notice) instead.
- Use the **transactional outbox** so that "update my database" and "publish the next event" can't be separated by a crash.
- Persist the saga's state so it can resume after a crash.
- **Sagas lack isolation:** other requests can see intermediate states (an order that is PENDING while payment is charged). Reduce the damage with explicit states (`PENDING`, `CONFIRMED`), semantic locks (don't let a user modify a pending order), and ordering steps so the hardest-to-undo step comes last.

### 2PC vs Saga

|              | 2PC                                            | Saga                                      |
| ------------ | ---------------------------------------------- | ----------------------------------------- |
| Consistency  | Atomic (all or nothing)                        | Eventually consistent (with compensation) |
| Isolation    | Yes (locks held)                               | No (intermediate states visible)          |
| Availability | Lower (all participants must be up)            | Higher (each step independent)            |
| Latency      | Locks held across round trips                  | Each step commits quickly                 |
| Complexity   | In the protocol                                | In your compensation logic                |
| Typical use  | Within a database or tightly coupled resources | Across microservices                      |

**In an interview:** prefer **avoiding** distributed transactions by putting data that must change together in one service. When you can't, use a saga, and say clearly which compensations exist and how duplicates are handled.

## Consensus and Leader Election

### The Problem

A group of nodes must **agree on a value** (who is the leader, what is the next entry in the log, what is the current configuration), even though some nodes may crash and messages may be lost. This is **consensus**.

Consensus is the foundation of coordination services and replicated databases. Real systems that rely on it: **etcd** and **Consul** (Raft), **ZooKeeper** (ZAB, a similar protocol), **Kafka** (KRaft, built on Raft), **Spanner** (Paxos), **CockroachDB** (Raft).

### Quorums: The Core Idea

Decisions need a **majority** (a quorum) of nodes. Any two majorities overlap in at least one node, so two conflicting decisions can't both succeed.

| Cluster size | Majority needed | Failures tolerated |
| ------------ | --------------- | ------------------ |
| 3            | 2               | 1                  |
| 5            | 3               | 2                  |
| 7            | 4               | 3                  |

With `n = 2f + 1` nodes you survive `f` failures. **Even sizes don't help:** 4 nodes need 3 for a majority and still tolerate only 1, the same as 3 nodes. Production clusters are typically 3 or 5 nodes; more nodes make writes slower, not faster.

### Raft in Plain Language

Raft is a consensus algorithm designed to be understandable. Each node is a **follower**, **candidate**, or **leader**.

1. **Leader election:** time is split into numbered **terms**. If a follower hears no heartbeat from a leader within a randomized **election timeout**, it becomes a candidate, increments the term, and asks for votes. A candidate that wins votes from a **majority** becomes leader. The randomized timeout makes ties rare.
2. **Log replication:** all writes go through the leader. It appends the command to its log and sends it to followers. Once a **majority** have stored the entry, it is **committed** and applied, and the client gets its answer.
3. **Safety:** committed entries are never lost. A candidate can only win if its log is at least as up to date as a majority, so a new leader always has every committed entry.
4. **Failure:** if the leader dies, followers time out and elect a new one. Minority-side nodes during a partition can't commit anything, which prevents split-brain.

**Paxos** solves the same problem and is older and harder to understand; Raft is what most newer systems use.

> A theoretical result (FLP) says no algorithm can guarantee consensus in a purely asynchronous system if even one node may crash. Practical algorithms use timeouts and randomization, so they can pause during bad network conditions but never return wrong answers.

### Leader Election in Practice

Many systems need exactly one leader at a time (one primary database, one scheduler, one cron runner). Instead of implementing consensus yourself, use a coordination service:

- **etcd / ZooKeeper / Consul:** each candidate tries to create a key (or ephemeral node) with a **lease**. The one that succeeds is leader and must keep renewing the lease. If it dies, the lease expires and others compete.
- **Database-based:** a row with `leader_id` and `lease_expires_at`, updated with a conditional update. Simple, and fine for modest needs.

**The danger: a leader that doesn't know it lost leadership.** A paused leader (long garbage collection, network stall) may wake up thinking it is still in charge while a new leader has been elected. Both then act, which is **split-brain**. The cure is fencing (next section).

## Distributed Locks and Fencing Tokens

A distributed lock ensures that only one process at a time does something (processes a payment, runs a job, edits a resource).

### Why Locks Are Dangerous

Locks are held with a **lease** (expiry), so a crashed holder can't block everyone forever. But consider:

1. Process A acquires the lock (lease 30 seconds).
2. A pauses for 40 seconds (garbage collection, slow network).
3. The lease expires; process B acquires the lock.
4. A wakes up, still believes it holds the lock, and writes to the shared resource.

Now both A and B are acting, and the lock didn't protect anything.

### Fencing Tokens

The fix is a **fencing token**: every time the lock is granted, it comes with a number that **always increases** (33, then 34, and so on). The protected resource (database, storage) **remembers the highest token it has seen and rejects any request with a lower one.**

```
A gets lock with token 33 → pauses
B gets lock with token 34 → writes (resource records "34")
A wakes up, writes with token 33 → resource rejects (33 < 34)
```

This makes the **resource** the final judge, not the lock holder's belief.

### Practical Advice

- A lock service built on consensus (etcd, ZooKeeper) can give you monotonic tokens. A single Redis lock (`SET key value NX EX ttl`) doesn't provide fencing by itself. It is fine as an **efficiency optimization** (avoid duplicate work), but not as the **only** safeguard for correctness. Multi-node Redis lock algorithms exist but remain debated for strict safety.
- For correctness, prefer a **conditional write in the database** (`UPDATE ... WHERE status = 'AVAILABLE'` or a version check), which is effectively optimistic concurrency. See how this is applied in [Design Uber](system-design/uber-design).
- Always release a lock only if you still own it (compare the value before deleting).
- Keep critical sections short, and set the lease longer than the work needs.

## Time, Clocks, and Ordering

### Why Wall Clocks Can't Order Events

Each machine's clock drifts and is corrected by network time protocols, so two machines can differ by milliseconds (or far more when something goes wrong). A clock can even jump backwards. If machine A timestamps an event at 10:00:00.005 and machine B at 10:00:00.003, you can't conclude that B's event came first.

Consequences: **last-write-wins by timestamp can silently discard the newer write**, and "expire at time T" checks across machines can disagree.

### Logical Clocks

Instead of real time, track **causality**: what could have influenced what.

- **Lamport clock:** every node keeps a counter. Increment it on each event; stamp each message with it; on receiving a message, set `counter = max(local, received) + 1`. If event A caused event B, then `A's timestamp < B's timestamp`. But a smaller timestamp does **not** prove causality, and two concurrent events can't be told apart.
- **Vector clock:** each node keeps a counter **per node**. Comparing two vectors tells you whether one event happened before the other **or whether they were concurrent** (neither happened-before). Used by Dynamo-style databases to detect conflicting writes that need merging.
- **Hybrid logical clock (HLC):** combines physical time with a logical counter, so timestamps stay close to real time while preserving causality. Used by some distributed SQL databases.

**Spanner's TrueTime** exposes clock uncertainty explicitly (using atomic clocks and GPS) and waits out the uncertainty window, which is how it achieves strongly consistent global transactions. It is an exception that requires special hardware.

### Handling Conflicts

When two replicas accept concurrent writes to the same data, you need a strategy:

| Strategy                    | Notes                                                                                     |
| --------------------------- | ----------------------------------------------------------------------------------------- |
| **Last-write-wins**         | Simple, but can silently lose data, and relies on clocks                                  |
| **Version vectors + merge** | Detect conflicts and merge by application logic                                           |
| **CRDTs**                   | Data types designed so concurrent updates always merge deterministically (counters, sets) |
| **Avoid conflicts**         | Route all writes for an item to a single leader                                           |

## Unique ID Generation

In a distributed system many machines must create IDs **without coordinating** on every call, and ideally IDs that sort roughly by time.

| Approach                          | Pros                                                              | Cons                                                             |
| --------------------------------- | ----------------------------------------------------------------- | ---------------------------------------------------------------- |
| **Auto-increment (single DB)**    | Simple, ordered                                                   | Single point of failure and bottleneck; collisions across shards |
| **UUID v4** (random)              | No coordination, collision-free in practice                       | 128 bits, not time-ordered, poor index locality                  |
| **UUID v7 / ULID** (time-ordered) | No coordination, sortable by creation time, better index locality | Larger than 64-bit IDs                                           |
| **Snowflake-style**               | 64-bit, roughly time-ordered, very fast                           | Needs unique machine IDs; depends on clocks                      |

### Snowflake-Style IDs

A 64-bit ID built from three parts:

```
| 1 bit unused | 41 bits timestamp (ms) | 10 bits machine ID | 12 bits sequence |
```

- **41-bit timestamp** (milliseconds since a custom epoch): lasts about 69 years.
- **10-bit machine ID:** up to 1,024 machines, each generating independently.
- **12-bit sequence:** up to 4,096 IDs per machine per millisecond (about 4 million per second per machine).

**Issues to mention:** if a machine's **clock goes backwards**, it could repeat IDs, so the generator must wait or refuse until time catches up; and machine IDs must be assigned uniquely (for example by a coordination service). IDs are roughly sorted by time, which makes database indexes efficient. See [Databases](system-design/databases) for IDs across shards.

## Failure Detection and Gossip

- **Heartbeats:** nodes send periodic "I'm alive" messages; silence beyond a timeout marks a node as suspected dead. Too short a timeout causes false alarms; too long delays recovery.
- **Adaptive detectors** (such as phi accrual) estimate suspicion from observed heartbeat patterns instead of a fixed timeout.
- **Gossip protocols:** each node periodically shares what it knows (membership, state) with a few random peers. Information spreads exponentially fast, needs no central coordinator, and tolerates failures. Used for cluster membership in Cassandra and Consul.

## Probabilistic Data Structures

When exact answers are too expensive, these structures trade a small, controlled error for a huge saving in memory.

### Bloom Filter

Answers "**is this item in the set?**" using a bit array and several hash functions.

- Answers are either **"definitely not in the set"** or **"possibly in the set."** There are **false positives**, but **never false negatives**.
- Items can't be removed (a counting variant can).

**Sizing:** for `n` items and a target false-positive rate `p`:

```
bits needed  m = −n × ln(p) / (ln 2)²
hash functions k = (m / n) × ln 2
```

For 1 million items at a 1% false-positive rate: about **9.6 bits per item, roughly 1.2 MB total, with 7 hash functions**, far smaller than storing the items themselves.

**Uses:**

- **Skip needless lookups:** LSM-tree databases (Cassandra, RocksDB) check a Bloom filter before reading a disk file; caches reject requests for keys that can't exist (preventing cache penetration, see [Caching and CDN](system-design/caching-and-cdn)).
- **Web crawlers:** "have I already seen this URL?"
- **Username or email checks** before hitting the database.

### HyperLogLog

Estimates the number of **distinct items** (unique visitors, unique searches) using a tiny, fixed amount of memory.

- Redis's implementation uses at most about **12 KB** per counter, with a standard error of about **0.81%**, whether you count thousands or billions of items.
- Counters can be **merged**, so you can combine per-day counts into a weekly total.

### Count-Min Sketch

Estimates **how often** each item appears (frequency), using a small grid of counters. It can **overestimate but never underestimate**. Used for "top K" heavy hitters (trending hashtags, most-requested keys) and rate limiting at scale.

## Interview Questions

**Q: Why can't we just use a normal database transaction across microservices?**
Each service owns its own database, so there is no single transaction manager, and holding locks across network calls hurts availability. Use a saga, or keep data that must change together inside one service.

**Q: 2PC or Saga?**
2PC gives atomicity and isolation but blocks if the coordinator fails and requires every participant to be available, so it suits tightly coupled resources. A saga is eventually consistent with compensating actions, and it is the usual choice across microservices.

**Q: What makes a saga reliable?**
Idempotent, retryable steps and compensations, durable saga state, an outbox for publishing events, and explicit intermediate states to handle the lack of isolation.

**Q: How does Raft elect a leader and keep data safe?**
Followers that stop hearing from the leader become candidates after a randomized timeout and request votes; a majority makes a leader. Writes are committed once a majority have the log entry, and only candidates with up-to-date logs can win, so committed data survives leader changes.

**Q: How many nodes should a consensus cluster have?**
Usually three or five. With 2f+1 nodes you tolerate f failures; even numbers add cost without extra fault tolerance.

**Q: How do you prevent split-brain?**
Require a majority (quorum) to act as leader, so at most one side of a partition can proceed, and use fencing tokens so a stale leader's writes are rejected.

**Q: What is a fencing token?**
A monotonically increasing number issued with each lock or lease. The protected resource rejects requests carrying a token lower than one it has already seen, so a paused ex-leader can't corrupt data.

**Q: Why not order events with timestamps?**
Clocks on different machines drift and can jump, so timestamps can't prove which event came first. Use logical or vector clocks for causality, or route conflicting writes through one leader.

**Q: Design a unique ID generator for a distributed system.**
Use a Snowflake-style 64-bit ID: timestamp, machine ID, and a per-millisecond sequence. It generates IDs without coordination and sorts by time. Handle clock regression and ensure unique machine IDs.

**Q: When would you use a Bloom filter?**
When a cheap "definitely not present" check saves an expensive lookup, such as avoiding database reads for non-existent keys or skipping disk files in an LSM tree, and a small false-positive rate is acceptable.

## Summary

- Distribution brings unreliable networks, partial failures, and unsynchronized clocks, and you can never tell a slow node from a dead one.
- Across services, avoid distributed transactions if you can; otherwise use **sagas** with idempotent steps and compensations. **2PC** is mostly kept inside databases.
- **Consensus** (Raft, Paxos) uses majority quorums to agree on leaders and logs; use etcd, ZooKeeper, or Consul instead of building it.
- Leases can expire while the holder is paused, so protect resources with **fencing tokens** or conditional writes.
- Wall clocks can't order events. Use logical or vector clocks, or avoid conflicts altogether.
- **Snowflake-style IDs** give unique, time-ordered 64-bit IDs without coordination.
- **Bloom filters, HyperLogLog, and Count-Min sketches** give approximate answers with tiny memory.

## Continue Learning

- [Databases](system-design/databases) — replication, quorums, consistent hashing, and CAP.
- [Event-Driven Architecture](system-design/event-driven) — choreography, orchestration, and the outbox pattern.
- [Messaging and Communication](system-design/messaging-and-communication) — idempotency, Kafka, and delivery guarantees.
- [Design Uber](system-design/uber-design) — locks and conditional updates in a real design.
- [High Availability](system-design/high-availability) — failover and split-brain in practice.
