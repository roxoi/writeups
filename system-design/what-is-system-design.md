---
title: "What is System Design?"
description: "A comprehensive introduction to system design — the process of defining architecture, interfaces, and data flows to build scalable, reliable, and efficient software systems."
author: ["name": "Rajendra Pancholi", "email": "rpancholi522@gmail.com"]
created: "2024-05-20"
updated: "2026-04-18"
thumbnail: "/images/system-design-intro.png"
tags: [system-design, architecture, software-engineering, fundamentals]
keywords: ["What is system design", "System design introduction", "Software architecture basics", "How to design scalable systems"]
---

# What is System Design?

System design is the process of defining the **architecture, interfaces, and data** for a system that satisfies specific requirements. It meets the needs of your business or organization through coherent and efficient systems, requiring a systematic approach to building and engineering software at scale.

A good system design forces you to think about everything, from infrastructure all the way down to how data is stored, transferred, and processed.

A useful analogy is constructing a building. Before anyone lays bricks, an architect decides where the load-bearing walls go, how plumbing and wiring run, and how many people the building must hold. Those decisions are cheap to change on paper and extremely expensive to change after the walls are up. System design is the same exercise for software.

![What is System Design](/images/system-design-intro.png)

## A Quick Example

Suppose you are asked to build a photo-sharing app. A working prototype is easy: one server, one database, photos saved on disk. It works perfectly for you and ten friends.

System design is about the questions that prototype ignores:

- What happens when 10 million people upload photos every day?
- Where do the photo files live, and how do they load quickly for a user in another country?
- What happens when the database server crashes at 2 a.m.?
- How fast must the feed load, and what are we willing to trade to achieve that?
- How do ten engineers work on this system without breaking each other's work?

Answering these questions _before_ building, and being able to justify each answer, is what system design is.

## Why is System Design Important?

System design helps define a solution that meets business requirements. It is one of the **earliest and most consequential decisions** made when building a system. These high-level decisions are notoriously difficult to correct later, making upfront architectural thinking essential.

Key reasons system design matters:

- **Translates business requirements into technical solutions**: forces early alignment between product goals and engineering constraints.
- **Enables reasoning about architectural changes**: a well-designed system can evolve without complete rewrites.
- **Prevents scaling disasters**: poor design that works at 1,000 users may catastrophically fail at 1,000,000.
- **Reduces technical debt**: structural decisions made upfront prevent accumulated shortcuts that become liabilities.
- **Controls cost**: knowing where the bottlenecks are means you spend on the right resources instead of over-provisioning everything.
- **Facilitates team collaboration**: a documented architecture creates shared understanding across engineering teams.

## Core Pillars of System Design

Every system design exercise revolves around balancing these foundational concerns:

### 1. Scalability

The ability of a system to handle growing amounts of work by adding resources. This includes both **vertical scaling** (adding more power to existing machines) and **horizontal scaling** (adding more machines).

Scalability is not the same as raw speed. A system is scalable when performance and cost stay predictable as load grows, for example when doubling the servers roughly doubles the capacity. Whatever limits that growth (a single database, a shared lock, a non-stateless server) is the bottleneck you are designing around.

Learn more in [Vertical vs Horizontal Scaling](system-design/scaling) and [Load Balancing](system-design/load-balancing).

### 2. Reliability

A system is reliable when it continues to work correctly even in the face of hardware faults, software bugs, or human error.

Two terms are worth separating:

- A **fault** is one component deviating from its specification (a disk dies, a service returns an error).
- A **failure** is the system as a whole no longer providing its service to users.

You cannot prevent all faults, so the goal is to build **fault-tolerant** systems where faults do not become failures. Reliability is commonly measured with **Mean Time Between Failures (MTBF)**, the average time a system runs before failing, and **Mean Time To Repair (MTTR)**, the average time it takes to recover.

Note that reliability is about _correctness_, not just uptime. A system that is up but returns wrong data is available, yet not reliable.

### 3. Availability

Availability is the percentage of time a system is operational. It is expressed in "nines". For example, 99.9% availability (three nines) allows roughly 8.77 hours of downtime per year.

| Availability         | Downtime per Year |
| -------------------- | ----------------- |
| 99% (two nines)      | 3.65 days         |
| 99.9% (three nines)  | 8.77 hours        |
| 99.99% (four nines)  | 52.6 minutes      |
| 99.999% (five nines) | 5.26 minutes      |

> Figures assume an average year of 365.25 days.

Availability ties directly to the two measures above:

```text
Availability = MTBF / (MTBF + MTTR)
```

This shows two ways to improve it: fail less often (raise MTBF) or recover faster (lower MTTR). In practice, fast automated recovery is usually cheaper than trying to never fail.

**Combining components changes the math.**

- **In series** (a request needs _all_ of them), availabilities multiply. Two services at 99.9% each give 0.999 × 0.999 ≈ **99.8%**, about 17.5 hours of downtime per year instead of 8.77. Every extra dependency in the request path lowers your availability.
- **In parallel** (redundant copies, any one can serve), the chance of _all_ failing is small. Two replicas at 99% each give 1 − (0.01 × 0.01) = **99.99%**. This assumes failures are independent. If both replicas share a rack, power supply, or region, the real number is lower.

Common ways to improve availability include [load balancing](system-design/load-balancing) with health checks and failover, and [replicating data](system-design/databases) across nodes. See [High Availability](system-design/high-availability) for the full picture.

### 4. Performance

Measured primarily by **latency** (time to complete a single request) and **throughput** (number of requests processed per unit time). These are often in tension with one another. For example, batching requests can raise throughput while increasing the latency of each individual request.

Averages hide problems, so latency is usually reported in **percentiles**. The p99 latency is the value that 99% of requests are faster than. If your average is 100 ms but your p99 is 3 seconds, one in a hundred users is having a very bad experience, and those slow requests add up quickly for a user who makes many calls per page.

[Caching](system-design/caching-and-cdn) and CDNs are among the most effective tools for reducing latency. See [Latency vs Throughput](system-design/cheat-sheet) for more.

### 5. Maintainability

How easily can the system be understood, modified, and operated over time? Maintainability breaks down into three parts:

- **Operability**: making it easy for operations teams to keep the system healthy (good monitoring, safe deployments).
- **Simplicity**: managing complexity with clean abstractions so new engineers can understand the system.
- **Evolvability**: making it easy to change the system as requirements change.

### Other Qualities You Will Meet

Beyond the five pillars, most designs also have to weigh:

- **Consistency**: whether every reader sees the latest write (see [CAP Theorem](system-design/databases) and Consistency Models).
- **Security**: protecting data and access (see [Security Best Practices](system-design/auth-and-security)).
- **Cost**: infrastructure and engineering time are both finite.

## The System Design Process

A structured approach to [system design interviews](system-design/system-design-interview-guide) and real-world architecture work follows these stages:

1. **Requirements Clarification**: understand functional requirements (what the system must do) and non-functional requirements (how well it must do it: latency, availability, consistency). See [Functional vs Non-Functional Requirements](system-design/system-design-interview-guide).
2. **Estimation and Constraints**: back-of-the-envelope calculations for traffic (requests per second, RPS), storage needs, and bandwidth help scope the solution. See [Capacity Estimation](system-design/cheat-sheet).
3. **Data Model Design**: define entities and relationships, and choose between SQL and NoSQL databases (see [SQL vs NoSQL](system-design/databases)).
4. **API Design**: define the interface contracts between services and clients (see [REST API Design](system-design/messaging-and-communication)).
5. **High-Level Component Design**: identify major components: [DNS](system-design/dns), [load balancers](system-design/load-balancing), [databases](system-design/databases), [caches and CDNs](system-design/caching-and-cdn), and [message queues](system-design/messaging-and-communication).
6. **Detailed Design**: deep dive into critical components: partitioning strategy, caching policy, replication model.
7. **Identify and Resolve Bottlenecks**: single points of failure, hot spots, and latency sources must be addressed.

**A tiny estimation example.** Say your product has 1 million daily active users, each making about 10 requests a day. That is 10 million requests per day. A day has about 86,400 seconds, so the average load is roughly **115 requests per second**. Traffic is rarely flat, so a common rule of thumb is to plan for peaks of 2 to 3 times the average (more for spiky products like ticket sales), which is roughly 230 to 350 RPS. A number like this immediately tells you whether one server is enough or whether you need to scale out.

The process is not strictly linear. Estimation often forces you to revisit requirements, and a bottleneck found in step 7 may change the data model from step 3. That is normal.

## High-Level Design vs Low-Level Design

You will see these two terms constantly, especially in interviews:

|                       | High-Level Design (HLD)                                          | Low-Level Design (LLD)                                                       |
| --------------------- | ---------------------------------------------------------------- | ---------------------------------------------------------------------------- |
| **Focus**             | The whole system and how its components fit together             | The internals of one component                                               |
| **Typical questions** | Which database? Where do we add a cache? How do we scale writes? | What classes and interfaces? Which design pattern? What is the table schema? |
| **Example problems**  | Design a URL shortener, Design WhatsApp                          | Design a parking lot, Design an LRU cache                                    |
| **Output**            | Architecture diagram, data flow, technology choices              | Class diagrams, interfaces, schemas, code structure                          |

This series focuses on **high-level design**. SDE interviews commonly include both, so treat low-level design as a separate skill to practice alongside it.

## How a System Evolves

Real systems are not designed at full scale on day one. They grow in stages, and each stage exists to solve a specific bottleneck. Seeing this progression is the best way to understand why each building block exists.

| Stage                             | Bottleneck you hit                                                 | What you add                                                     | Read more                                                                        |
| --------------------------------- | ------------------------------------------------------------------ | ---------------------------------------------------------------- | -------------------------------------------------------------------------------- |
| 1. Single server                  | Everything shares one machine; if it dies, everything dies         | Start simple: web app, database, and files on one box            | [Client-Server Architecture](system-design/networking-fundamentals)              |
| 2. Separate web and data tiers    | App and database compete for CPU and memory                        | Move the database to its own server so each scales independently | [SQL vs NoSQL](system-design/databases)                                          |
| 3. Horizontal scaling             | One app server cannot handle the traffic                           | Run many stateless app servers behind a load balancer            | [Scaling](system-design/scaling), [Load Balancing](system-design/load-balancing) |
| 4. Caching and CDN                | Database is overloaded; users far away see slow pages              | Cache hot data in memory, serve static content from the edge     | [Caching](system-design/caching-and-cdn), CDN                                    |
| 5. Database replication           | One database is a read bottleneck and a single point of failure    | Add read replicas and automatic failover                         | [Replication](system-design/databases)                                           |
| 6. Sharding                       | Data or write volume exceeds what one database can hold            | Split data across multiple databases                             | [Sharding](system-design/databases)                                              |
| 7. Asynchronous processing        | Slow work (emails, video encoding) blocks user requests            | Move it to background workers via a queue                        | [Message Queues](system-design/messaging-and-communication)                      |
| 8. Service decomposition          | A large codebase and team slow each other down                     | Split into independently deployable services                     | [Microservices](system-design/microservices)                                     |
| 9. Multi-region and observability | A regional outage takes everything down; problems are hard to find | Deploy across regions, add metrics, logs, and traces             | [Disaster Recovery](system-design/observability-and-deployment), Observability   |

Every stage adds complexity as well as capacity. A system with 100 users does not need sharding or microservices, and adding them early costs real engineering time for no benefit. The skill is knowing _which bottleneck you have right now_ and picking the simplest tool that fixes it.

## Trade-offs: Nothing is Free

There is no universally best design, only designs that fit specific requirements. Almost every decision trades one quality for another:

| Trade-off                   | What you gain                                            | What you give up                                          |
| --------------------------- | -------------------------------------------------------- | --------------------------------------------------------- |
| Consistency vs availability | Every read sees the latest write                         | The system may refuse requests during a network partition |
| Latency vs throughput       | Batching processes more work per second                  | Each individual request waits longer                      |
| Performance vs cost         | More replicas, caches, and faster hardware               | A larger infrastructure bill                              |
| Simplicity vs flexibility   | A monolith is easy to build and debug                    | Harder to scale teams and components independently        |
| Read speed vs write speed   | Indexes and denormalized data make reads fast            | Writes are slower and use more storage                    |
| Durability vs write latency | Waiting for data to reach disk or replicas keeps it safe | Each write takes longer to acknowledge                    |

A strong design answer always names the trade-off being made and why it is acceptable for _this_ system's requirements.

## What Good System Design Looks Like

A well-designed system has the following characteristics:

- **Loose coupling** between components: changes to one part do not cascade failures to others.
- **High cohesion** within services: each service has a clear, focused responsibility.
- **Layered resilience**: multiple layers of protection such as redundancy, timeouts, rate limiting, and circuit breakers, so one failing layer does not take everything down.
- **Observability**: logs, metrics, and distributed tracing are built in from day one.
- **Graceful degradation**: the system continues to serve reduced functionality rather than failing completely under stress.
- **Appropriate simplicity**: the simplest design that meets the requirements, with a clear path to grow when they change.

## System Design vs. Software Architecture

These terms overlap and are often used interchangeably, but they emphasize different things:

- **Software Architecture** defines the fundamental structure of a software system: its major components, how they relate to one another, and the principles and patterns (layered, microservices, event-driven, and so on) that guide how it evolves. It is concerned with the quality attributes, such as scalability, security, and maintainability, that those structural choices must satisfy.
- **System Design** is the broader process of turning requirements into a concrete, working design: choosing components and infrastructure, defining data models and APIs, estimating capacity, and making the trade-offs needed to serve users at scale.

Architecture sets the structural vision and constraints; system design fills it in with concrete components, data flows, and trade-offs. In practice, engineers move between both. For a deeper look at architectural patterns, see [Monolithic Architecture](system-design/monolith), [Microservices](system-design/microservices), and [Event-Driven Architecture](system-design/event-driven).

## Common Mistakes

- **Jumping to technology before requirements.** Saying "we'll use Kafka and Cassandra" before knowing the read/write pattern and scale is the most common mistake.
- **Over-engineering.** Microservices, sharding, and multi-region setups for a system with a few thousand users add cost and failure modes without solving a real problem.
- **Skipping estimation.** Without rough numbers you cannot tell whether a design is overkill or will collapse under load.
- **Ignoring failure.** Every network call can fail or be slow. Designs with single points of failure, no timeouts, and no retry strategy only work on good days.
- **Treating tools as silver bullets.** Every database, cache, and queue has trade-offs. If you cannot say what a choice costs you, you do not yet understand it.
- **Designing without looking at the data.** Access patterns (what is read, how often, how big) should drive the choice of storage, not habit.

## Interview Questions

**Q: What is system design, and why does it matter?**
It is the process of defining a system's architecture, components, data models, and interfaces so that it meets its functional and non-functional requirements. It matters because these are early, high-impact decisions that are expensive to change later.

**Q: What is the difference between scalability and performance?**
Performance is how fast the system responds for a given load. Scalability is how well that performance (and cost) holds up as the load grows. A system can be fast for 100 users and still not be scalable.

**Q: What is the difference between reliability and availability?**
Availability is the fraction of time the system is up and responding. Reliability is whether it behaves _correctly_ while running. A system that is up but returns corrupted data is available but not reliable.

**Q: Two services, each with 99.9% availability, are both required to serve a request. What is the combined availability?**
About 99.8%, because availabilities in series multiply (0.999 × 0.999 ≈ 0.998). That is roughly 17.5 hours of downtime a year instead of 8.77.

**Q: What is the difference between high-level and low-level design?**
High-level design covers the overall architecture: components, data flow, and technology choices. Low-level design covers the internals of a component: classes, interfaces, schemas, and design patterns.

**Q: Your system handles 100 RPS today and is expected to handle 10 times that in six months. What do you do first?**
Measure first to find the actual bottleneck instead of guessing (CPU, database, network, a slow dependency). Then apply the simplest fix: make the app tier stateless and scale it horizontally behind a load balancer, add caching for hot reads, and add read replicas if the database is the limit. Only consider sharding or service splits if simpler steps are not enough.

## Summary

- System design turns requirements into an architecture of components, data models, and interfaces, and it is hard to change later, so it pays to think early.
- The core pillars are scalability, reliability, availability, performance, and maintainability, and they often pull against each other.
- Availability is `MTBF / (MTBF + MTTR)`. Dependencies in series reduce it; redundancy in parallel raises it.
- Systems grow in stages. Add each building block only when a specific bottleneck demands it.
- There is no perfect design, only trade-offs. A good answer names them and justifies them against the requirements.

System design is both a discipline and a skill. It requires balancing competing constraints (cost vs. performance, consistency vs. availability, simplicity vs. flexibility) with no universally correct answer. The goal is always to build a system that meets the specific needs of its users and can evolve gracefully as those needs change.

The remaining guides in this series cover each building block of system design in depth, followed by real-world case studies. Use the roadmap below to continue.

## Continue Learning

### Start Here

- [System Design Interview Guide](system-design/system-design-interview-guide) — a step-by-step framework for answering design questions.
- [System Design Cheat Sheet](system-design/cheat-sheet) — latency numbers, formulas, trade-offs, and practice questions.

### Fundamentals

- [Networking Fundamentals](system-design/networking-fundamentals) — how data moves between clients and servers.
- [DNS](system-design/dns) — how domain names are resolved to IP addresses.
- [Scaling](system-design/scaling) — vertical vs horizontal scaling, stateless services, and autoscaling.
- [Load Balancing](system-design/load-balancing) — distributing traffic across servers.
- [Caching and CDN](system-design/caching-and-cdn) — serving data faster and closer to users.
- [Databases](system-design/databases) — storing, indexing, replicating, and sharding data.
- [Messaging and Communication](system-design/messaging-and-communication) — queues, Kafka, and APIs.

### Architecture and Reliability

- [Microservices Architecture](system-design/microservices) and [Monolithic Architecture](system-design/monolith) — structuring systems.
- [Event-Driven Architecture](system-design/event-driven) and [CQRS and Event Sourcing](system-design/cqrs-and-event-sourcing) — decoupling with events.
- [High Availability](system-design/high-availability), [Circuit Breaker Pattern](system-design/circuit-breaker), and [Rate Limiting](system-design/rate-limiting) — designing for failure.
- [Distributed Systems Essentials](system-design/distributed-systems-essentials) — sagas, consensus, locks, clocks, and IDs.
- [Observability and Deployment](system-design/observability-and-deployment) and [Authentication and Security](system-design/auth-and-security) — running and protecting systems.

### Case Studies

- [Design a URL Shortener](system-design/url-shortener-design)
- [Design a Rate Limiter](system-design/rate-limiter-design)
- [Design WhatsApp](system-design/whatsapp-design)
- [Design Twitter](system-design/twitter-design)
- [Design a Notification System](system-design/notification-system-design)
- [Design Search Autocomplete](system-design/typeahead-design)
- [Design a Web Crawler](system-design/web-crawler-design)
- [Design a File Storage Service](system-design/file-storage-design)
- [Design a Ticket Booking System](system-design/ticket-booking-design)
- [Design Uber](system-design/uber-design)
- [Design Netflix](system-design/netflix-design)
