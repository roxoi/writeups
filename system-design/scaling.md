---
title: "Vertical vs Horizontal Scaling"
description: "How systems handle growth: vertical and horizontal scaling, stateless design, scaling each layer, autoscaling, and the limits that stop systems from scaling."
author: ["name": "Rajendra Pancholi", "email": "rpancholi522@gmail.com"]
created: "2026-04-18"
updated: "2026-10-17"
thumbnail: "/images/load-balancing.png"
tags: [scalability, scaling, horizontal-scaling, vertical-scaling, autoscaling, system-design]
keywords: ["Vertical vs horizontal scaling", "Scalability in system design", "Stateless services", "Autoscaling", "Amdahl's law", "How to scale a system"]
---

# Vertical vs Horizontal Scaling

Scalability is one of the most important non-functional requirements in system design. This guide covers what scalability really means, the two ways to scale, why stateless design matters, how each layer of a system scales, and where scaling hits its limits.

## What Does "Scalable" Mean?

Scalability measures how well a system **copes with growth in load** by adding resources. It is not the same as being fast. A system that answers in 10 ms for one user is fast; it is scalable only if it still performs acceptably, at a predictable cost, when a million users show up.

Before you can talk about scaling you need to describe the load. Common **load parameters** are:

- Requests per second (RPS)
- Ratio of reads to writes
- Number of concurrent users or connections
- Data volume and growth rate
- Fan-out (how many followers or recipients one action touches)

The question to ask is: _if load doubles and I double the resources, does the system keep up?_ If yes, it scales well. If it needs five times the resources, or never catches up, you have found a scalability problem.

## Vertical Scaling (Scale Up)

Increase the capacity of an existing machine: more CPU cores, more RAM, faster storage.

| Advantages                                    | Disadvantages                                                  |
| --------------------------------------------- | -------------------------------------------------------------- |
| Simple, with no application changes needed    | Hard upper limit (you cannot upgrade a single machine forever) |
| No distributed system complexity              | Risk of downtime during upgrades                               |
| Consistent data: one machine, no partitioning | Single point of failure                                        |
| Lower operational overhead                    | Cost grows faster than capacity at the high end                |

**Best for:** databases that are hard to shard, stateful services, legacy applications, and early-stage products where simplicity matters more than limitless growth.

## Horizontal Scaling (Scale Out)

Add more machines and distribute load across them.

| Advantages                                                              | Disadvantages                                               |
| ----------------------------------------------------------------------- | ----------------------------------------------------------- |
| Capacity can grow far beyond one machine                                | Requires stateless services or distributed state management |
| Better fault tolerance: one node failing does not take down the service | Higher operational complexity                               |
| Cost-effective with commodity hardware                                  | Data consistency challenges                                 |
| Enables rolling deployments without downtime                            | Needs load balancing and service discovery                  |

**Best for:** stateless services, web servers, API layers, caches.

### Which One Should You Choose?

In practice most systems use both. A common approach is **diagonal scaling**: scale up while it is cheap and simple, then scale out when you approach the limits of a single machine or need redundancy. Application servers are usually scaled out early (it is cheap and gives you failover), while databases are scaled up and replicated first and sharded last.

## Stateless vs Stateful Services

Horizontal scaling works best when any server can handle any request. That requires **stateless** servers: they keep no user-specific data between requests, so the [load balancer](system-design/load-balancing) can send each request to any instance, and instances can be added or removed freely.

A **stateful** server remembers something about the client (a login session, a shopping cart in memory, an open connection). If that server dies, the state dies with it, and requests must keep going back to the same server.

To make an application tier stateless, move the state out:

| State           | Where to put it                                                                    |
| --------------- | ---------------------------------------------------------------------------------- |
| User sessions   | A shared store such as Redis, or a signed token (such as a JWT) held by the client |
| Uploaded files  | [Object storage](system-design/file-storage-design), not the local disk            |
| Cached data     | A shared [cache](system-design/caching-and-cdn)                                    |
| Background work | A [message queue](system-design/messaging-and-communication)                       |

**Sticky sessions** (always routing a user to the same server) are a workaround, not a solution. They make load uneven and lose the session when that server fails.

Some components cannot be stateless by nature, such as databases and WebSocket gateways holding open connections. Those need other techniques: replication, sharding, and connection draining.

## Scaling Each Layer

| Layer                | How it scales                                                                | Notes                                       |
| -------------------- | ---------------------------------------------------------------------------- | ------------------------------------------- |
| App / web servers    | Add instances behind a load balancer                                         | Must be stateless                           |
| Static content       | Serve from a [CDN](system-design/caching-and-cdn)                            | Offloads traffic from your servers entirely |
| Cache                | Add nodes and spread keys with [consistent hashing](system-design/databases) | Cache misses still hit the database         |
| Database reads       | [Read replicas](system-design/databases), caching                            | Replicas lag behind the leader              |
| Database writes      | Scale up first, then [shard](system-design/databases)                        | Hardest layer to scale                      |
| File storage         | Object storage (S3-style)                                                    | Designed for horizontal growth              |
| Slow background work | Queue plus a pool of workers                                                 | Workers scale independently of the web tier |

### Scaling Reads vs Writes

Start by measuring your **read-to-write ratio**, because the two scale very differently.

- **Reads are easy to scale.** The same data can be copied: cache it, replicate it, push it to a CDN, or precompute (denormalize) it.
- **Writes are hard to scale.** Each write must go to the authoritative copy, so one leader becomes a bottleneck. Options are sharding (spread writes across leaders), batching writes, buffering them through a queue, and using storage engines built for heavy writes.

A system that is 99% reads and one that is 50% writes need very different designs.

## Autoscaling

Autoscaling adds or removes instances automatically based on demand, so you pay for what you use and survive traffic spikes without manual work.

**Approaches**

- **Reactive (metric-based):** scale on CPU, memory, requests per instance, or queue depth.
- **Scheduled:** scale ahead of known events (a sale at 8 p.m., a daily batch job).
- **Predictive:** use historical patterns to scale before the load arrives.

A common **target-tracking** rule is:

```text
desired instances = ceil(current instances × current metric / target metric)
```

For example, 4 instances running at 90% CPU with a 60% target gives ceil(4 × 90 / 60) = **6 instances**.

**Things to configure**

- Minimum and maximum instance counts (the minimum protects availability; the maximum protects your bill and your database).
- **Cooldown periods** so the system does not add and remove instances repeatedly (thrashing).
- **Warm-up time**, because a new instance is not useful until it has started, loaded its caches, and passed health checks.

**What autoscaling does not fix**

- A slow database: adding app servers sends _more_ load to it.
- Scaling lag: a sudden spike can overwhelm you before new instances are ready. Queues, caches, and rate limiting absorb the gap.
- Downstream limits such as third-party API quotas or database connection limits.

## Limits to Scaling

Adding machines does not give unlimited speed-ups.

**Amdahl's Law.** If a fraction `p` of the work can be parallelized and the rest is serial, the best possible speed-up with `n` processors is:

```text
speedup = 1 / ((1 − p) + p / n)
```

If 5% of the work is serial (p = 0.95), the speed-up is only about 6.9x with 10 processors, and it can never exceed 20x no matter how many you add. The serial part, such as a single lock, one leader database, or one coordinator, is what caps you.

**Coordination costs.** The more nodes must talk to each other (to agree, to share data, to keep caches consistent), the more overhead each added node brings. Past a point, more nodes can even _reduce_ throughput.

**Hot spots.** Even with many nodes, if all traffic hits one partition, one cache key, or one row, that single spot is your limit.

### Finding the Bottleneck

Do not guess. Measure each resource (CPU, memory, disk, network, connections), look at its **utilization**, **saturation** (queueing), and **errors**, fix the biggest bottleneck, then measure again. A load test before launch shows where the system breaks first. Tools for this are covered in [Observability](system-design/observability-and-deployment).

## Trade-offs

| Choice                 | You gain                   | You give up                              |
| ---------------------- | -------------------------- | ---------------------------------------- |
| Scale up               | Simplicity                 | A ceiling, and a single point of failure |
| Scale out              | Capacity and redundancy    | Complexity, consistency challenges       |
| Autoscale              | Lower cost, handles spikes | Configuration effort, scaling lag        |
| Keep servers stateless | Easy scaling and failover  | Need an external store for state         |

## Real-World Usage

- **E-commerce flash sale:** scale the app tier out ahead of time on a schedule, cache product pages, put checkout behind a queue, and rate-limit abusive clients.
- **Early-stage startup:** one large database with a few stateless app servers handles a surprising amount of traffic. Premature sharding costs more than it saves.
- **Global products:** serve static assets from a CDN and replicate data near users to cut latency.

## Interview Questions

**Q: When would you choose vertical scaling over horizontal scaling?**
When the workload is hard to distribute (a single relational database, a stateful legacy service), when you want the simplest possible setup, or early on when a bigger machine is cheaper than the engineering effort of distribution.

**Q: Why do stateless services matter for scalability?**
Any instance can serve any request, so you can add or remove instances freely and survive instance failure without losing user state.

**Q: How would you scale a database?**
Reads first: add caching and read replicas. For heavier load, scale up the leader. For write or data-size limits, shard the data. Each step adds complexity, so move to the next only when the current one is not enough.

**Q: Why might adding more servers not improve performance?**
The bottleneck may be elsewhere (a database, a lock, a hot key, a third-party limit), or a serial portion of the work limits the speed-up as described by Amdahl's Law.

**Q: Your autoscaling group keeps adding and removing instances every few minutes. What is wrong?**
Likely thrashing: the thresholds are too close together, there is no cooldown, or instances take too long to warm up. Widen the thresholds, add cooldown, and scale on a smoother metric.

## Summary

- Scalability means performance and cost stay predictable as load grows. It is different from raw speed.
- Scale up for simplicity, scale out for capacity and redundancy. Most systems do both.
- Stateless services are the foundation of horizontal scaling. Move sessions, files, and cache out of the app server.
- Reads are easy to scale; writes are hard.
- Autoscaling helps but does not fix a bottleneck somewhere else.
- Always measure to find the real bottleneck before adding resources.

## Continue Learning

- [Load Balancing](system-design/load-balancing) — spreading traffic across the servers you add.
- [Caching and CDN](system-design/caching-and-cdn) — reducing load before it reaches the database.
- [Databases](system-design/databases) — replication, sharding, and scaling reads and writes.
- [High Availability](system-design/high-availability) — staying up while you scale.
