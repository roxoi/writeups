---
title: "Load Balancing — Distributing Traffic for High Availability and Scale"
description: "A complete guide to load balancing in system design: L4 vs L7, algorithms (round robin, least connections, power of two choices, consistent hashing), health checks, connection draining, sticky sessions, redundancy, and global load balancing."
author: ["name": "Rajendra Pancholi", "email": "rpancholi522@gmail.com"]
created: "2026-04-18"
updated: "2026-10-17"
thumbnail: "/images/load-balancing.png"
tags: [load-balancing, system-design, scalability, high-availability, infrastructure]
keywords: ["Load balancing explained", "L4 vs L7 load balancer", "Load balancing algorithms", "Round robin vs least connections", "gRPC load balancing", "Health checks and connection draining", "How load balancers work"]
---

# Load Balancing — Distributing Traffic for High Availability and Scale

Load balancing is the practice of **distributing incoming network traffic across multiple servers** to prevent any single server from becoming a bottleneck. It is one of the most fundamental techniques for building high-availability, scalable systems.

Without load balancing, a single server must handle all requests — creating a single point of failure and a performance ceiling. With load balancing, traffic is spread across a pool of servers, enabling horizontal scaling, improved reliability, and zero-downtime deployments.

![Load Balancer Architecture](/images/load-balancing.png)

## Why Load Balancing is Essential

Modern web applications serving millions of users cannot run on a single server. Load balancers solve several critical problems simultaneously:

- **Eliminate single points of failure** — If one server crashes, the load balancer routes traffic to remaining healthy servers.
- **Enable horizontal scaling** — New servers can be added to the pool and begin receiving traffic without downtime (see [Scaling](system-design/scaling)).
- **Improve performance** — Distributing work across multiple machines reduces response times and prevents resource exhaustion.
- **Enable rolling deployments** — Servers can be updated one at a time while the load balancer keeps traffic flowing to healthy instances.
- **Provide SSL termination** — The load balancer can handle TLS handshakes, offloading cryptographic work from backend servers.

## Workload Distribution Strategies

Load balancers can make routing decisions based on different signals:

| Strategy          | Description                                                                                              |
| ----------------- | -------------------------------------------------------------------------------------------------------- |
| **Host-based**    | Routes requests based on the `Host` header (e.g., `api.example.com` vs `static.example.com`)             |
| **Path-based**    | Routes based on the URL path (e.g., `/api/*` to backend servers, `/images/*` to CDN or media servers)    |
| **Content-based** | Inspects message content to make routing decisions (e.g., based on a header, query parameter, or cookie) |

## L4 vs L7 Load Balancers

Load balancers operate at one of two OSI model layers, each with distinct capabilities and trade-offs.

### Layer 4 — Transport Layer Load Balancer

Operates on **IP addresses and TCP/UDP ports** without inspecting the content of the traffic. Routing decisions use only network-level information, and the balancer typically forwards a whole **connection** to one backend.

**Characteristics:**

- Very fast and low overhead — no application-level parsing
- Cannot distinguish between different HTTP requests (sees only connections)
- Does not support content-based routing
- Can pass **encrypted TLS traffic straight through** (TLS passthrough) without decrypting it
- Implemented in software on commodity servers at large scale (Google's Maglev and Meta's Katran are well-known examples), in cloud-managed services, or in dedicated appliances

**Use cases:** High-throughput TCP/UDP services, non-HTTP protocols, databases and message brokers, very large connection counts, when L7 features are not needed.

### Layer 7 — Application Layer Load Balancer

Operates on the **full content of HTTP/HTTPS requests** — headers, URLs, cookies, query parameters. It terminates the client's connection, reads each request, and opens or reuses connections to backends, so it balances **individual requests**, not just connections.

**Characteristics:**

- Can route `/api/*` to one server pool and `/static/*` to another
- Supports cookie-based sticky sessions
- Can inspect and modify headers, add compression, enforce rate limits, and authenticate
- Understands HTTP/2 and gRPC
- Higher processing overhead than L4

**Use cases:** Web applications, microservices routing, A/B testing, canary deployments, HTTP/2 and gRPC traffic.

![L4 vs L7 Load Balancer](/images/load-balancing-layers.png)

### The gRPC (and HTTP/2) Gotcha

gRPC runs over **HTTP/2**, which keeps **one long-lived connection and multiplexes many requests over it**. An L4 balancer balances _connections_, so it sends the one connection — and therefore **all of that client's calls** — to a single backend. The result is badly uneven load even though the balancer looks healthy.

Fix it with one of:

- An **L7, HTTP/2-aware balancer** that balances individual requests (for example Envoy, or an application load balancer with gRPC support)
- **Client-side load balancing**, where the client learns all backend addresses and spreads calls itself
- A **service mesh** that does this per request (see [Microservices](system-design/microservices))

The same issue affects any long-lived, multiplexed connection. See [Messaging and Communication](system-design/messaging-and-communication).

### TLS: Termination vs Passthrough

| Mode                     | How it works                                                          | Trade-off                                                                                                                                             |
| ------------------------ | --------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| **TLS termination** (L7) | Balancer decrypts HTTPS, then sends HTTP (or re-encrypts) to backends | Can inspect and route; offloads crypto; the balancer holds certificates. Re-encrypt to backends (or use mTLS) if internal traffic must stay encrypted |
| **TLS passthrough** (L4) | Balancer forwards encrypted bytes untouched                           | End-to-end encryption; no content-based routing or HTTP features                                                                                      |

## Load Balancing Algorithms

The routing algorithm determines which server receives each request. The right choice depends on server characteristics and traffic patterns.

### Round-Robin

Requests are distributed to servers in sequential rotation: Server 1 → Server 2 → Server 3 → Server 1 → ...

**Best for:** Servers with identical hardware and roughly uniform request times.

**Limitation:** Ignores current load — a slow, long-running request on Server 1 does not stop it from receiving the next one.

### Weighted Round-Robin

Assigns each server a **weight** proportional to its capacity. A server with weight 3 receives 3x as many requests as one with weight 1.

**Best for:** Heterogeneous pools where some servers are more powerful.

### Least Connections

New requests go to the server with the **fewest active connections** right now, which accounts for varying request durations.

**Best for:** Variable request durations, and long-lived connections such as WebSockets.

### Least Response Time

Combines fewest active connections with the fastest observed response time, sending requests to the server likely to answer quickest.

**Best for:** Latency-sensitive applications where server performance varies.

### Least Bandwidth

Routes to the server currently using the least network bandwidth. Useful when response sizes vary widely (file downloads).

### Power of Two Choices (P2C)

Pick **two servers at random** and send the request to the one with fewer active requests. It gets close to the balance of "least connections" while needing little shared state, and avoids a classic problem: when many balancers all pick "the single least-loaded server" from slightly stale data, they all pile onto it at once.

**Best for:** Large fleets and multiple balancer instances. This is the default style in several modern proxies.

### Hash-Based (IP Hash, Consistent Hashing)

A hash of the client IP, session token, or request key chooses the server, so the same key goes to the same server.

- **IP hash** gives simple session affinity, but users behind one NAT or corporate proxy share an IP and land on one server, and a client whose IP changes (mobile networks) loses affinity.
- **Consistent hashing** maps servers and keys onto a ring so that adding or removing a server moves only a small fraction of keys. It is ideal when a request's key matters, such as routing to the cache node that holds a key, or sending a user to the shard that holds their data.

**Limitation:** if a server dies, the keys it served move elsewhere, and any state kept in that server's memory is lost.

### Choosing an Algorithm

| Situation                                             | Good choice              |
| ----------------------------------------------------- | ------------------------ |
| Identical servers, short uniform requests             | Round-robin              |
| Different server sizes                                | Weighted round-robin     |
| Variable or long request duration                     | Least connections or P2C |
| Many balancer instances, large fleet                  | P2C                      |
| Need the same key on the same server (caches, shards) | Consistent hashing       |

## Health Checks

Health checks are what keep traffic away from broken servers.

- **Active checks:** the balancer probes each backend on an interval (HTTP request to `/health`, or a TCP connect).
- **Passive checks (outlier detection):** the balancer watches real traffic and ejects servers that return many errors or time out.
- **Thresholds:** a server is marked unhealthy only after several consecutive failures and healthy again after several successes, so one blip doesn't flap it in and out.

**Design the health endpoint carefully:**

| Check type                                      | What it tests     | Risk                                                                                                                             |
| ----------------------------------------------- | ----------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| **Shallow** (is the process up?)                | The server itself | Can report "healthy" while the app is broken                                                                                     |
| **Deep** (checks database, cache, dependencies) | Dependencies too  | If a **shared** dependency has a brief problem, **every** server fails the check at once and the balancer removes the whole pool |

A good compromise is to separate **liveness** ("restart me if I'm stuck") from **readiness** ("send me traffic now"), keep readiness focused on the instance's ability to serve, and avoid pulling every server out because one optional dependency is down. If all servers are unhealthy, many balancers fail open and send traffic anyway, which is usually better than a total outage.

## Sticky Sessions

Sticky sessions route a client to the same backend on every request.

- **Cookie-based** stickiness (the balancer sets a cookie naming the server) is more reliable than IP hashing.
- Downsides: uneven load, and sessions are lost when that server fails or is replaced.
- **Better approach:** keep servers **stateless** by moving sessions to a shared store (such as Redis) or to signed tokens, so any server can handle any request. See [Scaling](system-design/scaling) and [Caching and CDN](system-design/caching-and-cdn).

Use stickiness only when you truly can't externalize state (for example, WebSocket connections that live on a specific server).

## Types of Load Balancers

### Software Load Balancers

Run as software on commodity hardware or virtual machines. Highly configurable and cost-effective, and capable of very high throughput (large internet companies run their balancing tiers entirely in software).

- **Advantages:** Flexible configuration, easy to update, can be containerized, lower cost
- **Disadvantages:** You operate and tune it yourself
- **Examples:** NGINX, HAProxy, Envoy, Traefik

### Hardware Load Balancers

Physical appliances purpose-built for traffic distribution, common in traditional enterprise data centers.

- **Advantages:** Very high throughput with dedicated hardware, vendor support
- **Disadvantages:** Expensive, inflexible, requires physical management, hard to scale dynamically
- **Examples:** F5 BIG-IP, Citrix ADC

### Cloud-Managed Load Balancers

Provided by the cloud platform, with built-in scaling, health checks, and redundancy. Usually the default choice on the cloud (see the table below).

### DNS Load Balancing

Distributes traffic by returning multiple IP addresses for a domain, rotating through them (round-robin DNS).

- **Advantages:** Simple, no extra infrastructure
- **Disadvantages:** Plain round-robin DNS does no health checking, failover depends on TTL and client caching, and distribution is uneven because resolvers and clients cache answers. Managed DNS services add health checks and failover routing, but remain limited by caching. See [DNS](system-design/dns).

## Deployments: Draining and Slow Start

Two features make rolling deployments safe:

- **Connection draining (deregistration delay):** when a server is being removed or updated, the balancer stops sending it **new** requests but lets in-flight requests and connections **finish** before the server shuts down. Without it, users see dropped requests during every deploy.
- **Slow start:** a freshly started server (cold caches, un-warmed code) is ramped up gradually instead of being flooded. This matters especially with "least connections," which would otherwise send a new, empty server a surge of traffic.

## Redundant Load Balancers

A load balancer itself can become a single point of failure. Production systems avoid this in a few ways:

**Active-passive pair:**

- An **active** balancer handles all traffic
- A **passive** standby watches it via heartbeats
- If the active fails, the standby takes over the shared virtual IP, using protocols such as VRRP (tools like keepalived). Failover typically takes a few seconds, and can be tuned lower.

![Redundant Load Balancers](/images/redundant-load-balancer.png)

**Active-active, scaled out:**

- Several balancer nodes all serve traffic. They may share one IP using **Anycast and ECMP routing** (the network spreads flows across the nodes), or be fronted by DNS.
- Cloud-managed balancers do this for you across multiple availability zones.

**Direct Server Return (DSR)** is an L4 technique where backends send responses **directly to the client**, bypassing the balancer. It helps when requests are small and responses are large (video, downloads), since the balancer then handles only the small inbound side.

See [High Availability](system-design/high-availability) for the broader picture.

## Key Load Balancer Features

| Feature                     | Description                                                                                                       |
| --------------------------- | ----------------------------------------------------------------------------------------------------------------- |
| **Health Checks**           | Probe backends (HTTP, TCP) and remove unhealthy servers from rotation                                             |
| **SSL/TLS Termination**     | Decrypt HTTPS at the balancer (re-encrypt to backends if required)                                                |
| **Sticky Sessions**         | Route a client to the same backend via cookie or hash                                                             |
| **Autoscaling Integration** | Add/remove servers from the pool as the group scales                                                              |
| **Connection Draining**     | Let in-flight requests finish before a server is removed                                                          |
| **Compression**             | Compress responses (gzip/brotli) at the balancer                                                                  |
| **Request Tracing**         | Inject unique request IDs for distributed tracing across microservices                                            |
| **Rate Limiting**           | Limit requests per client to prevent abuse (see [Rate Limiting](system-design/rate-limiting))                     |
| **Timeouts and Retries**    | Bound slow backends; retry carefully to avoid retry storms (see [Circuit Breaker](system-design/circuit-breaker)) |
| **Logging**                 | Log request metadata for auditing and analytics                                                                   |

## Load Balancing in Multi-Layer Architectures

Load balancing is applied at multiple tiers:

```
Client → DNS / GeoDNS → L7 Load Balancer (Edge)
       → API Gateway (L7) → Microservices (client-side or mesh balancing)
       → Cache nodes (consistent hashing)
       → Database: read/write-aware proxy or driver → Primary + Read Replicas
```

![Multi-Layer Load Balancing](/images/load-balancer-layers.png)

**For databases,** a plain L4 balancer can spread connections but doesn't know which node is the writer or how far a replica lags. Use a database-aware proxy or driver that sends writes to the primary and reads to suitable replicas. See [Databases](system-design/databases).

## Global Load Balancing

For users around the world, balance at two levels:

1. **Global:** direct users to the best **region**.
   - **GeoDNS / latency-based DNS** returns the address of the nearest or fastest region.
   - **Anycast** announces one IP from many locations and lets the network deliver users to the closest.
   - Add **health-based failover** so a failed region is removed automatically.
2. **Regional:** a local load balancer spreads traffic across servers (and availability zones) inside that region.

## Production Load Balancer Solutions

| Solution                                                                          | Type          | Notes                                                                                                         |
| --------------------------------------------------------------------------------- | ------------- | ------------------------------------------------------------------------------------------------------------- |
| [Amazon ELB](https://aws.amazon.com/elasticloadbalancing)                         | Cloud (AWS)   | ALB (L7, supports HTTP/2 and gRPC), NLB (L4, TCP/UDP/TLS), Gateway Load Balancer (L3, for network appliances) |
| [NGINX](https://www.nginx.com)                                                    | Software      | Widely used reverse proxy and L7 balancer                                                                     |
| [HAProxy](https://www.haproxy.org)                                                | Software      | High-performance L4/L7, battle-tested                                                                         |
| [Envoy](https://www.envoyproxy.io)                                                | Software      | Modern L7 proxy, gRPC-aware, backbone of many service meshes                                                  |
| [Azure Load Balancer](https://azure.microsoft.com/services/load-balancer)         | Cloud (Azure) | **L4**. For L7 use Azure Application Gateway, and Azure Front Door for global L7                              |
| [GCP Load Balancing](https://cloud.google.com/load-balancing)                     | Cloud (GCP)   | Global and regional options, both L7 (Application) and L4 (Network)                                           |
| [DigitalOcean Load Balancer](https://www.digitalocean.com/products/load-balancer) | Cloud         | Simple managed balancer                                                                                       |

## System Design Takeaways

- Place load balancers between clients and servers, and between service tiers — never expose a single server as the public endpoint.
- Use **L7 load balancers** for HTTP microservices that need path/header routing, and for **HTTP/2 and gRPC**, which need request-level balancing.
- Use **L4 load balancers** for raw TCP/UDP, very high throughput, and TLS passthrough. Don't use plain L4 for gRPC unless clients balance on their own.
- **Health checks are non-negotiable**, but keep them meaningful without letting one shared dependency remove the whole pool.
- Enable **connection draining** and **slow start** so deployments and scale-out don't hurt users.
- Prefer **stateless servers** over sticky sessions.
- Make the balancer itself redundant (active-passive, or active-active with Anycast or a managed service).
- For global applications, combine **GeoDNS or Anycast** (pick the region) with **regional load balancers** (spread within a region).

## Interview Questions

**Q: What is the difference between L4 and L7 load balancing?**
L4 balances TCP/UDP connections using IP and port only: fast, protocol-agnostic, no content-based routing. L7 terminates and reads HTTP requests, so it can route by path, host, or header and balance individual requests, at higher cost.

**Q: Why can gRPC cause uneven load behind an L4 balancer?**
gRPC multiplexes many calls over one long-lived HTTP/2 connection. An L4 balancer balances connections, so all of a client's calls go to one backend. Use an L7 HTTP/2-aware balancer, client-side balancing, or a mesh.

**Q: Round-robin vs least connections vs power of two choices?**
Round-robin is simple but ignores load. Least connections adapts to varying request times. Power of two choices picks two random servers and takes the less loaded, giving near-optimal balance with little state and avoiding herding when many balancers act on stale data.

**Q: How do you do sticky sessions, and what's the problem?**
Use a balancer-set cookie (more reliable than IP hash). Problems: uneven load and lost sessions on failure. Better to make servers stateless with a shared session store.

**Q: How do health checks work, and what can go wrong?**
Active probes and passive error detection mark servers unhealthy after several failures. A deep check that depends on a shared service can remove every server at once, so keep readiness checks focused and fail open if everything looks unhealthy.

**Q: How do you avoid dropped requests during a deployment?**
Connection draining so in-flight requests finish before a server is removed, plus slow start so a new server isn't overloaded immediately.

**Q: How do you make the load balancer itself highly available?**
Run redundant balancers: an active-passive pair sharing a virtual IP (VRRP), or multiple active nodes via Anycast/ECMP or a managed multi-AZ service.

**Q: How would you load balance across regions?**
Use GeoDNS, latency-based routing, or Anycast to choose the nearest healthy region, then regional balancers within it, with health-based failover between regions.

## Summary

- Load balancers spread traffic, remove single points of failure, and enable scaling and zero-downtime deploys.
- L4 balances connections and is fast; L7 balances HTTP requests and enables content-based routing. gRPC and HTTP/2 need L7 or client-side balancing.
- Algorithms: round-robin and weighted for uniform traffic, least connections or P2C for variable load, consistent hashing for key-based routing.
- Healthy operation depends on good health checks, connection draining, and slow start.
- Prefer stateless servers to sticky sessions, and make the balancer itself redundant.
- Go global with GeoDNS or Anycast on top of regional balancers.

## Continue Learning

- [Scaling](system-design/scaling) — why stateless servers behind a balancer are the foundation.
- [High Availability](system-design/high-availability) — failover, redundancy, and failure domains.
- [DNS](system-design/dns) — how DNS-based and global balancing work.
- [Messaging and Communication](system-design/messaging-and-communication) — gRPC and WebSockets in depth.
- [Circuit Breaker Pattern](system-design/circuit-breaker) — protecting callers from failing backends.
