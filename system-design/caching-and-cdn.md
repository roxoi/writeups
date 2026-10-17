---
title: "Caching and CDN — Speeding Up Systems at Every Layer"
description: "A complete guide to caching strategies, eviction policies, cache invalidation, distributed caches, and Content Delivery Networks (CDN) for building fast, scalable systems."
author: ["name": "Rajendra Pancholi", "email": "rpancholi522@gmail.com"]
created: "2026-04-18"
updated: "2026-04-18"
thumbnail: "/images/caching-cdn.png"
tags: [caching, cdn, system-design, performance, scalability, redis, memcached]
keywords: ["Caching strategies system design", "Cache eviction policies", "CDN explained", "Write-through vs write-back cache", "Redis vs Memcached", "Content Delivery Network"]
---

# Caching and CDN — Speeding Up Systems at Every Layer

> _"There are only two hard things in Computer Science: cache invalidation and naming things."_ — Phil Karlton

Caching is the practice of **storing frequently accessed data in a fast-access storage layer** so that future requests for that data are served faster. It is one of the most powerful and universally applicable performance optimization techniques in system design.

A Content Delivery Network (CDN) is caching taken global — storing static content at geographically distributed edge locations so that users around the world experience low latency.

![Caching Architecture](/images/caching-cdn.png)

## Why Caching Matters

Every time a request hits a backend server and queries a database, it incurs:

- **Network latency** (round trips between services)
- **Database query time** (disk I/O, query parsing, locking)
- **Compute cost** (CPU cycles for computation)

Caching eliminates these costs for repeated access patterns by serving precomputed or previously fetched results from memory — orders of magnitude faster than disk-based storage.

**Caches exploit the principle of locality of reference:** data that was recently requested is likely to be requested again soon.

## Where Caching Happens

A single request can pass through many caches. Knowing each layer helps you decide where to cache:

| Layer                           | Example                  | Typical use                                     |
| ------------------------------- | ------------------------ | ----------------------------------------------- |
| **Browser**                     | HTTP cache, localStorage | Static assets, API responses with cache headers |
| **CDN / edge**                  | CloudFront, Cloudflare   | Static files, videos, cacheable GET responses   |
| **Reverse proxy / API gateway** | NGINX, gateway cache     | Shared responses for many clients               |
| **Application (in-process)**    | HashMap, Caffeine, Guava | Tiny, very hot data per server                  |
| **Distributed cache**           | Redis, Memcached         | Shared data across all app servers              |
| **Database**                    | Buffer pool, query cache | Handled by the database itself                  |

The closer to the user the cache is, the faster the response and the less load on everything behind it.

## Cache Hit and Cache Miss

| Term           | Description                                                                                    |
| -------------- | ---------------------------------------------------------------------------------------------- |
| **Cache Hit**  | Requested data is found in the cache — served immediately without going to the primary store   |
| **Cache Miss** | Requested data is not in the cache — must be fetched from the primary store and added to cache |
| **Cold Cache** | Cache is empty or barely populated — most requests are misses (startup state)                  |
| **Warm Cache** | Cache has been populated with commonly accessed data — mix of hits and misses                  |
| **Hot Cache**  | Highly populated cache with most requests served as hits — optimal steady-state                |

### Hit Ratio Matters More Than You Think

```text
average latency = hit_ratio × cache_latency + (1 − hit_ratio) × database_latency
```

With a 1 ms cache and a 50 ms database:

| Hit ratio | Average latency          |
| --------- | ------------------------ |
| 80%       | 0.8 + 10 = **10.8 ms**   |
| 95%       | 0.95 + 2.5 = **3.45 ms** |
| 99%       | 0.99 + 0.5 = **1.49 ms** |

Going from 80% to 95% cuts latency by about 3x and cuts database load by 4x (from 20% of requests to 5%). That is why cache hit ratio is the key metric to watch.

## Reading Patterns

These patterns describe **who loads data into the cache on a miss**.

### Cache-Aside (Lazy Loading)

The **application** manages the cache. This is the most widely used pattern.

1. The app checks the cache.
2. **Hit:** return the cached value.
3. **Miss:** the app reads from the database, **stores the result in the cache** (with a TTL), and returns it.

```text
value = cache.get(key)
if value is None:
    value = db.query(key)
    cache.set(key, value, ttl=300)
return value
```

**Advantages:** only data that is actually requested gets cached; a cache failure does not break reads (the app falls back to the database); simple.

**Disadvantages:** the first request for each key is slow (a miss); the cache can go stale if the database changes (see invalidation below); the app contains the caching logic.

### Read-Through

The **cache itself** loads data from the database on a miss. The application only talks to the cache, never directly to the database for reads. Caching libraries with a "loader" function and managed services such as DynamoDB Accelerator (DAX) work this way.

**Difference from cache-aside:** the loading logic lives in the cache layer instead of the application. This keeps application code cleaner but ties you to a cache that supports it.

### Refresh-Ahead

The cache **proactively refreshes** popular entries shortly _before_ they expire, so users rarely see a miss on hot data. It reduces latency spikes but wastes effort if the prediction is wrong.

## Write Policies

These describe **how writes are handled** relative to the cache and database.

### Write-Through Cache

Data is written to **both the cache and the database simultaneously**. On every write operation, the cache is updated in real time.

![Write-Through Cache](/images/write-through-cache.png)

**Advantages:**

- Cache is always consistent with the database
- No risk of serving stale data after a write

**Disadvantages:**

- Higher write latency (must wait for both cache and DB write to complete)
- Every write hits the cache even for data that may never be read again (wasted cache space)

**Best for:** Systems where data consistency is critical and write frequency is manageable (e.g., user profile updates).

### Write-Around Cache

Writes go **directly to the database, bypassing the cache**. The cache is only populated when data is read (on cache miss).

![Write-Around Cache](/images/write-around-cache.png)

**Advantages:**

- Cache is not polluted with data that may never be read
- Reduces cache churn for write-heavy workloads

**Disadvantages:**

- Read after write will cause a cache miss (latency spike)
- Not suitable for applications with high read-after-write patterns
- A previously cached copy of the key becomes stale unless it is invalidated or expires

**Best for:** Write-heavy workloads where reads are infrequent or on different data than what was just written.

### Write-Back (Write-Behind) Cache

Data is written **only to the cache**. The write is acknowledged immediately. The cache then asynchronously syncs the data to the database.

![Write-Back Cache](/images/write-back-cache.png)

**Advantages:**

- Extremely low write latency (write to fast memory, not slow disk)
- High write throughput — batches writes to reduce database load

**Disadvantages:**

- **Risk of data loss** — if the cache crashes before syncing, writes are lost
- More complex implementation (async sync mechanism, crash recovery)

**Best for:** Write-intensive data where losing a few seconds of writes is acceptable (e.g., view counters, analytics events, leaderboards). **Do not use it** for money, orders, or anything that must not be lost.

### Comparison

| Policy        | Write goes to         | Consistency        | Write latency | Main risk          |
| ------------- | --------------------- | ------------------ | ------------- | ------------------ |
| Write-through | Cache + DB (together) | Strong             | Higher        | Wasted cache space |
| Write-around  | DB only               | Cache may be stale | Normal        | Miss after write   |
| Write-back    | Cache only, DB later  | Eventual           | Lowest        | Data loss on crash |

In practice, many systems combine **cache-aside for reads** with **write-around plus explicit invalidation** for writes (described next).

## Cache Invalidation and Consistency

Cache invalidation — ensuring cached data doesn't become stale — is the hardest part of caching. The common tools are:

### 1. TTL (Time To Live)

Every cached entry gets an **expiry time**. After the TTL, the entry is removed and the next read reloads fresh data.

- Simple and a safety net for every other strategy: even if invalidation fails, stale data disappears eventually.
- **Short TTL:** fresher data, more misses and database load.
- **Long TTL:** better hit ratio, staler data.
- Choose TTL based on how stale the data is allowed to be (a product description: minutes to hours; a stock count: seconds or no caching).

### 2. Invalidate on Write

When data changes, **update the database first, then delete the cache key**. The next read repopulates it.

```text
db.update(key, new_value)
cache.delete(key)
```

**Why delete instead of updating the cache?** Updating the cache on every write wastes effort on data that may never be read, and concurrent writers can leave the cache with an older value than the database. Deleting is simpler and safer.

**Why database first, then cache?** If you delete the cache first, a concurrent reader can miss, read the _old_ value from the database, and put it back in the cache before your database update finishes, leaving stale data cached.

### 3. A Race That Still Exists

Even with "update database, then delete cache", there is a rare race:

1. Reader A misses the cache and reads the **old** value from the database.
2. Writer B updates the database and deletes the cache key.
3. Reader A (slow) now writes the **old** value into the cache.

The cache is stale until the TTL expires. Mitigations:

- **Always set a TTL** so staleness is bounded.
- **Delayed double delete:** delete the key again shortly after the write.
- **Event-based invalidation:** read the database's change log (change data capture) and delete keys from there, so invalidation follows the actual committed change.
- **Versioned keys** (include a version number in the key) so old and new values never collide.

### 4. Choose a Consistency Level on Purpose

Caching means accepting **eventual consistency** for cached reads. For each piece of data, decide: how stale is acceptable? Some data (a user's own recent post, an account balance before payment) should bypass the cache or read from the primary.

## Cache Eviction Policies

When a cache reaches capacity, it must evict existing entries to make room for new ones. The eviction policy determines which entries are removed.

| Policy                          | Description                                           | Best For                                                    |
| ------------------------------- | ----------------------------------------------------- | ----------------------------------------------------------- |
| **LRU** (Least Recently Used)   | Evicts the entry that was accessed least recently     | General-purpose; temporal locality workloads                |
| **LFU** (Least Frequently Used) | Evicts the entry with the fewest total accesses       | Workloads where some items are always popular               |
| **MRU** (Most Recently Used)    | Evicts the most recently accessed entry               | Specific access patterns (e.g., video scrubbing)            |
| **FIFO** (First In First Out)   | Evicts the oldest inserted entry regardless of access | Simple queue-like caches                                    |
| **LIFO** (Last In First Out)    | Evicts the most recently inserted entry               | Rarely used; specific stack-like patterns                   |
| **Random Replacement**          | Randomly selects and evicts an entry                  | Simple to implement; acceptable for uniform access patterns |

**LRU is the most widely used** policy in general-purpose caches because it efficiently exploits temporal locality — recently accessed data is more likely to be accessed again.

**Redis note:** by default Redis does **not evict anything** (`noeviction`) — writes fail when memory is full. For a cache you must set `maxmemory` and a policy such as `allkeys-lru` or `allkeys-lfu`. Redis uses an approximated LRU/LFU (it samples keys) rather than an exact one.

**Eviction vs expiry:** _expiry_ removes entries whose TTL ran out; _eviction_ removes entries because memory is full. A healthy cache mostly expires entries; heavy eviction means the cache is too small.

## Common Cache Problems

| Problem                              | What happens                                                                                              | Typical fixes                                                                                                                                                           |
| ------------------------------------ | --------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Cache stampede** (thundering herd) | A hot key expires and hundreds of concurrent requests all miss at once and hit the database together      | **Request coalescing / lock:** only one request rebuilds the value while others wait or get stale data; **probabilistic early refresh**; serve stale while revalidating |
| **Cache avalanche**                  | Many keys expire at the same moment (or the cache goes down), sending a flood to the database             | Add **random jitter** to TTLs; stagger warm-ups; replicate the cache; use a second cache layer; rate-limit and protect the database                                     |
| **Cache penetration**                | Requests for keys that **do not exist** always miss and always hit the database (accidental or malicious) | **Cache the "not found" result** briefly (negative caching); validate input; use a **Bloom filter** of valid keys to reject impossible ones early                       |
| **Hot key**                          | One key gets so much traffic that a single cache node becomes the bottleneck                              | Keep a small **local in-process copy**; **replicate the key** under several names (`key#1`, `key#2`) and read a random one                                              |
| **Stale data**                       | Cache differs from the database                                                                           | TTL, invalidate on write, event-based invalidation                                                                                                                      |

A rate limiter in front of the database is a good last line of defense — see [Rate Limiting](system-design/rate-limiting).

## Local vs Distributed Cache (L1/L2/L3)

Similar to CPU caches, application caches operate in levels:

```
L1: Application in-process cache (fastest, smallest — e.g., HashMap in application memory)
L2: Distributed cache (fast, shared across instances — e.g., Redis cluster)
L3: Database/origin (slowest, authoritative source of truth)
```

A request checks L1 first. On miss, checks L2. On miss, fetches from L3 and populates both L2 and L1 for future requests.

|             | In-process (L1)                                          | Distributed (L2)             |
| ----------- | -------------------------------------------------------- | ---------------------------- |
| Speed       | Fastest (no network)                                     | Fast (one network hop)       |
| Size        | Limited by each server's memory                          | Can scale to many nodes      |
| Consistency | **Each server has its own copy**, so copies can disagree | One shared copy              |
| Best for    | Very hot, small, rarely changing data (use a short TTL)  | Most shared application data |

## Distributed Cache

A single cache server is a single point of failure and has limited capacity. **Distributed caches** spread cached data across multiple nodes.

![Distributed Cache](/images/distributed-cache.png)

Key characteristics:

- **Horizontal scalability** — add nodes to increase total cache capacity
- **Consistent hashing** — determines which node stores which key (minimizes cache reshuffling when nodes join/leave). With a plain `hash(key) % N`, adding one node remaps almost every key and causes a mass of misses; consistent hashing remaps only about `1/N` of keys.
- **Replication** — data copied to multiple nodes for fault tolerance

### Redis vs Memcached

|                        | Redis                                                                               | Memcached                        |
| ---------------------- | ----------------------------------------------------------------------------------- | -------------------------------- |
| Data types             | Strings, hashes, lists, sets, sorted sets, streams, and more                        | Simple key-value (strings/blobs) |
| Persistence            | Optional: RDB snapshots, AOF log                                                    | None                             |
| Replication / failover | Built-in replicas; Sentinel for automatic failover                                  | None built in                    |
| Clustering             | Redis Cluster shards keys across 16,384 hash slots                                  | Client-side sharding             |
| Threading              | Commands run on a single thread (each command is atomic), with optional I/O threads | Multi-threaded                   |
| Extras                 | Pub/sub, Lua scripting, atomic counters, TTL per key                                | Very simple and fast             |
| Typical use            | Caching **and** data structures (rate limiters, leaderboards, sessions, locks)      | Pure, simple caching             |

Rule of thumb: pick **Redis** unless you need only the simplest possible cache. Remember that a cache is **not** a source of truth — if you rely on Redis persistence for important data, treat it as a database and plan accordingly.

**Widely used distributed cache technologies:**

| Technology                                               | Notes                                                                                                      |
| -------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| [Redis](https://redis.io)                                | In-memory data structure store; supports strings, hashes, lists, sets, sorted sets, pub/sub, Lua scripting |
| [Memcached](https://memcached.org)                       | Simple key-value store; extremely fast; no persistence or advanced data types                              |
| [Amazon ElastiCache](https://aws.amazon.com/elasticache) | Managed Redis or Memcached on AWS                                                                          |
| [Aerospike](https://aerospike.com)                       | Hybrid in-memory/SSD store; exceptional for real-time bidding                                              |

## Sizing and Monitoring

**Sizing with the 80/20 rule:** roughly 20% of the data serves about 80% of requests, so you often do not need to cache everything.

Example: if users request about 50 million distinct objects per day at 1 KB each, that is about 50 GB of data per day. Caching the hottest 20% needs about **10 GB**, which fits comfortably on one cache node. Add headroom for key overhead and replication.

**Metrics to watch:**

- **Hit ratio** — the headline number; investigate if it drops
- **Eviction rate** — high means the cache is too small
- **Latency** (p95/p99) of cache calls
- **Memory usage** and **key count**
- **Database load** — it should fall when the cache is healthy

**Warming:** after a restart, a cold cache sends a burst of misses to the database. Pre-load the hottest keys or bring nodes back gradually.

## When NOT to Use Caching

Caching is not universally beneficial. Avoid caching when:

- **Data changes very frequently** — cache becomes stale immediately; cache miss rate approaches 100%
- **Requests have very low repetition** — random access patterns provide no cache hit benefit
- **Cache access is as slow as primary store access** — no latency benefit
- **Strong consistency is required** — cached data may be stale; if you cannot tolerate eventual consistency, avoid caching that data

## Content Delivery Network (CDN)

A CDN is a **geographically distributed network of servers (edge nodes)** that caches and delivers static content close to the user's physical location.

Without CDN: A user in Mumbai accessing a server in New York experiences roughly 200 ms round-trip latency.
With CDN: The same user accesses an edge node in Mumbai — round-trip drops to roughly 10–20 ms.

![CDN Architecture](/images/cdn.png)

### How CDN Works

1. The origin server holds the canonical version of content (HTML, CSS, JS, images, videos).
2. The CDN deploys **edge servers** in dozens to hundreds of global locations (PoPs — Points of Presence).
3. When a user requests a resource, they are routed to the nearest edge server, either by **DNS-based geo routing** or by **Anycast** (one IP announced from many locations so the network delivers you to the closest).
4. If the edge server has the content cached, it serves it directly (cache hit).
5. If not (cache miss), the edge server fetches from origin, caches locally, and serves the user.
6. Subsequent requests from nearby users are served from the edge cache.

![CDN Map](/images/cdn-map.png)

### Push CDN vs Pull CDN

| Type         | Mechanism                                                     | Best For                                                                      |
| ------------ | ------------------------------------------------------------- | ----------------------------------------------------------------------------- |
| **Push CDN** | Content is explicitly pushed to CDN when created or updated   | Low-traffic sites; content that changes infrequently; full control over cache |
| **Pull CDN** | CDN fetches content from origin on first request, then caches | High-traffic sites; content that changes often; self-managing cache           |

Most modern setups use **pull CDNs** because they need little management.

### Controlling CDN and Browser Caching with HTTP Headers

The `Cache-Control` response header tells browsers and CDNs how to cache:

| Directive                  | Meaning                                                                        |
| -------------------------- | ------------------------------------------------------------------------------ |
| `max-age=N`                | Cache for N seconds (browser and shared caches)                                |
| `s-maxage=N`               | Same, but for **shared caches** such as CDNs (overrides `max-age` there)       |
| `public` / `private`       | `private` means only the user's browser may cache it (e.g., personalized data) |
| `no-cache`                 | Cache it, but **revalidate** with the origin before reusing                    |
| `no-store`                 | Never store it (sensitive data)                                                |
| `immutable`                | The file will never change, so do not revalidate                               |
| `stale-while-revalidate=N` | Serve a stale copy while fetching a fresh one in the background                |

**Revalidation:** the origin sends an `ETag` (a version fingerprint). The client later asks "has it changed?" with `If-None-Match`, and the origin replies `304 Not Modified` with no body if nothing changed — saving bandwidth.

### CDN Invalidation: Version Your Files

Purging content from hundreds of edge nodes is slow and can be costly. The cleanest approach is to **avoid needing to purge**:

- Put a **content hash in the filename** (`app.3f9a1c.js`) and cache it for a very long time (`max-age=31536000, immutable`).
- When the file changes, the filename changes, so every user fetches the new file immediately.
- Keep your HTML short-lived (or `no-cache`) so it always points at the latest filenames.

For content that can't be versioned, use the CDN's purge API or short TTLs.

### CDN Benefits

- **Reduced latency** — users served from geographically proximate edge servers
- **Reduced origin load** — edge servers absorb the majority of requests
- **DDoS protection** — CDN's distributed nature absorbs volumetric attacks
- **High availability** — content remains accessible even if origin is temporarily down

### CDN Disadvantages

- **Cost** — CDN bandwidth fees can be significant for high-traffic services
- **Cache invalidation complexity** — purging stale content across all edge nodes requires coordination
- **Geographic restrictions** — CDN providers may not have coverage in certain regions
- **Do not cache personalized or private responses** on a shared cache by mistake

### CDN Use Cases in System Design

| Use Case         | Notes                                                                   |
| ---------------- | ----------------------------------------------------------------------- |
| Static assets    | JS, CSS, fonts, images — ideal CDN content; rarely changes              |
| Video streaming  | Large files; CDN dramatically reduces origin bandwidth cost             |
| API acceleration | Some CDNs can cache GET API responses at the edge                       |
| Dynamic content  | Advanced CDNs support edge-side logic (Cloudflare Workers, Lambda@Edge) |

### Popular CDN Providers

| Provider                                               | Notes                                                              |
| ------------------------------------------------------ | ------------------------------------------------------------------ |
| [Amazon CloudFront](https://aws.amazon.com/cloudfront) | Integrated with AWS services; Lambda@Edge for compute              |
| [Cloudflare CDN](https://www.cloudflare.com/cdn)       | Global network; DDoS protection included; Workers for edge compute |
| [Google Cloud CDN](https://cloud.google.com/cdn)       | Integrated with GCP backends                                       |
| [Fastly](https://www.fastly.com)                       | Fast, near-instant cache purging; programmable edge                |

## Caching in System Design Interviews

When discussing caching in interviews, be prepared to address:

1. **What data to cache** — not everything should be cached; prioritize high-read, low-write data
2. **Pattern** — cache-aside for reads in most designs; say why
3. **Write and invalidation strategy** — write-through, write-around, or write-back; TTL; invalidate on write
4. **Eviction policy** — typically LRU for general workloads
5. **Cache size estimation** — use the 80/20 rule: cache 20% of data to serve ~80% of requests
6. **Cache layer placement** — browser, CDN, in-process, or distributed
7. **Consistency guarantees** — what staleness is acceptable? How do you handle stale reads?
8. **Failure modes** — stampede, avalanche, penetration, hot keys, and what happens if the cache is down

### Interview Questions

**Q: How does cache-aside work, and what are its downsides?**
The app checks the cache, on a miss reads the database and stores the result with a TTL. Downsides: the first request is slow, data can go stale, and the caching logic lives in application code.

**Q: How do you keep the cache consistent with the database?**
Set a TTL as a safety net, update the database first and then delete the cache key, and for stronger guarantees use change-data-capture-based invalidation. Accept eventual consistency for cached reads and bypass the cache for data that must be fresh.

**Q: What is a cache stampede and how do you prevent it?**
A hot key expires and many concurrent requests miss and hit the database at once. Prevent it with request coalescing (one rebuilder, others wait), serving stale data while refreshing, probabilistic early refresh, and jittered TTLs.

**Q: What is the difference between cache avalanche and cache penetration?**
Avalanche is many keys expiring together (or the cache failing) so a flood reaches the database. Penetration is repeated requests for keys that do not exist, so every request misses. Fix avalanche with TTL jitter and redundancy; fix penetration with negative caching or a Bloom filter.

**Q: Redis or Memcached?**
Redis for most cases: rich data types, replication, persistence option, clustering, atomic operations. Memcached only for the simplest multi-threaded key-value caching.

**Q: How would you cache a URL shortener's redirects?**
Cache-aside with `shortCode → longUrl` in Redis with an LRU policy, since reads vastly outnumber writes, and a long TTL because mappings rarely change. See [Design a URL Shortener](system-design/url-shortener-design).

## Summary

- Caching trades **freshness for speed**. Cache hit ratio is the metric that decides whether it works.
- Use **cache-aside** for reads and **TTL plus invalidate-on-write** for consistency. Write-through, write-around, and write-back control how writes interact with the cache.
- Always set a TTL, with random jitter, and plan for stampede, avalanche, penetration, and hot keys.
- Redis is the default distributed cache; set `maxmemory` and an eviction policy, and treat the cache as disposable, not a source of truth.
- CDNs cache static content at the edge. Version your filenames and use `Cache-Control` headers instead of relying on purges.

## Continue Learning

- [Databases](system-design/databases) — what sits behind the cache.
- [Load Balancing](system-design/load-balancing) — spreading traffic across the servers in front.
- [Rate Limiting](system-design/rate-limiting) — protecting the database when the cache misses.
- [Scaling](system-design/scaling) — where caching fits into growing a system.
- [Design a URL Shortener](system-design/url-shortener-design) — caching applied to a real design.
