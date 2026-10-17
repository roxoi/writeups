---
title: "Message Queues, Kafka, Pub/Sub, Real-Time Communication, and REST vs gRPC — Communication Patterns in System Design"
description: "A practical guide to communication patterns: message queues, delivery guarantees and idempotency, publish-subscribe, Apache Kafka, long polling, WebSockets, SSE, and REST vs GraphQL vs gRPC."
author: ["name": "Rajendra Pancholi", "email": "rpancholi522@gmail.com"]
created: "2026-04-18"
updated: "2026-10-17"
thumbnail: "/images/message-broker.png"
tags: [message-queue, kafka, pub-sub, websockets, rest, grpc, graphql, idempotency, system-design, communication]
keywords: ["Message queue vs pub-sub", "Apache Kafka system design", "Kafka partitions and consumer groups", "Exactly-once delivery idempotency", "WebSockets vs SSE", "REST vs gRPC comparison", "RabbitMQ vs Kafka", "Long polling explained"]
---

# Message Queues, Kafka, Pub/Sub, Real-Time Communication, and REST vs gRPC — Communication Patterns in System Design

How services communicate is one of the most consequential decisions in system design. The wrong communication pattern creates tight coupling, bottlenecks, and cascading failures. The right one enables loose coupling, scalability, and resilience.

This guide covers the full spectrum — from synchronous request-response APIs to asynchronous message-driven architectures, including how Kafka works and how to handle duplicates safely.

![Message Broker](/images/message-broker.png)

## Synchronous vs Asynchronous Communication

| Type             | Description                                                           | When to Use                                                      |
| ---------------- | --------------------------------------------------------------------- | ---------------------------------------------------------------- |
| **Synchronous**  | Caller waits for a response before proceeding (HTTP, gRPC)            | When the result is needed immediately; user-facing operations    |
| **Asynchronous** | Caller sends a message and continues without waiting (queues, events) | Background processing, decoupling services, handling load spikes |

Synchronous chains reduce availability (every service in the chain must be up) and add latency. Asynchronous messaging decouples services in time but gives you eventual consistency and harder debugging. Most real systems use **both**: synchronous calls for queries and immediate answers, messages for everything that can happen afterward. See [Microservices](system-design/microservices) and [Event-Driven Architecture](system-design/event-driven).

## Message Queues

A message queue provides **asynchronous point-to-point communication** between services. Producers send messages to a queue; consumers read messages from it.

![Message Queue](/images/message-queue.png)

### How It Works

1. A **producer** sends a message to the queue (including the job payload and metadata).
2. The message is **stored in the queue** until a consumer is available.
3. A **consumer** receives the message, processes it, and **acknowledges** it. Only then is it removed from the queue.
4. Each message is typically processed by **one consumer** (point-to-point), so multiple consumers share the work.

If the consumer crashes before acknowledging, the message becomes available again and is redelivered. In Amazon SQS this is controlled by the **visibility timeout**: while one consumer is working, the message is hidden from others for that period; if it isn't deleted in time, it reappears. Set the timeout longer than your processing time, or the same message will be processed twice.

### Key Features

| Feature                          | Description                                                                                                                                     |
| -------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| **At-Least-Once Delivery**       | Message is redelivered until acknowledged; **duplicates are possible**                                                                          |
| **FIFO Ordering**                | Messages are delivered in order, usually **per key or message group**, not globally                                                             |
| **Deduplication**                | Some queues (e.g., SQS FIFO) drop repeated messages with the same ID inside a time window; this helps but does not replace idempotent consumers |
| **Dead-Letter Queue (DLQ)**      | Messages that fail after N retries are moved aside for inspection and replay                                                                    |
| **Delayed / Scheduled Delivery** | Messages stay invisible for a set time before delivery                                                                                          |
| **Backpressure**                 | Producers must be slowed or limited when consumers can't keep up (see below)                                                                    |

### Backpressure and Load

Backpressure means **not letting a fast producer overwhelm a slow consumer**. Queues absorb bursts, but they only delay the problem if consumers are permanently slower.

- Monitor **queue depth** and **message age**, and alert when they grow.
- **Autoscale consumers** based on queue depth.
- Depending on the broker, a full queue may reject publishes, block the producer, or drop old messages; some managed queues are effectively unbounded. Know your broker's behavior.
- Apply [rate limiting](system-design/rate-limiting) on producers if needed.

### Advantages

- **Decoupling** — producers and consumers are independent; neither needs to know about the other's availability
- **Resilience** — if the consumer crashes, messages wait in the queue until it recovers
- **Load leveling** — the queue buffers traffic spikes; consumers process at their own pace
- **Scalability** — add more consumer instances to increase throughput

### Popular Message Queue Solutions

| Solution                                 | Notes                                                                        |
| ---------------------------------------- | ---------------------------------------------------------------------------- |
| [Amazon SQS](https://aws.amazon.com/sqs) | Managed; standard (at-least-once, best-effort ordering) and FIFO queues      |
| [RabbitMQ](https://www.rabbitmq.com)     | Open-source; flexible routing through exchanges; supports multiple protocols |
| [ActiveMQ](https://activemq.apache.org)  | JMS-based; enterprise feature set                                            |
| [ZeroMQ](https://zeromq.org)             | Lightweight embedded messaging library (not a broker)                        |

## Delivery Guarantees and Idempotency

| Guarantee         | Meaning                                | Problem                     |
| ----------------- | -------------------------------------- | --------------------------- |
| **At-most-once**  | Delivered zero or one time; no retries | Messages can be **lost**    |
| **At-least-once** | Retried until acknowledged             | **Duplicates** are possible |
| **Exactly-once**  | Processed exactly one time             | Very hard end to end        |

**The practical truth:** networks fail between "processed" and "acknowledged," so a broker can't know whether the consumer finished. Real systems run **at-least-once delivery** and make the consumer **idempotent** — processing the same message twice has the same effect as processing it once. That gives "exactly-once _effect_," even though delivery itself is at-least-once.

### Making a Consumer Idempotent

- **Deduplication table:** store each processed message ID with a unique constraint, in the **same database transaction** as the side effect. A duplicate hits the constraint and is skipped.
- **Naturally idempotent operations:** "set status = SHIPPED" is safe to repeat; "add 10 to balance" is not.
- **Idempotency keys for API calls:** the client sends a unique key (e.g., payment attempt ID); the server remembers the result and returns it again instead of repeating the action.
- **Conditional writes:** update only if a version or state matches (`WHERE status = 'PENDING'`).

### Retries, Poison Messages, and Ordering

- Retry transient failures with **exponential backoff and jitter** (random delay) so retries don't arrive in a synchronized wave.
- A **poison message** (one that can never succeed) must be moved to a DLQ after a few attempts, or it will block the queue or partition.
- Ordering costs parallelism: strict order means one consumer per ordering key. Order **per entity** (e.g., per order ID), not globally.

The outbox pattern, which makes "save to the database and publish an event" safe, is covered in [Event-Driven Architecture](system-design/event-driven).

## Publish-Subscribe (Pub/Sub)

In pub/sub, a **publisher sends messages to a topic**; all subscribers to that topic receive the message. Unlike point-to-point queues, pub/sub enables **fan-out** — one message reaching many consumers.

![Publish-Subscribe](/images/publish-subscribe.png)

### How It Works

1. Publishers send messages to a **topic** (a named channel).
2. Subscribers register interest in one or more topics.
3. The broker **delivers the message to all subscribers**; each subscriber gets its own copy and processes it independently.

### Pub/Sub vs Message Queue

| Aspect             | Message Queue                     | Pub/Sub                               |
| ------------------ | --------------------------------- | ------------------------------------- |
| Delivery           | One message → one consumer        | One message → many consumers          |
| Relationship       | Point-to-point                    | Fan-out                               |
| Consumer awareness | Producers target a specific queue | Publishers are unaware of subscribers |
| Use case           | Task distribution, job queues     | Event notifications, real-time feeds  |

A common production pattern combines both: publish an event to a topic (e.g., Amazon SNS), and **each subscribing service has its own queue** (SQS) so it can process at its own pace with its own retries and DLQ.

### Features

- **Filtering** — subscribers receive only a subset of messages (by attributes, content, type)
- **Durability** — messages are stored on multiple servers to prevent loss
- **Fan-out** — one event triggers parallel processing by multiple independent services

### Popular Pub/Sub Solutions

| Solution                                                | Notes                                                                                  |
| ------------------------------------------------------- | -------------------------------------------------------------------------------------- |
| [Apache Kafka](https://kafka.apache.org)                | Distributed log; very high throughput; messages retained and replayable (next section) |
| [Google Cloud Pub/Sub](https://cloud.google.com/pubsub) | Managed; at-least-once by default; push and pull modes                                 |
| [Amazon SNS](https://aws.amazon.com/sns)                | Managed; pushes to SQS, Lambda, HTTP, email, SMS                                       |
| [NATS](https://nats.io)                                 | Lightweight, very fast, cloud-native                                                   |
| [RabbitMQ](https://www.rabbitmq.com)                    | Supports queues and pub/sub through exchanges                                          |

## Apache Kafka

Kafka is a **distributed, durable, append-only log**. Unlike a traditional queue, which deletes a message once it's consumed, Kafka **keeps messages** for a configured time or size, and each consumer tracks its own position. That makes it a streaming platform: many independent consumers, replay, and very high throughput.

### Core Concepts

| Concept              | Meaning                                                                                                                                           |
| -------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Topic**            | A named stream of messages (e.g., `orders`)                                                                                                       |
| **Partition**        | A topic is split into partitions; each is an **ordered, append-only log**. Partitions are the unit of parallelism and ordering                    |
| **Offset**           | A message's position within a partition                                                                                                           |
| **Broker**           | A Kafka server; a cluster has many brokers                                                                                                        |
| **Producer**         | Writes messages; the **key** decides the partition (`hash(key) % partitions`), so the same key always lands in the same partition                 |
| **Consumer group**   | A set of consumers sharing a topic; **each partition is read by exactly one consumer in the group**. Different groups each receive _all_ messages |
| **Committed offset** | The consumer group's saved progress per partition, so it can resume after a restart                                                               |

```text
Topic "orders" (3 partitions)
  P0: [m0][m1][m2][m3] ──► Consumer A ┐
  P1: [m0][m1][m2]     ──► Consumer B ├─ Group "billing"   (shares the work)
  P2: [m0][m1]         ──► Consumer C ┘
  (all partitions)     ──► Group "analytics" (gets every message independently)
```

### Ordering and Parallelism

- Order is guaranteed **only within a partition**. To keep all events for one order in sequence, use the order ID as the **key**.
- Maximum parallelism for a group = **number of partitions**. More consumers than partitions leaves some idle.
- You can increase partitions later but **not decrease** them, and increasing changes the key-to-partition mapping for new messages. Plan partition counts from target throughput.
- A skewed key (one very busy customer) creates a **hot partition**.

### Durability and Replication

- Each partition has a **replication factor** (commonly 3): one **leader** handles reads and writes; **followers** copy it. If the leader's broker dies, an **in-sync follower** is elected leader.
- For safety, use `acks=all` on producers with `min.insync.replicas=2` (with replication factor 3): a write is confirmed only when multiple replicas have it, so one broker can fail without losing acknowledged data. The cost is higher write latency.
- Modern Kafka manages cluster metadata itself (**KRaft**) instead of relying on ZooKeeper.

### Retention and Compaction

- **Time/size retention:** keep messages for, say, 7 days (or forever), regardless of whether they've been read. Consumers can **replay** by resetting their offset.
- **Log compaction:** keep only the **latest message per key**, useful for "current state" topics such as user profiles.

### Delivery Semantics in Kafka

How and when the consumer commits its offset decides the guarantee:

| Approach                            | Behavior                                                           |
| ----------------------------------- | ------------------------------------------------------------------ |
| Commit offset **before** processing | At-most-once (a crash loses the message)                           |
| Commit offset **after** processing  | At-least-once (a crash causes reprocessing)                        |
| Idempotent producer + transactions  | **Exactly-once** for read-process-write pipelines **inside Kafka** |

The idempotent producer (`enable.idempotence=true`) prevents duplicate writes caused by producer retries. Exactly-once does **not** extend to external side effects (sending an email, charging a card), so consumers still need idempotency.

### Rebalancing and Lag

- When consumers join or leave a group, partitions are **rebalanced**, which briefly pauses consumption. Long processing that exceeds the poll timeout causes consumers to be considered dead, triggering repeated rebalances.
- **Consumer lag** (latest offset minus committed offset) is the key health metric. Growing lag means consumers can't keep up.

### Why Kafka Is Fast

Sequential disk writes, the OS page cache, batching and compression of messages, and zero-copy transfers to consumers. Consumers **pull** data at their own pace.

### When to Use Kafka

**Good fit:** event streaming at high volume, multiple consumers needing the same data, replay and reprocessing, log or metrics pipelines, change data capture, stream processing.

**Poor fit:** simple background task queues needing per-message acknowledgment, delayed delivery, priority, or complex routing — a queue such as SQS or RabbitMQ is simpler.

## Queue vs Log: Choosing a Broker

|                                | Amazon SQS                           | RabbitMQ                           | Kafka                              |
| ------------------------------ | ------------------------------------ | ---------------------------------- | ---------------------------------- |
| Model                          | Managed queue                        | Broker with exchanges and queues   | Distributed partitioned log        |
| After consumption              | Message deleted                      | Message deleted after ack          | Message **retained**; offset moves |
| Replay                         | No                                   | Limited (streams feature aside)    | **Yes**                            |
| Multiple independent consumers | Via SNS fan-out                      | Via exchanges                      | **Native** (consumer groups)       |
| Ordering                       | FIFO per message group (FIFO queues) | Per queue                          | Per partition                      |
| Throughput                     | High (managed)                       | Moderate to high                   | **Very high**                      |
| Routing flexibility            | Basic                                | **Rich** (topic, direct, fanout)   | By topic and key                   |
| Operations                     | None (managed)                       | Moderate                           | Highest (or use a managed service) |
| Best for                       | Simple, reliable task queues         | Complex routing, classic messaging | Event streaming, replay, pipelines |

## Long Polling, WebSockets, and Server-Sent Events

Traditional HTTP is **request-driven** — the client always initiates. For real-time applications, the server needs to push data to clients. Three patterns address this.

### Long Polling

The client sends an HTTP request; the server **holds the connection open** until new data is available or a timeout occurs. Once data is returned, the client immediately sends another request.

![Long Polling](/images/long-polling.png)

**Pros:** Simple; works everywhere HTTP works; no special infrastructure.
**Cons:** Each "push" costs a new request; many held connections; higher server overhead at scale.

**Use cases:** Low-frequency notifications, simple chat, when WebSocket infrastructure is unavailable.

### WebSockets

WebSockets establish a **persistent, full-duplex connection** (over TCP) between client and server, allowing both to send data at any time.

![WebSockets](/images/websockets.png)

**Handshake:** The client sends an HTTP request with an `Upgrade: websocket` header. If the server accepts, the connection is upgraded to the WebSocket protocol (`ws://` or `wss://`).

**Pros:**

- True full-duplex — both sides send independently
- Very low latency — no per-message connection overhead
- Efficient for high-frequency updates (stock prices, gaming, chat)

**Cons:**

- Dropped connections are not automatically recovered — the client must reconnect
- Connections are **stateful**, which complicates horizontal scaling
- Proxies and load balancers may close idle long-lived connections

**Use cases:** Real-time chat, live gaming, collaborative editing, financial tickers, live dashboards.

#### Scaling WebSockets

Each connection lives on **one specific server** for its whole life, which raises problems that stateless HTTP doesn't have:

- **Routing messages between servers:** if user A is connected to server 1 and user B to server 2, a message from A to B must reach server 2. Common solutions: a **pub/sub backplane** (Redis Pub/Sub, Kafka) that every server subscribes to, or a **connection registry** (`userId → server`) consulted before sending. See [Design WhatsApp](system-design/whatsapp-design).
- **Heartbeats:** send periodic ping/pong so dead connections are detected and idle timeouts on proxies don't kill live ones.
- **Reconnection:** clients reconnect with exponential backoff and jitter, and resume from the last received event ID to avoid gaps.
- **Capacity:** idle connections are cheap but not free (memory and file descriptors per connection), so plan connections per server and scale out. A mass reconnect after a server restart can overwhelm the system, so spread the reconnects.
- **Load balancer:** it must support WebSocket upgrades and use generous idle timeouts. See [Load Balancing](system-design/load-balancing).

### Server-Sent Events (SSE)

SSE establishes a **persistent, one-way HTTP connection** — the server streams events to the client, but the client cannot send data back on the same connection.

![Server-Sent Events](/images/server-sent-events.png)

**Pros:**

- Simpler than WebSockets — plain HTTP, works through most proxies
- Built-in automatic reconnection and `Last-Event-ID` for resuming
- Native browser `EventSource` API

**Cons:**

- One-directional (client sends via normal requests)
- Text only (no binary)
- Under **HTTP/1.1** browsers limit connections per domain (about 6); HTTP/2 multiplexes many streams over one connection, largely removing this limit

**Use cases:** News feeds, live scores, notifications, log streaming, streaming AI responses.

### Comparison

| Feature        | Long Polling           | WebSockets                    | SSE                        |
| -------------- | ---------------------- | ----------------------------- | -------------------------- |
| Directionality | Request-response       | Full-duplex                   | Server-to-client only      |
| Protocol       | HTTP                   | ws:// / wss://                | HTTP                       |
| Latency        | Moderate               | Low                           | Low                        |
| Binary data    | No                     | Yes                           | No                         |
| Complexity     | Low                    | Medium                        | Low                        |
| Auto-reconnect | No (must re-implement) | No                            | Yes                        |
| Best for       | Infrequent updates     | High-frequency, bidirectional | Server push, notifications |

## REST vs GraphQL vs gRPC

These three API styles represent different philosophies for how services expose capabilities.

### REST (Representational State Transfer)

REST is an **architectural style** for designing networked APIs using HTTP verbs (GET, POST, PUT, PATCH, DELETE) and resource-based URLs.

**Core constraints:**

- Uniform interface — consistent resource URLs and HTTP verb semantics
- Stateless — no client state stored on the server between requests
- Cacheable — responses declare cacheability
- Client-server separation

**Advantages:** Simple, widely understood, excellent HTTP caching, human-readable.

**Disadvantages:** Over-fetching (more data than needed) and under-fetching (multiple round trips to assemble a view).

**Best for:** Public APIs, CRUD services, general-purpose APIs.

**REST design essentials interviewers expect:**

| Topic                  | Guidance                                                                                                                                                        |
| ---------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Idempotent methods** | GET, PUT, DELETE are idempotent (safe to retry); **POST is not** — use an idempotency key for operations like payments                                          |
| **Status codes**       | 200 OK, 201 Created, 204 No Content, 400 bad request, 401 unauthenticated, 403 forbidden, 404 not found, 409 conflict, 429 too many requests, 5xx server errors |
| **Pagination**         | Prefer **cursor-based** pagination for large or changing data; offset pagination gets slow and can skip or repeat items                                         |
| **Versioning**         | Version the API (`/v1/...` or a header) so changes don't break clients                                                                                          |
| **Rate limiting**      | Return 429 with `Retry-After` (see [Rate Limiting](system-design/rate-limiting))                                                                                |

### GraphQL

GraphQL is a **query language and runtime** where the client specifies exactly which fields it needs. Developed by Facebook, open-sourced in 2015.

```graphql
# Client requests exactly what it needs
{
  user(id: "123") {
    name
    email
    posts {
      title
    }
  }
}
```

**Advantages:**

- No over-fetching or under-fetching
- Single endpoint for all operations
- Strongly typed schema acts as documentation
- Excellent for complex, graph-like data models

**Disadvantages:**

- The **N+1 query problem** (one query per item in a list) needs batching, e.g., DataLoader
- HTTP caching is harder because every query can differ
- Needs protection against expensive queries (depth and complexity limits)
- More server-side complexity (resolvers)

**Best for:** Mobile apps (bandwidth-sensitive), complex data graphs, BFF pattern (see [API Gateway](system-design/api-gateway)).

### gRPC

gRPC is a **high-performance RPC framework** from Google using Protocol Buffers for serialization over HTTP/2.

```protobuf
service UserService {
  rpc GetUser (GetUserRequest) returns (User);
  rpc StreamUsers (Empty) returns (stream User);
}
```

**Advantages:**

- High performance — binary messages are typically much smaller and faster to parse than JSON
- Streaming — server, client, or bidirectional
- Strongly typed contracts — `.proto` files generate clients in many languages
- Built-in deadlines/timeouts and code generation

**Disadvantages:**

- Limited browser support (needs gRPC-Web and a proxy)
- Binary format is harder to debug by eye
- Steeper learning curve
- **Load balancing needs care:** gRPC keeps long-lived HTTP/2 connections and multiplexes many calls on them, so an L4 (connection-level) balancer sends all of a client's calls to one backend. Use an **L7 / HTTP/2-aware balancer** (e.g., Envoy), **client-side load balancing**, or a service mesh.

**Best for:** Internal microservice-to-microservice calls, high-throughput services, polyglot environments.

### Comparison Table

| Aspect          | REST                      | GraphQL                 | gRPC                                         |
| --------------- | ------------------------- | ----------------------- | -------------------------------------------- |
| Coupling        | Low                       | Medium                  | Higher (shared contract and generated stubs) |
| Chattiness      | High (multiple endpoints) | Low (single query)      | Medium                                       |
| Performance     | Good                      | Good                    | Excellent                                    |
| Caching         | Excellent (HTTP-native)   | Complex (custom)        | Complex (custom)                             |
| Browser support | Excellent                 | Excellent               | Limited (gRPC-Web)                           |
| Schema/contract | Optional (OpenAPI)        | Required (.graphql)     | Required (.proto)                            |
| Code generation | Mediocre                  | Good                    | Excellent                                    |
| Best for        | Public APIs, CRUD         | Complex queries, mobile | Internal services                            |

## Event Sourcing

Instead of storing current state, event sourcing stores the **complete history of events** that produced that state; the current state is rebuilt by replaying them.

![Event Sourcing](/images/event-sourcing.png)

**Advantages:** full audit log; can rebuild state at any point in time; natural fit with CQRS.

**Disadvantages:** large event stores; querying current state needs snapshots or read models; event schema evolution is hard.

**Use cases:** financial ledgers, audit-heavy systems, collaborative editing. The full treatment, including snapshots, projections, and how it pairs with CQRS, is in [CQRS and Event Sourcing](system-design/cqrs-and-event-sourcing).

## Service Discovery

In a dynamic microservices environment, instances start and stop constantly. **Service discovery** lets services find each other without hardcoded IP addresses.

### Client-Side Discovery

The client queries a service registry (e.g., Consul, etcd), picks an instance, and calls it directly.

### Server-Side Discovery

The client calls a load balancer or router, which queries the registry and forwards to a healthy instance. Container platforms such as Kubernetes provide this built in through DNS names for services.

### Service Registry

A database of available instances and their addresses. Instances register on startup and deregister on shutdown; **health checks** remove dead ones.

**Examples:** [Consul](https://www.consul.io), [etcd](https://etcd.io), [Apache ZooKeeper](https://zookeeper.apache.org), [Eureka](https://github.com/Netflix/eureka) (Netflix)

## Choosing a Pattern

| Need                                               | Use                    |
| -------------------------------------------------- | ---------------------- |
| Answer needed right now, simple CRUD               | REST                   |
| Fast internal service-to-service calls             | gRPC                   |
| Clients need flexible, nested queries              | GraphQL                |
| Background tasks, load leveling                    | Message queue          |
| One event, many independent consumers              | Pub/Sub or Kafka topic |
| High-volume event stream, replay                   | Kafka                  |
| Server pushes updates (one-way)                    | SSE                    |
| Two-way real-time (chat, gaming)                   | WebSockets             |
| Simple, infrequent push without new infrastructure | Long polling           |

## Interview Questions

**Q: Queue or pub/sub?**
A queue distributes work: each message is handled by one consumer. Pub/sub broadcasts: each subscriber gets every message. Often combined — a topic fans out to a separate queue per service.

**Q: Can you guarantee exactly-once delivery?**
Not end to end in the general case. Use at-least-once delivery with idempotent consumers (deduplicate by message ID or make operations naturally idempotent), which gives exactly-once _effects_. Kafka transactions give exactly-once only within Kafka pipelines.

**Q: How does Kafka scale and preserve order?**
Topics are split into partitions; consumers in a group split the partitions, so parallelism is limited by the partition count. Order holds only within a partition, so messages that must stay ordered share a key.

**Q: What is a consumer group?**
A set of consumers that share a topic's partitions so each partition is read by one member. Different groups each get all messages independently.

**Q: Kafka vs RabbitMQ or SQS?**
Kafka for high-throughput streaming, retention, and replay with many consumers. RabbitMQ for rich routing and classic messaging. SQS for a simple, fully managed task queue.

**Q: What if a consumer crashes mid-processing?**
The message isn't acknowledged (or the offset isn't committed), so it is redelivered. The consumer must therefore be idempotent. Messages that keep failing go to a DLQ.

**Q: WebSockets, SSE, or long polling?**
WebSockets for two-way, high-frequency communication; SSE for one-way server push with simplicity and auto-reconnect; long polling only when nothing else is available.

**Q: How do you scale a WebSocket chat across many servers?**
Each user's connection is pinned to one server, so use a pub/sub backplane or a connection registry to route a message to the server holding the recipient, plus heartbeats and resumable reconnects.

**Q: Why does gRPC need special load balancing?**
Long-lived HTTP/2 connections carry many requests, so an L4 balancer pins all of one client's calls to a single backend. Use an HTTP/2-aware L7 balancer, client-side balancing, or a mesh.

## Summary

- Use synchronous calls when you need an answer now, and messaging when work can happen later; most systems use both.
- Queues distribute work; pub/sub broadcasts; Kafka is a durable, replayable log with partitions and consumer groups.
- Delivery is realistically **at-least-once**, so design **idempotent consumers**, retry with backoff and jitter, and use DLQs for poison messages.
- Kafka orders messages only within a partition; the key decides the partition, and partitions cap parallelism.
- WebSockets are stateful and need a backplane to scale; SSE is the simpler choice for one-way push.
- REST for public CRUD, GraphQL for flexible client queries, gRPC for internal high-throughput calls (with L7 or client-side load balancing).

## Continue Learning

- [Event-Driven Architecture](system-design/event-driven) — the outbox pattern, schema evolution, and choreography vs orchestration.
- [CQRS and Event Sourcing](system-design/cqrs-and-event-sourcing) — events as the source of truth.
- [Design WhatsApp](system-design/whatsapp-design) — WebSockets and message delivery in a real design.
- [Load Balancing](system-design/load-balancing) — balancing WebSockets and gRPC.
- [Rate Limiting](system-design/rate-limiting) — protecting APIs and producers.
