---
title: "Microservices Architecture"
description: "A practical guide to microservices: service boundaries, data ownership, communication, resilience, observability, the distributed monolith trap, and when not to use them."
author: ["name": "Rajendra Pancholi", "email": "rpancholi522@gmail.com"]
created: "2026-04-18"
updated: "2026-10-17"
thumbnail: "/images/microservices.png"
tags: [microservices, architecture, distributed-systems, domain-driven-design, system-design]
keywords: ["Microservices architecture", "Microservices vs monolith", "Distributed monolith", "Service boundaries", "Database per service", "Microservices best practices"]
---

# Microservices Architecture

Microservices decompose an application into a collection of **small, independent, single-purpose services**. Each service owns its own data, is deployed independently, and communicates with others over well-defined APIs.

![Microservices Architecture](/images/microservices.png)

## Characteristics

- **Loosely coupled:** services interact only through APIs; internal implementations are hidden.
- **Single responsibility:** each service implements one business capability.
- **Independent deployability:** a service can be built, deployed, and scaled without touching the others.
- **Data ownership:** each service owns its data. Other services access it only through the owner's API, never by reading its tables directly.
- **Technology heterogeneity:** different services can use different languages, frameworks, and databases.
- **Organized around business capabilities,** often with a small team owning each service end to end.

## Advantages

- **Independent scaling:** scale only the bottleneck service.
- **Technology flexibility:** use the right tool for each job.
- **Fault isolation:** a failure in one service does not necessarily crash others (with proper resilience patterns).
- **Parallel development:** several teams work and release at the same time without a shared release train.
- **Smaller, faster deployments** for individual services.

## Disadvantages

- **Distributed-system complexity:** network latency, partial failures, timeouts, and service discovery.
- **Harder data consistency:** no ACID transactions across services (see [Distributed Transactions](system-design/distributed-systems-essentials)).
- **Harder testing:** integration across services is difficult to test.
- **Operational overhead:** many deployments, many pipelines, monitoring and log aggregation for each.
- **Cost of mistakes:** wrong service boundaries are expensive to fix.

Sometimes called the **microservices premium**: you pay a large, permanent complexity cost up front, and it only pays off when the system or organization is big enough to benefit.

## Defining Service Boundaries

Getting boundaries right is the hardest part. Good boundaries come from the **business domain**, not from technical layers.

**Domain-Driven Design (DDD)** gives a useful vocabulary:

- A **bounded context** is a part of the domain where a term has one consistent meaning. "Customer" means something different to Billing (an account that pays) and to Support (a person with tickets), so they belong to different contexts, and probably different services.
- Aim for **high cohesion** inside a service and **low coupling** between services.

Guidelines:

- A service should be small enough for one team to own and understand, but not so small that every operation requires a chain of calls (this "nano-service" extreme is a common mistake).
- If two services always change together, or one cannot work without the other, they probably should be one service.
- **Conway's Law:** systems tend to mirror the communication structure of the organization that builds them. Align services with team ownership.

## Data Management

### Database per Service

Each service owns its tables or database. It may share a database _server_ for cost reasons, but never shares tables or schemas. If Service B reads Service A's tables directly, they are coupled at the data level and cannot change independently.

This means you lose cross-service joins and transactions, so you need other techniques:

| Need                                       | Technique                                                                                                                                                   |
| ------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Query data from several services           | **API composition** (an aggregator calls each service), or a **read model** built from events (see [CQRS](system-design/cqrs-and-event-sourcing))           |
| Keep data consistent across services       | **Saga** pattern: a sequence of local transactions with compensating actions (see [Distributed Transactions](system-design/distributed-systems-essentials)) |
| Reliably publish an event after a DB write | **Transactional outbox** (see [Event-Driven Architecture](system-design/event-driven))                                                                      |

## Communication Between Services

| Style            | Examples                      | Use for                                                  | Watch out for                                            |
| ---------------- | ----------------------------- | -------------------------------------------------------- | -------------------------------------------------------- |
| **Synchronous**  | REST, gRPC                    | Queries and requests that need an immediate answer       | Chains of calls multiply latency and reduce availability |
| **Asynchronous** | Message queues, event streams | Notifications, workflows, anything that can happen later | Eventual consistency, harder debugging                   |

**Availability cost of synchronous chains:** if a request needs five services in series, each at 99.9%, the whole request is only about 99.5% available. Prefer asynchronous communication and caching for non-critical dependencies, and keep synchronous call chains short.

Related building blocks:

- [API Gateway](system-design/api-gateway) as the client-facing entry point.
- [Service Discovery and Service Mesh](system-design/messaging-and-communication) so services can find each other and communicate securely.
- [REST](system-design/messaging-and-communication) and gRPC for service APIs.

## Resilience

In a distributed system, every call can fail or be slow. Design every service call defensively:

- **Timeouts** on every remote call. A call without a timeout can hold resources forever.
- **Retries with exponential backoff and jitter,** only for safe (idempotent) operations (see [Idempotency and Retries](system-design/messaging-and-communication)).
- **[Circuit breakers](system-design/circuit-breaker)** to stop calling a failing service.
- **Bulkheads** to isolate resources (thread pools, connection pools) per dependency, so one slow dependency cannot exhaust everything.
- **Fallbacks and graceful degradation** when a non-critical dependency is down.
- **[Rate limiting](system-design/rate-limiting)** to protect services from overload.

See [Fault Tolerance](system-design/high-availability) for how these fit together.

## Observability and Testing

With many services, debugging "why was this request slow?" is impossible without tooling:

- **Centralized logging** with a **correlation ID** passed through every call.
- **Metrics** per service (latency, error rate, saturation).
- **Distributed tracing** (Jaeger, Zipkin, OpenTelemetry) to follow one request across services.

See [Observability](system-design/observability-and-deployment).

For testing, rely on many fast unit tests per service, plus **contract tests** (such as consumer-driven contracts) to verify that the API a service provides matches what its callers expect, instead of relying only on slow end-to-end tests.

## Beware: The Distributed Monolith

A distributed monolith has the _structure_ of microservices but the _coupling_ of a monolith. It carries the cost of a distributed system with none of the benefits.

**Signs you have built one:**

- Services cannot be deployed independently.
- Services share a database or data model.
- A change to Service A forces a change in Service B.
- Services call each other synchronously in long chains.
- Many services must be released together in a specific order.

If you see these, the fix is usually to merge tightly coupled services, or redraw boundaries around business capabilities.

## When to Use Microservices (and When Not To)

**Good fit:**

- Many teams that need to deploy independently.
- Parts of the system with very different scaling or technology needs.
- A well-understood domain with stable boundaries.
- A team with strong DevOps maturity: CI/CD, monitoring, and on-call.

**Poor fit:**

- A small team or an early-stage product.
- An unclear domain where boundaries would be guesses.
- No automation for deployment and monitoring.

If you are unsure, start with a [modular monolith](system-design/monolith) and extract services when a concrete problem justifies it. To migrate an existing monolith, use the strangler fig approach described there.

## Trade-offs

| Choice                     | You gain                                  | You give up                            |
| -------------------------- | ----------------------------------------- | -------------------------------------- |
| Split into services        | Independent scaling and deployment        | Simplicity and easy transactions       |
| Database per service       | Loose coupling, freedom to choose storage | Joins and ACID across services         |
| Asynchronous communication | Resilience and decoupling                 | Eventual consistency, debugging effort |
| Many small services        | Team autonomy                             | Operational overhead                   |

## Real-World Usage

- **E-commerce:** separate services for catalog, cart, orders, payments, inventory, and notifications. Placing an order triggers a workflow across them (usually event-driven with a saga).
- **Streaming and ride-hailing platforms** publicly describe hundreds or thousands of services, which was only practical because they invested heavily in tooling, platform teams, and observability.

## Interview Questions

**Q: What are microservices, and how do they differ from a monolith?**
Small, independently deployable services, each owning its data and communicating over APIs, versus one deployable unit. Microservices trade simplicity for independent scaling, deployment, and team autonomy.

**Q: How do you handle a transaction that spans multiple services?**
Not with a distributed ACID transaction. Use a saga: a series of local transactions, each with a compensating action to undo it if a later step fails, usually coordinated by events or an orchestrator.

**Q: How do you decide service boundaries?**
Use business capabilities and bounded contexts from Domain-Driven Design. Keep high cohesion inside a service, minimize chatty cross-service calls, and align with team ownership.

**Q: What is a distributed monolith?**
Services that are separately deployed but tightly coupled, such as shared databases, lock-step releases, or long synchronous chains. It has the downsides of both architectures.

**Q: How do you debug a slow request that crosses ten services?**
Use distributed tracing with a correlation ID propagated through every call, correlate with per-service metrics and logs, and look for the slowest span.

**Q: When would you not use microservices?**
With a small team, an early product, an unclear domain, or weak DevOps automation. A modular monolith is often a better start.

## Summary

- Microservices are small, independently deployable services that own their data.
- Their benefits (independent scaling, deployment, and team autonomy) come with real costs: distributed complexity, eventual consistency, and operational overhead.
- Draw boundaries from business capabilities, not technical layers.
- Never share databases; use API composition, events, and sagas instead.
- Every remote call needs timeouts, retries, circuit breakers, and observability.
- Avoid the distributed monolith. Start modular and split only when you must.

## Continue Learning

- [Monolithic Architecture](system-design/monolith) — the alternative, and the starting point.
- [Event-Driven Architecture](system-design/event-driven) — decoupling services with events.
- [Distributed Systems Essentials](system-design/distributed-systems-essentials) — sagas and the outbox pattern.
- [Messaging and Communication](system-design/messaging-and-communication) — service discovery, queues, REST and gRPC between services.
- [Circuit Breaker Pattern](system-design/circuit-breaker) — containing failures between services.
