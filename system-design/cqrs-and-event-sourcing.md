---
title: "CQRS and Event Sourcing"
description: "Separating reads from writes with CQRS, storing state as a log of events with event sourcing, how the two work together, and when the complexity is worth it."
author: ["name": "Rajendra Pancholi", "email": "rpancholi522@gmail.com"]
created: "2026-04-18"
updated: "2026-10-17"
thumbnail: "/images/command-and-query-responsibility-segregation.png"
tags: [cqrs, event-sourcing, architecture, event-driven, system-design]
keywords: ["CQRS pattern", "Event sourcing", "Command Query Responsibility Segregation", "Read model projection", "Event store", "CQRS vs event sourcing"]
---

# CQRS and Event Sourcing

**CQRS** (Command and Query Responsibility Segregation) separates **write operations (commands)** from **read operations (queries)** into distinct models. **Event sourcing** stores a system's state as a **sequence of events** instead of overwriting the current state. They are separate patterns that are often used together, so this guide covers both.

![CQRS Pattern](/images/command-and-query-responsibility-segregation.png)

## Part 1: CQRS

### The Problem

In a typical application one model handles both reading and writing. That works until the needs diverge:

- Writes need **strict validation, normalized data, and transactions**.
- Reads need **fast, denormalized, query-friendly views** (a dashboard that joins six tables, a search page).
- Read traffic is often **orders of magnitude higher** than write traffic, so the two need to scale differently.

Forcing one model to serve both makes it a poor compromise for each.

### The Idea

- **Command side:** handles state-changing operations (create, update, delete). A command expresses intent (`PlaceOrder`), is validated, and may be rejected. It returns success or failure, not data. Optimized for correctness and write performance.
- **Query side:** handles reads. It never changes state. It uses a **read model**, a pre-computed, often denormalized view designed for fast queries.

```text
            Commands                          Queries
Client ───► Command handler ──► Write model   Client ◄─── Query handler ◄── Read model
                                    │                                           ▲
                                    └────────── events / change feed ───────────┘
                                              (updates the read model)
```

### Levels of CQRS

CQRS is not all-or-nothing:

1. **Same database, separate models:** separate command and query code paths (and often separate classes), but one database. Low cost and a good first step.
2. **Separate read store:** the read model lives in its own database, chosen for query needs (a search engine, a document store, a cache). It is kept up to date from the write side.
3. **Fully separate services:** command and query sides deployed and scaled independently.

### Keeping the Read Model in Sync

The read model is updated from the write side, typically by:

- **Events:** the write side publishes events (`OrderPlaced`) and **projections** update the read model (see [Event-Driven Architecture](system-design/event-driven)).
- **Change data capture (CDC):** a tool reads the write database's change log and feeds the read store.

Because this is usually asynchronous, the read side is **eventually consistent**: there is a short lag between a write and the read model reflecting it.

### Handling Read-Your-Own-Writes

A user submits a form and then immediately views a page that does not show their change yet. This confuses users. Common fixes:

- Return the **created or updated data** in the command response and show it directly in the UI.
- **Optimistically update** the UI, then confirm.
- Read from the write model for a short time after a write, or pass a **version or timestamp** so the query waits until the read model has caught up.
- Route the user's own reads to a source that is guaranteed up to date (see [Consistency Models](system-design/consistency-models)).

### When to Use CQRS

**Good fit:**

- Read and write workloads that differ greatly in scale.
- Read views that are very different from the write model (complex reporting, search, dashboards).
- Complex business rules on the write side that you want to keep separate from query concerns.
- Collaborative domains with many users acting on the same data.

**Poor fit:**

- Simple CRUD applications. CQRS adds code, infrastructure, and consistency problems for no gain.
- Teams that cannot yet handle eventual consistency.

### Disadvantages

- More moving parts: two models, plus synchronization.
- **Eventual consistency** between sides, which users and developers must account for.
- Duplicate data and the work of keeping projections correct.
- More to test, monitor, and operate.

## Part 2: Event Sourcing

### The Idea

Normally a database stores the **current state**: a bank account row says `balance = 70`. When it changes, the old value is overwritten and the history is gone.

With **event sourcing**, you store every change as an immutable event, and the current state is **derived** by replaying them:

```text
AccountOpened(id=42)
MoneyDeposited(amount=100)
MoneyWithdrawn(amount=30)
        ──► replay ──► balance = 70
```

The sequence of events (the **event store**) is the source of truth. Events are **append-only**: you never edit or delete them. A mistake is fixed by adding a new compensating event (such as `WithdrawalReversed`), not by changing history.

### Benefits

- **Complete audit trail** for free: you know exactly what happened and why.
- **Time travel:** rebuild the state as of any past moment, useful for debugging and "what if" analysis.
- **Rebuild and add read models** at any time by replaying events, including views you did not think of originally.
- Natural fit for **event-driven** systems, since events already exist.
- Good for domains where history matters: finance, orders, inventory, compliance.

### Key Concepts

| Concept                    | Meaning                                                                                                 |
| -------------------------- | ------------------------------------------------------------------------------------------------------- |
| **Event store**            | Append-only storage of events, grouped by entity (an "aggregate" or "stream")                           |
| **Aggregate**              | The entity whose state is rebuilt from its events (an account, an order)                                |
| **Projection**             | Code that consumes events to build a read model                                                         |
| **Snapshot**               | A saved copy of the state at some event number, so you replay only the events after it                  |
| **Optimistic concurrency** | Each write states the version it expects; if the stream has moved on, the write is rejected and retried |

### Snapshots

Replaying thousands of events to load one entity is slow. A **snapshot** stores the state every N events (say, every 100). To load, read the latest snapshot and replay only the events after it. Snapshots are just an optimization, and you can always discard them and rebuild from events.

### Challenges

- **Event schema evolution:** events live forever, so a change in shape must keep old events readable. Techniques include versioned events and **upcasting** (converting old versions to the new shape when reading).
- **Querying is hard:** you cannot easily ask "which accounts have balance over 1000?" from the event store alone. This is why event sourcing is almost always paired with projections or CQRS.
- **Deleting data:** immutability conflicts with privacy rules such as the right to be forgotten. A common approach is **crypto-shredding**: encrypt personal data with a per-person key and delete the key when erasure is requested.
- **Replay cost and complexity:** rebuilding large projections takes time, and the whole model is unfamiliar to many developers.
- **Eventual consistency** between events and projections.

### Implementation Options

- A purpose-built event store such as **EventStoreDB**.
- An **append-only table** in a relational database (simple and works well for modest scale).
- A log such as **Kafka** can carry events, but be careful using it as the long-term source of truth for individual aggregates (per-entity reads and optimistic concurrency are awkward there).

## CQRS and Event Sourcing Together

They complement each other but are **independent**:

|                   | CQRS only           | Event sourcing only                            | Both                          |
| ----------------- | ------------------- | ---------------------------------------------- | ----------------------------- |
| Write side stores | Current state       | Events                                         | Events                        |
| Read side         | Separate read model | Rebuilt by replay each time (slow for queries) | Read models built from events |

- You can use CQRS without event sourcing (the write model stores plain current state and publishes change events).
- You can use event sourcing without CQRS, but querying becomes painful.
- Together: the write side validates commands and appends events; projections turn events into read models for fast queries.

Both patterns add significant complexity. Adopt them **only for the part of the system that benefits**, not the whole application.

## Trade-offs

| Choice         | You gain                                     | You give up                                         |
| -------------- | -------------------------------------------- | --------------------------------------------------- |
| CQRS           | Independent scaling and tailored read models | Simplicity, immediate consistency                   |
| Event sourcing | Audit trail, replay, time travel             | Simple querying, easy schema changes, data deletion |
| Snapshots      | Fast loading                                 | Extra mechanism to maintain                         |

## Real-World Usage

- **Banking and accounting ledgers:** balances derived from an immutable list of transactions (accounting has always worked this way).
- **E-commerce orders:** an order's lifecycle (placed, paid, shipped, returned) as events, with read models for the order-history page and for analytics.
- **Collaborative editing and audit-heavy systems** where "who changed what, when" is a requirement.

## Interview Questions

**Q: What is CQRS and why use it?**
It separates the write model (commands) from the read model (queries) so each can be optimized and scaled independently, especially when read and write workloads or data shapes differ a lot.

**Q: What is the downside of CQRS?**
Added complexity and eventual consistency between the write and read sides, which requires handling cases like read-your-own-writes.

**Q: What is event sourcing?**
Storing state as an append-only sequence of events and deriving the current state by replaying them, instead of storing and overwriting the current state.

**Q: How does event sourcing handle performance when many events have accumulated?**
With snapshots: store the state periodically and replay only the events that came after the latest snapshot.

**Q: How do you handle personal data deletion with immutable events?**
Crypto-shredding: encrypt personal data with a per-subject key, and delete the key to make the data unreadable. Alternatively, keep personal data outside the events.

**Q: Do you need event sourcing to use CQRS?**
No. They are independent patterns. CQRS can use ordinary state storage on the write side.

## Summary

- CQRS splits commands (writes) from queries (reads) so each side can use the model that fits it.
- The read model is usually eventually consistent, so plan for read-your-own-writes.
- Event sourcing stores events as the source of truth and derives state by replay, giving audit, replay, and time travel.
- Challenges include schema evolution, querying, data deletion, and complexity.
- They work well together but are independent. Use them selectively, not everywhere.

## Continue Learning

- [Event-Driven Architecture](system-design/event-driven) — the events that feed projections.
- [Consistency Models](system-design/consistency-models) — what eventual consistency means for users.
- [Distributed Transactions](system-design/distributed-transactions) — coordinating across services.
- [Event Streaming (Kafka)](system-design/kafka) — a common transport for events.
