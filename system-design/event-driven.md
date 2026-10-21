---
title: "Event-Driven Architecture"
description: "How services communicate through events: event types, choreography vs orchestration, delivery guarantees, idempotent consumers, the outbox pattern, ordering, and schema evolution."
author: ["name": "Rajendra Pancholi", "email": "rpancholi522@gmail.com"]
created: "2026-04-18"
updated: "2026-10-17"
thumbnail: "/images/message-broker.png"
tags: [event-driven, architecture, messaging, outbox, choreography, system-design]
keywords: ["Event-driven architecture", "Choreography vs orchestration", "Outbox pattern", "Event schema evolution", "At-least-once delivery", "Idempotent consumer"]
---

# Event-Driven Architecture

In event-driven architecture (EDA), services **communicate by publishing and consuming events** through a message broker, instead of calling each other directly with synchronous requests. A service announces that something happened; any number of other services react in their own time.

## The Basics

### Events vs Commands

|                      | Event                                       | Command                               |
| -------------------- | ------------------------------------------- | ------------------------------------- |
| Meaning              | Something **that already happened**         | A **request** to do something         |
| Naming               | Past tense: `OrderPlaced`, `UserRegistered` | Imperative: `PlaceOrder`, `SendEmail` |
| Receivers            | Zero, one, or many consumers                | Exactly one handler                   |
| Sender's expectation | Does not care who reacts                    | Expects the action to be performed    |

### Key Components

- **Event producers** generate events when something happens (an order is placed).
- **Event broker / router** receives events and delivers them to interested consumers (Apache Kafka, RabbitMQ, cloud queues and topics). See [Message Queues](system-design/messaging-and-communication) and Kafka.
- **Event consumers** react to events and update their own state or trigger further work.

### Example: Placing an Order

```text
Order Service ── OrderPlaced ──►  Broker
                                   ├──► Payment Service      (charge the card)
                                   ├──► Inventory Service    (reserve stock)
                                   ├──► Notification Service (send confirmation)
                                   └──► Analytics Service    (record the sale)
```

The Order Service does not know which services react. Adding a new reaction (say, a loyalty-points service) means subscribing a new consumer, with no change to the Order Service.

## Advantages

- **Loose coupling:** producers do not know about consumers, and consumers do not know about each other.
- **Scalability:** consumers scale independently, and slow work is absorbed by the broker's buffer.
- **Resilience:** if a consumer is down, events wait in the broker and are processed when it recovers.
- **Extensibility:** new consumers can be added without touching producers.
- **Audit trail and replay:** with a durable log, you have a history of what happened and can rebuild state or fix bugs by reprocessing.

## Common Event Patterns

| Pattern                          | Idea                                                                                       | Trade-off                                                                                                     |
| -------------------------------- | ------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------- |
| **Event notification**           | A small event says "something changed" (with an ID). Consumers call back to fetch details. | Simple and small, but creates callback traffic and coupling to the producer's API                             |
| **Event-carried state transfer** | The event carries all the data consumers need.                                             | Consumers need no callback and can work offline, but events are larger and consumers hold copies of data      |
| **Event sourcing**               | Events are the source of truth; state is derived from them.                                | Powerful audit and replay, but complex (see [CQRS and Event Sourcing](system-design/cqrs-and-event-sourcing)) |

### Choreography vs Orchestration

There are two ways to coordinate a multi-step workflow across services:

|                  | Choreography                                                            | Orchestration                                                             |
| ---------------- | ----------------------------------------------------------------------- | ------------------------------------------------------------------------- |
| How              | Each service reacts to events and emits its own. No central controller. | A central orchestrator tells each service what to do and tracks progress. |
| Coupling         | Very loose                                                              | Services depend on the orchestrator                                       |
| Visibility       | Hard to see the whole flow                                              | Whole flow is in one place                                                |
| Failure handling | Spread across services                                                  | Centralized                                                               |
| Best for         | Simple flows with few steps                                             | Complex flows with branching and compensation                             |

Complex business processes with rollback (a saga) often use an orchestrator for clarity. See [Distributed Transactions](system-design/distributed-systems-essentials).

## Delivery Guarantees and Idempotency

Brokers offer different delivery guarantees:

| Guarantee         | Meaning                                  | Risk                    |
| ----------------- | ---------------------------------------- | ----------------------- |
| **At-most-once**  | A message is delivered zero or one time  | Messages can be lost    |
| **At-least-once** | A message is delivered one or more times | **Duplicates** possible |
| **Exactly-once**  | Processed exactly one time               | Very hard end to end    |

In practice, most systems use **at-least-once delivery** and make consumers **idempotent**, meaning processing the same event twice has the same effect as processing it once. "Exactly-once processing" is usually achieved this way: at-least-once delivery plus deduplication or idempotent operations.

Ways to make a consumer idempotent:

- Record processed **event IDs** and skip any already seen.
- Use **natural idempotency**: "set status to SHIPPED" is safe to repeat, "add 1 to counter" is not.
- Use a **unique constraint** in the database so duplicates fail harmlessly.

See [Idempotency and Retries](system-design/messaging-and-communication).

## Ordering

Brokers typically guarantee order only **within a single queue or partition**, not across the whole system. To keep related events in order (all events for one order, for example), publish them with the same **partition key** (such as `orderId`) so they land in the same partition and are processed sequentially. Do not design as if there were a global order.

## The Outbox Pattern (The Dual-Write Problem)

A service often needs to update its database _and_ publish an event. Doing both separately is unsafe:

- Save to the database, then crash before publishing: the event is **lost**.
- Publish first, then the database write fails: the event describes something that **never happened**.

The **transactional outbox** pattern solves this:

1. In **one local database transaction**, write the business data and also insert the event into an `outbox` table.
2. A separate **relay** reads new rows from the outbox and publishes them to the broker. This is done by polling or by change data capture (CDC) tools such as Debezium.
3. Mark the outbox rows as sent.

The relay may publish a message twice (for example, if it crashes after publishing but before marking it sent), so consumers must be idempotent. The result is **at-least-once delivery without losing events**.

## Failure Handling

- **Retries** with exponential backoff for temporary failures.
- **Dead-letter queue (DLQ):** events that keep failing are moved aside so they do not block the rest. Monitor and replay them after a fix.
- **Poison messages** (events that can never be processed) must not block a partition forever.
- **Consumer lag** (how far behind consumers are) is a key health metric: alert when it grows.
- **Replay:** with a durable log, you can reprocess past events to rebuild a read model or recover from a bug. Consumers must be able to handle replays safely.

## Event Design and Schema Evolution

Events are a **contract** between teams, and old events may live for a long time, so design them carefully:

- Name events in **past tense** and describe a business fact, not a technical step.
- Include a unique **event ID**, a **timestamp**, a **type**, and a **schema version**.
- Treat published events as **immutable**.
- Use a defined schema format (Avro, Protobuf, or JSON Schema) and, where available, a **schema registry** to enforce compatibility.

Compatibility rules:

- **Backward compatible:** new consumers can read old events.
- **Forward compatible:** old consumers can read new events.
- Safe changes: add **optional** fields. Unsafe changes: rename or remove fields, or change the meaning or type of a field. Introduce a new event version instead.

## Challenges

- **Eventual consistency:** after an event is published, other services' data is briefly out of date.
- **Debugging:** a flow spans many services and happens asynchronously. You need correlation IDs and [distributed tracing](system-design/observability-and-deployment).
- **Duplicates and ordering** must be handled explicitly.
- **Testing** asynchronous flows is harder than request-response.
- **Hidden coupling:** if services depend on each other's events without clear ownership, you get a tangled web that is hard to change.

## When to Use EDA

**Good fit:** workflows where work can happen later (notifications, analytics, indexing), many consumers react to the same fact, spiky load needs a buffer, or systems need to be loosely coupled.

**Poor fit:** operations that need an immediate, strongly consistent answer (checking a balance before allowing a purchase), or very simple systems where a plain function call is enough.

Most real systems **mix** both: synchronous APIs for queries and immediate needs, events for everything that can happen afterward.

## Trade-offs

| Choice                         | You gain                              | You give up                         |
| ------------------------------ | ------------------------------------- | ----------------------------------- |
| Events instead of direct calls | Decoupling, resilience, extensibility | Immediate consistency, easy tracing |
| At-least-once delivery         | No lost messages                      | Must handle duplicates              |
| Choreography                   | Looser coupling                       | Visibility of the whole flow        |
| Orchestration                  | Clear, central control                | A central dependency                |

## Interview Questions

**Q: What is the difference between an event and a command?**
An event states a fact that already happened and may have many consumers. A command requests an action and goes to a single handler.

**Q: How do you get exactly-once processing?**
Usually you do not get true exactly-once delivery. You use at-least-once delivery with idempotent consumers (deduplicating by event ID or using naturally idempotent operations), which produces exactly-once _effects_.

**Q: What is the dual-write problem and how do you solve it?**
Updating a database and publishing an event are two separate operations that can partly fail. The transactional outbox pattern writes the event into an outbox table in the same transaction as the data, and a relay publishes it afterward.

**Q: How do you preserve ordering in an event-driven system?**
Brokers order events only within a partition, so use a partition key (such as the entity ID) so that all events for the same entity are processed in order.

**Q: Choreography or orchestration?**
Choreography for simple flows with few steps and maximum decoupling. Orchestration for complex flows that need visibility, branching, and compensation.

**Q: How do you change an event schema without breaking consumers?**
Add only optional fields, never remove or repurpose existing ones, use a schema registry to enforce compatibility, and publish a new version for breaking changes.

## Summary

- Services publish events (facts) and consumers react, so producers and consumers are decoupled.
- Use at-least-once delivery and idempotent consumers; true exactly-once is rarely practical.
- Ordering is guaranteed only within a partition, so choose partition keys carefully.
- The outbox pattern makes "update the database and publish an event" safe.
- Treat events as long-lived contracts and evolve schemas compatibly.
- Handle failures with retries, dead-letter queues, and lag monitoring, and invest in tracing.

## Continue Learning

- [Messaging and Communication](system-design/messaging-and-communication) — the building blocks of event delivery.
- [CQRS and Event Sourcing](system-design/cqrs-and-event-sourcing) — events as the source of truth.
- [Distributed Systems Essentials](system-design/distributed-systems-essentials) — sagas for multi-service workflows.
