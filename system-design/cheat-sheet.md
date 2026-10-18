---
title: "System Design Cheat Sheet — Numbers, Formulas, Trade-offs, and Practice Questions"
description: "A quick-reference sheet for system design interviews: latency numbers, units and conversions, capacity formulas, rough throughput figures, component selection, trade-off tables, a 45-minute answer template, and a bank of practice questions with approaches."
author: ["name": "Rajendra Pancholi", "email": "rpancholi522@gmail.com"]
created: "2026-10-17"
updated: "2026-10-17"
thumbnail: "/images/system-design-interview.png"
tags: [system-design, interview, cheat-sheet, estimation, trade-offs, practice]
keywords: ["System design cheat sheet", "Latency numbers every programmer should know", "Back of the envelope estimation", "System design trade-offs", "System design practice questions", "System design interview template"]
---

# System Design Cheat Sheet — Numbers, Formulas, Trade-offs, and Practice Questions

This page is for **revision** and for use during practice. It collects the numbers, formulas, and comparisons you need most often. Every figure is an **order-of-magnitude** estimate: in an interview, state your assumptions out loud and the exact value matters much less than the reasoning.

## 1. Latency Numbers (Approximate)

| Operation                                 | Approximate time                |
| ----------------------------------------- | ------------------------------- |
| L1 cache reference                        | ~1 ns                           |
| L2 cache reference                        | ~4 ns                           |
| Mutex lock/unlock                         | ~20 ns                          |
| Main memory (RAM) reference               | ~100 ns                         |
| Read 1 MB sequentially from RAM           | ~50–100 µs                      |
| SSD random read                           | ~20–150 µs                      |
| Read 1 MB sequentially from SSD           | ~0.3–2 ms                       |
| Round trip within the same data center    | ~0.5 ms                         |
| Redis/Memcached GET (same zone)           | ~0.3–1 ms                       |
| Simple indexed database query (same zone) | ~1–10 ms                        |
| HDD seek                                  | ~5–10 ms                        |
| Read 1 MB sequentially from HDD           | ~5–10 ms                        |
| Round trip across continents              | ~100–250 ms                     |
| TLS 1.3 handshake                         | 1 extra round trip (plus TCP's) |

**What to remember from the table:**

- RAM is roughly **1,000x faster** than an SSD random read, and an SSD roughly **100x faster** than an HDD seek.
- **Sequential** access is far cheaper than random access on every storage type.
- A **network round trip** inside a data center costs thousands of memory references, and a cross-continent trip costs a million. Minimize round trips, batch requests, and avoid synchronous cross-region calls on the hot path.
- A **cache hit saves milliseconds**; a CDN hit saves **hundreds of milliseconds**. See [Caching and CDN](system-design/caching-and-cdn).

## 2. Units and Conversions

### Data Sizes

| Power of 2 | Approx. value | Name |
| ---------- | ------------- | ---- |
| 2^10       | 1 thousand    | 1 KB |
| 2^20       | 1 million     | 1 MB |
| 2^30       | 1 billion     | 1 GB |
| 2^40       | 1 trillion    | 1 TB |
| 2^50       | 1 quadrillion | 1 PB |

### Time

| Period  | Seconds (round it)        |
| ------- | ------------------------- |
| 1 day   | 86,400 (~10^5)            |
| 1 month | ~2.6 million              |
| 1 year  | ~31.5 million (~3 × 10^7) |

### Requests per Day → Requests per Second

| Per day     | Per second (average) |
| ----------- | -------------------- |
| 1 million   | ~12                  |
| 10 million  | ~120                 |
| 100 million | ~1,200               |
| 1 billion   | ~12,000              |

**Peak traffic** is usually **2–3x** the average (more for spiky products such as ticket sales or flash sales).

### Network

- **Mbps ÷ 8 = MB/s.** A 1 Gbps link carries about 125 MB/s; 10 Gbps about 1.25 GB/s.
- 1 hour of video at 3 Mbps is about **1.35 GB**; 1 minute of 1080p video at 5 Mbps is about **37 MB**.

### Typical Object Sizes

| Item                                 | Size                         |
| ------------------------------------ | ---------------------------- |
| 64-bit ID or timestamp               | 8 bytes                      |
| UUID                                 | 16 bytes binary (36 as text) |
| Short text message / tweet           | ~100–300 bytes               |
| A typical JSON record / database row | ~0.5–1 KB                    |
| Compressed photo                     | ~200 KB – 3 MB               |
| Average web page (HTML + assets)     | ~1–3 MB                      |

## 3. Capacity Estimation Formulas

```text
Requests/sec (avg)   = DAU × actions per user per day / 86,400
Peak requests/sec    = average × 2–3
Read/write split     = total RPS × ratio  (e.g., 10:1 → reads 91%, writes 9%)

Storage/day          = writes per day × average record size
Total storage        = storage/day × 365 × years × replication factor (usually 3)

Bandwidth            = RPS × average payload size  (convert bytes → bits for Mbps)

Cache memory         = daily reads × 20% (hot data) × object size      (80/20 rule)

App servers          = peak RPS / RPS per server, then add 30–50% headroom and N+1
DB shards            = max(total data / data per node, peak ops/s / ops per node)

Concurrent connections (chat, live) ≈ users online at the same time
```

### A 60-Second Worked Example

> **100M DAU, each makes 10 requests/day, average response 5 KB, 10% are writes of 1 KB.**
>
> - Requests: 100M × 10 = 1B/day → 1B / 86,400 ≈ **12,000 RPS** average, **~30,000 RPS** at peak.
> - Writes: 10% of 1B = 100M writes/day × 1 KB = **100 GB/day**; ×365 ≈ 36 TB/year; ×3 replicas ≈ **110 TB/year**.
> - Egress: 12,000 RPS × 5 KB = 60 MB/s average ≈ **480 Mbps**; ~1.2 Gbps at peak.
> - Cache: 900M reads/day × 20% × 5 KB ≈ **900 GB**, so a small Redis cluster, not a single node.
>
> Conclusion: thousands of RPS is easy for a few app servers, and the real design questions are storage growth and the read path.

## 4. Rough Throughput of Common Components

These vary enormously with hardware and workload, so treat them as **order-of-magnitude** guides and say "assume" in an interview.

| Component                           | Rough capability per node                                         |
| ----------------------------------- | ----------------------------------------------------------------- |
| Typical API/app server              | hundreds to a few thousand requests/s                             |
| NGINX (proxying or static files)    | tens of thousands of requests/s                                   |
| Redis (simple operations)           | ~50,000–100,000+ ops/s                                            |
| PostgreSQL / MySQL (simple queries) | thousands to tens of thousands of queries/s; writes lower         |
| Cassandra (writes)                  | tens of thousands of writes/s                                     |
| Kafka broker                        | hundreds of MB/s; clusters reach millions of messages/s           |
| WebSocket server                    | tens of thousands of idle connections with tuning, more with care |

**Availability reference:** 99.9% is about 8.8 hours of downtime a year, 99.99% about 53 minutes. See [High Availability](system-design/high-availability).

## 5. If You Hear X, Think Y

| The problem says…                               | Reach for…                                                         | Read more                                                                                                       |
| ----------------------------------------------- | ------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------- |
| Read-heavy                                      | Cache (cache-aside), read replicas, CDN                            | [Caching and CDN](system-design/caching-and-cdn), [Databases](system-design/databases)                          |
| Write-heavy                                     | Sharding, queues to smooth bursts, LSM-based stores                | [Databases](system-design/databases)                                                                            |
| One hot key / celebrity                         | Local cache, key splitting, hybrid fan-out                         | [Design Twitter](system-design/twitter-design)                                                                  |
| Global users                                    | CDN, GeoDNS/Anycast, regional deployment                           | [DNS](system-design/dns), [Load Balancing](system-design/load-balancing)                                        |
| Must not charge or process twice                | Idempotency keys, deduplication, unique constraints                | [Messaging and Communication](system-design/messaging-and-communication)                                        |
| Ordering matters                                | Partition by key (one partition per entity)                        | [Messaging and Communication](system-design/messaging-and-communication)                                        |
| Traffic is bursty                               | Message queue for load leveling, autoscaling, rate limiting        | [Rate Limiting](system-design/rate-limiting)                                                                    |
| Two users must not get the same resource        | Atomic conditional update, short reservation with TTL              | [Design Uber](system-design/uber-design), [Design a Ticket Booking System](system-design/ticket-booking-design) |
| Real-time push                                  | WebSockets (two-way) or SSE (one-way) with a pub/sub backplane     | [Messaging and Communication](system-design/messaging-and-communication)                                        |
| Location / "nearby"                             | Geohash, H3, or quadtree                                           | [Design Uber](system-design/uber-design)                                                                        |
| Large files                                     | Object storage, chunking, pre-signed URLs, resumable uploads       | [Design a File Storage Service](system-design/file-storage-design)                                              |
| Full-text search / autocomplete                 | Inverted index (Elasticsearch), trie, precomputed top-K            | [Design Search Autocomplete](system-design/typeahead-design)                                                    |
| Unique counts at huge scale                     | HyperLogLog                                                        | [Distributed Systems Essentials](system-design/distributed-systems-essentials)                                  |
| "Have I seen this?"                             | Bloom filter                                                       | [Distributed Systems Essentials](system-design/distributed-systems-essentials)                                  |
| Unique IDs across machines                      | Snowflake-style or time-ordered IDs                                | [Distributed Systems Essentials](system-design/distributed-systems-essentials)                                  |
| Multi-step business transaction across services | Saga, outbox                                                       | [Distributed Systems Essentials](system-design/distributed-systems-essentials)                                  |
| Only one worker should run a job                | Lease with fencing token, or a database conditional update         | [Distributed Systems Essentials](system-design/distributed-systems-essentials)                                  |
| Dependency is slow or failing                   | Timeouts, retries with backoff and jitter, circuit breaker         | [Circuit Breaker Pattern](system-design/circuit-breaker)                                                        |
| Protect the API from abuse                      | Rate limiter at the gateway                                        | [Design a Rate Limiter](system-design/rate-limiter-design)                                                      |
| Decouple producers and consumers                | Queue (work distribution) or log such as Kafka (streaming, replay) | [Messaging and Communication](system-design/messaging-and-communication)                                        |
| Audit trail / history                           | Event sourcing, append-only log                                    | [CQRS and Event Sourcing](system-design/cqrs-and-event-sourcing)                                                |

## 6. Common Component Choices

| Need                           | Default choice             | Choose something else when…                                                                                                |
| ------------------------------ | -------------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| Primary database               | PostgreSQL / MySQL         | You need massive write throughput or key-based access at huge scale (Cassandra, DynamoDB), or flexible documents (MongoDB) |
| Cache                          | Redis                      | You need only the simplest cache (Memcached)                                                                               |
| Queue                          | SQS / RabbitMQ             | You need replay and many independent consumers (Kafka)                                                                     |
| Event stream                   | Kafka                      | A simple task queue is enough                                                                                              |
| Search                         | Elasticsearch / OpenSearch | The database's built-in text search is sufficient                                                                          |
| Files and media                | Object storage (S3) + CDN  | —                                                                                                                          |
| Real-time                      | WebSockets                 | One-way push only (SSE)                                                                                                    |
| Load balancer                  | L7 (ALB/NGINX/Envoy)       | Raw TCP/UDP or extreme connection counts (L4)                                                                              |
| Coordination / leader election | etcd / ZooKeeper / Consul  | A simple database lease is enough                                                                                          |
| Metrics                        | Prometheus + Grafana       | See [Observability and Deployment](system-design/observability-and-deployment)                                             |

## 7. Trade-off Quick Reference

| Choice                          | Pick the first when…                                                       | Pick the second when…                                       |
| ------------------------------- | -------------------------------------------------------------------------- | ----------------------------------------------------------- |
| SQL vs NoSQL                    | Relationships, ad-hoc queries, transactions                                | Known access patterns, huge scale, flexible schema          |
| Strong vs eventual consistency  | Money, inventory, bookings                                                 | Feeds, counters, likes, analytics                           |
| Cache-aside vs write-through    | Most read-heavy systems                                                    | Reads must always see the latest write                      |
| Fan-out on write vs on read     | Most users have few followers                                              | Users with millions of followers (use a hybrid)             |
| Push vs pull                    | Low-latency updates, few clients                                           | Many clients that can poll at their own pace                |
| Queue vs log (Kafka)            | Task distribution, per-message ack                                         | Streaming, replay, many consumers                           |
| Sync vs async                   | The caller needs the answer now                                            | Work can happen later, or the caller shouldn't wait         |
| REST vs gRPC vs GraphQL         | Public CRUD API / internal high-throughput calls / flexible client queries | (respectively)                                              |
| Monolith vs microservices       | Small team, early product                                                  | Many teams, independent scaling and deployment              |
| Polling vs WebSocket vs SSE     | Infrequent updates / two-way real-time / one-way push                      | (respectively)                                              |
| L4 vs L7 load balancing         | Raw TCP/UDP, TLS passthrough, huge scale                                   | HTTP routing, gRPC, path or header rules                    |
| Hash vs range sharding          | Even distribution                                                          | Range queries (watch for hot spots)                         |
| Fail open vs fail closed        | General APIs (availability first)                                          | Login, payments (security first)                            |
| Active-passive vs active-active | Simplicity                                                                 | Capacity use and instant takeover (but conflicts to handle) |

Always finish a choice with: **why this, what it costs, and what you'd use instead if the constraint changed.**

## 8. The 45-Minute Answer Template

| Minutes | Step                   | What to say and do                                                                                                                                |
| ------- | ---------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| 0–5     | **Clarify**            | Users, core features, scale, read/write ratio, latency, consistency, what is out of scope. Write down functional and non-functional requirements. |
| 5–10    | **Estimate**           | DAU → RPS (average and peak), storage per year, bandwidth, cache size. State assumptions.                                                         |
| 10–15   | **API and data model** | 4–6 key endpoints, main entities, the database choice and why.                                                                                    |
| 15–25   | **High-level design**  | Client → load balancer → services → cache → database, plus queues, CDN, and storage. Walk through one request end to end.                         |
| 25–40   | **Deep dive**          | Pick 2–3 hard parts (hot keys, ordering, locking, fan-out, search) and discuss alternatives and trade-offs.                                       |
| 40–45   | **Wrap up**            | Bottlenecks, single points of failure, failure handling, monitoring, security, and what you'd improve next.                                       |

**Closing checklist (say these out loud):**

- [ ] Single points of failure removed (load balancer, database, cache)?
- [ ] What happens when a dependency is slow or down (timeouts, retries, circuit breaker, fallback)?
- [ ] Is every retry-able operation idempotent?
- [ ] Hot keys or hot partitions?
- [ ] Data growth and retention, backups?
- [ ] Monitoring, metrics, alerts, and logs?
- [ ] Authentication, authorization, rate limiting, and encryption?
- [ ] Cost, and what you'd simplify for a first version?

For the full framework see the [System Design Interview Guide](system-design/system-design-interview-guide).

## 9. Practice Question Bank

For each question, say the **core idea** in one minute before designing. Pages that already contain the design are linked.

### Foundations

| #   | Question            | Core ideas                                                                                                                     |
| --- | ------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| 1   | URL shortener       | Base62 / key generation, cache-heavy reads, 301 vs 302, expiry. → [Design a URL Shortener](system-design/url-shortener-design) |
| 2   | Rate limiter        | Token bucket, Redis Lua, gateway placement, fail-open. → [Design a Rate Limiter](system-design/rate-limiter-design)            |
| 3   | Pastebin            | Like a URL shortener, with content in object storage and a metadata database                                                   |
| 4   | Unique ID generator | Snowflake-style IDs, clock regression. → [Distributed Systems Essentials](system-design/distributed-systems-essentials)        |
| 5   | Key-value store     | Consistent hashing, replication, quorum reads/writes, LSM tree. → [Databases](system-design/databases)                         |
| 6   | Distributed cache   | Sharding with consistent hashing, eviction, replication, hot keys. → [Caching and CDN](system-design/caching-and-cdn)          |
| 7   | Leaderboard         | Redis sorted sets, sharding by score range, periodic snapshots                                                                 |

### Real-Time and Social

| #   | Question                      | Core ideas                                                                                                                                    |
| --- | ----------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| 8   | Chat application              | WebSockets, connection registry, ordering, receipts, E2E encryption. → [Design WhatsApp](system-design/whatsapp-design)                       |
| 9   | News feed / social timeline   | Fan-out on write vs read, celebrity problem. → [Design Twitter](system-design/twitter-design)                                                 |
| 10  | Notification system           | Queues per channel, retries, deduplication, user preferences. → [Design a Notification System](system-design/notification-system-design)      |
| 11  | Nearby friends / places       | Geohash or H3, location updates, pub/sub. → [Design Uber](system-design/uber-design)                                                          |
| 12  | Collaborative document editor | WebSockets, operational transformation or CRDTs, versioning. → [Distributed Systems Essentials](system-design/distributed-systems-essentials) |
| 13  | Live comments / live scores   | SSE or WebSockets, pub/sub backplane, fan-out                                                                                                 |

### Media and Storage

| #   | Question              | Core ideas                                                                                                                     |
| --- | --------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| 14  | Video streaming       | Encoding pipeline, adaptive bitrate, CDN, DRM. → [Design Netflix](system-design/netflix-design)                                |
| 15  | File storage and sync | Chunking, deduplication, metadata service, sync protocol. → [Design a File Storage Service](system-design/file-storage-design) |
| 16  | Photo sharing         | Object storage, CDN, thumbnail generation, feed                                                                                |
| 17  | Web crawler           | URL frontier, politeness, deduplication with Bloom filters. → [Design a Web Crawler](system-design/web-crawler-design)         |
| 18  | Search autocomplete   | Trie, top-K per prefix, caching. → [Design Search Autocomplete](system-design/typeahead-design)                                |

### Transactions and Commerce

| #   | Question                | Core ideas                                                                                                                    |
| --- | ----------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| 19  | Ticket / seat booking   | Seat holds with TTL, atomic updates, waiting queue. → [Design a Ticket Booking System](system-design/ticket-booking-design)   |
| 20  | Ride hailing            | Geospatial index, dispatch locking, payments. → [Design Uber](system-design/uber-design)                                      |
| 21  | Payment system / wallet | Idempotency keys, double-entry ledger, reconciliation, webhooks. → [Design Uber](system-design/uber-design) (payment section) |
| 22  | E-commerce flash sale   | Inventory counters, queueing users, rate limiting. → [Design a Ticket Booking System](system-design/ticket-booking-design)    |
| 23  | Hotel booking           | Availability search, holds, double-booking prevention                                                                         |

### Infrastructure

| #   | Question                       | Core ideas                                                                                                               |
| --- | ------------------------------ | ------------------------------------------------------------------------------------------------------------------------ |
| 24  | Message queue / Kafka-like log | Partitions, replication, consumer groups. → [Messaging and Communication](system-design/messaging-and-communication)     |
| 25  | Job scheduler                  | Delayed queue, leases, idempotent jobs, retries                                                                          |
| 26  | Metrics and monitoring system  | Time-series storage, aggregation, alerting. → [Observability and Deployment](system-design/observability-and-deployment) |
| 27  | API gateway                    | Routing, auth, rate limiting. → [API Gateway](system-design/api-gateway)                                                 |
| 28  | CDN                            | Edge caching, routing, invalidation. → [Caching and CDN](system-design/caching-and-cdn)                                  |
| 29  | Login / authentication service | Sessions vs tokens, OAuth, refresh tokens. → [Authentication and Security](system-design/auth-and-security)              |

## 10. Study Map

| Topic                      | Page                                                                                                                                                                                           |
| -------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Framework and process      | [What is System Design?](system-design/what-is-system-design), [System Design Interview Guide](system-design/system-design-interview-guide)                                                    |
| Networking, DNS            | [Networking Fundamentals](system-design/networking-fundamentals), [DNS](system-design/dns)                                                                                                     |
| Scaling and load balancing | [Scaling](system-design/scaling), [Load Balancing](system-design/load-balancing)                                                                                                               |
| Caching and CDN            | [Caching and CDN](system-design/caching-and-cdn)                                                                                                                                               |
| Data                       | [Databases](system-design/databases)                                                                                                                                                           |
| Messaging, APIs, real-time | [Messaging and Communication](system-design/messaging-and-communication), [API Gateway](system-design/api-gateway)                                                                             |
| Architecture               | [Monolith](system-design/monolith), [Microservices](system-design/microservices), [Event-Driven](system-design/event-driven), [CQRS and Event Sourcing](system-design/cqrs-and-event-sourcing) |
| Reliability                | [High Availability](system-design/high-availability), [Circuit Breaker](system-design/circuit-breaker), [Rate Limiting](system-design/rate-limiting)                                           |
| Distributed systems        | [Distributed Systems Essentials](system-design/distributed-systems-essentials)                                                                                                                 |
| Operations                 | [Observability and Deployment](system-design/observability-and-deployment)                                                                                                                     |
| Security                   | [Authentication and Security](system-design/auth-and-security)                                                                                                                                 |
