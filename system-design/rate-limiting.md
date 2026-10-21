---
title: "Rate Limiting"
description: "How to control request rates: where to enforce limits, the five core algorithms, atomic Redis implementations, distributed rate limiting, fail-open vs fail-closed, and client-side behavior."
author: ["name": "Rajendra Pancholi", "email": "rpancholi522@gmail.com"]
created: "2026-04-18"
updated: "2026-10-17"
thumbnail: "/images/system-design-arch.png"
tags: [rate-limiting, throttling, token-bucket, redis, reliability, system-design]
keywords: ["Rate limiting", "Token bucket algorithm", "Sliding window rate limiter", "Distributed rate limiting", "Leaky bucket", "HTTP 429"]
---

# Rate Limiting

Rate limiting controls the **frequency of requests** a client can make to a service. It prevents abuse, protects downstream systems from overload, keeps usage fair between customers, and controls cost.

## Why Rate Limit?

- **Protect availability:** one buggy or malicious client should not take the service down for everyone.
- **Fairness:** stop a heavy user from starving others (the "noisy neighbor" problem).
- **Security:** slow down brute-force login attempts, credential stuffing, and scraping.
- **Cost control:** limit calls to expensive operations or paid third-party APIs.
- **Business rules:** enforce plan quotas (free tier vs paid tier).

### Rate Limiting vs Related Ideas

| Concept           | Meaning                                                                      |
| ----------------- | ---------------------------------------------------------------------------- |
| **Rate limiting** | Cap requests per client over time (per user, IP, API key)                    |
| **Throttling**    | Slow requests down (delay or queue) instead of rejecting them outright       |
| **Load shedding** | Drop some traffic when the _system_ is overloaded, regardless of who sent it |
| **Backpressure**  | Signal upstream to slow down because a consumer cannot keep up               |

## Where to Enforce Limits

| Location                                            | Notes                                                                                             |
| --------------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| **Client side**                                     | Polite clients limit themselves, but you can never rely on it                                     |
| **[API gateway](system-design/api-gateway) / edge** | The most common place: one policy for all services, rejects early before backends spend resources |
| **Inside a service**                                | Fine-grained limits tied to business logic (for example per-endpoint or per-tenant)               |
| **Per dependency**                                  | Protect your own calls to third-party APIs with their quota                                       |

You can limit by **user ID, API key, IP address, endpoint, tenant, or any combination**. IP-based limits are weak on their own (many users share an IP behind NAT, and attackers rotate IPs), so prefer authenticated identities where possible.

## What the Client Sees

When a client exceeds the limit, return **HTTP 429 Too Many Requests**, ideally with:

- A `Retry-After` header saying how long to wait.
- Rate limit headers showing the limit and remaining quota (commonly `X-RateLimit-Limit`, `X-RateLimit-Remaining`, and a reset time; header names vary by provider).

Clear responses let well-behaved clients back off instead of hammering you.

## Rate Limiting Algorithms

### 1. Token Bucket

A bucket holds up to **N tokens** (the capacity). Tokens are added at a fixed **refill rate**. Each request removes one token; if the bucket is empty, the request is rejected.

- Allows **bursts** up to the bucket size, while enforcing the long-term average rate.
- Two parameters: capacity (burst size) and refill rate (sustained rate).
- Memory-cheap: store only the token count and a timestamp per client.
- Widely used (many API providers and gateways use it).

### 2. Leaky Bucket

Requests enter a **queue** (the bucket) that drains at a **fixed rate**. If the queue is full, new requests are dropped.

- Produces a **smooth, constant output rate**, protecting backends from bursts.
- Bursts are absorbed up to the queue size but then delayed, which adds latency.
- Good when the downstream needs a steady rate.

### 3. Fixed Window Counter

Count requests in fixed time windows (for example, 100 per minute). The counter resets at each window boundary.

- Very simple and cheap (one counter per client per window).
- **Boundary problem:** a client can send 100 requests at 12:00:59 and another 100 at 12:01:00, so **up to 2x the limit** passes within a couple of seconds.

### 4. Sliding Window Log

Store the **timestamp of every request**. To check a request, drop timestamps older than the window and count the rest.

- **Most accurate:** no boundary problem.
- **Memory-heavy:** stores up to `limit` timestamps per client, which is expensive at large scale.

### 5. Sliding Window Counter

A hybrid: keep counters for the current and previous fixed windows, and estimate the sliding count by weighting the previous window:

```text
estimated count = previous_count × (1 − elapsed_fraction) + current_count
```

Example: limit 100 per minute. The previous minute had 80 requests; we are 25% into the current minute with 30 requests so far:

```text
estimated = 80 × 0.75 + 30 = 90  →  under 100, allow
```

- Cheap (two counters) and smooth enough in practice.
- It is an **approximation**, assuming requests were evenly spread in the previous window.

### Choosing an Algorithm

| Algorithm              | Bursts                   | Accuracy       | Memory   | Good for                        |
| ---------------------- | ------------------------ | -------------- | -------- | ------------------------------- |
| Token bucket           | Allowed (up to capacity) | Good           | Very low | General APIs                    |
| Leaky bucket           | Smoothed out             | Good           | Low      | Protecting steady-rate backends |
| Fixed window           | Boundary spikes          | Low            | Very low | Simple, coarse limits           |
| Sliding log            | Exact                    | Exact          | High     | Strict limits at small scale    |
| Sliding window counter | Close to exact           | Good (approx.) | Low      | Most large-scale limiters       |

## Implementing It with Redis

A single instance cannot enforce a **global** limit when you run many app servers, so counters live in a shared store, usually **Redis**, which is fast and offers atomic operations.

### Fixed Window with INCR

```text
key = "rl:{user_id}:{current_minute}"
count = INCR key
if count == 1: EXPIRE key 60
if count > limit: reject
```

Careful: if the app crashes between `INCR` and `EXPIRE`, the key may never expire. Wrap the steps in a Lua script or use a pipeline/transaction so they happen atomically.

### Sliding Window Log with a Sorted Set

Use a sorted set per client, with timestamps as scores:

```text
ZREMRANGEBYSCORE key 0 (now − window)   # drop old entries
ZADD key now unique_request_id          # record this request
ZCARD key                               # count entries in window
EXPIRE key window
reject if the count exceeds the limit
```

Run these in a transaction or Lua script. Note that this counts the request even if rejected, so decide whether rejected requests should consume quota.

### Token Bucket with a Lua Script

Read-modify-write on a bucket across several commands is a **race condition** when requests arrive concurrently. Redis runs a Lua script atomically, so the whole check happens as one step:

```lua
-- KEYS[1] = bucket key
-- ARGV[1] = capacity, ARGV[2] = refill tokens/sec, ARGV[3] = now (ms), ARGV[4] = cost
local data   = redis.call("HMGET", KEYS[1], "tokens", "ts")
local tokens = tonumber(data[1]) or tonumber(ARGV[1])
local ts     = tonumber(data[2]) or tonumber(ARGV[3])

local elapsed = math.max(0, tonumber(ARGV[3]) - ts) / 1000
tokens = math.min(tonumber(ARGV[1]), tokens + elapsed * tonumber(ARGV[2]))

local allowed = tokens >= tonumber(ARGV[4])
if allowed then tokens = tokens - tonumber(ARGV[4]) end

redis.call("HSET", KEYS[1], "tokens", tokens, "ts", ARGV[3])
-- let idle buckets expire (about twice the full-refill time)
redis.call("PEXPIRE", KEYS[1],
           math.ceil(tonumber(ARGV[1]) / tonumber(ARGV[2]) * 1000) * 2)
return allowed and 1 or 0
```

Pass the time in from one source, or read Redis's own clock, so that **clock skew between app servers** does not distort refill.

## Rate Limiting in Distributed Systems

| Challenge                      | Approaches                                                                               |
| ------------------------------ | ---------------------------------------------------------------------------------------- |
| **Race conditions**            | Atomic operations: `INCR`, Lua scripts, transactions                                     |
| **Latency of a central store** | Keep Redis close (same region/zone); use a local in-memory pre-check                     |
| **Central store as a SPOF**    | Replicate and fail over Redis; decide failure behavior (below)                           |
| **Huge scale**                 | Shard limiter keys across Redis nodes (all of one client's keys in one shard)            |
| **Cross-region limits**        | Per-region limits that sum to the global limit, or accept approximate global enforcement |
| **Approximate vs exact**       | Allow each node a local budget and sync periodically: faster, but may overshoot slightly |

### Fail-Open vs Fail-Closed

If the rate limiter store is unreachable, what should happen?

- **Fail-open:** allow requests. Keeps the product working but removes protection. Common for general APIs.
- **Fail-closed:** reject requests. Safer for security-sensitive endpoints (login, payments) but turns a limiter outage into a product outage.

Choose per endpoint, and make the decision explicit.

## Advanced Considerations

- **Multiple tiers of limits:** for example 10 requests per second (burst) _and_ 10,000 per day (quota) per key.
- **Different limits per plan or endpoint:** a cheap read endpoint and an expensive search endpoint should not share a limit.
- **Cost-based limiting:** charge more tokens for expensive operations (`cost` in the script above).
- **Concurrency limits:** cap simultaneous in-flight requests per client, not just rate.
- **Hard vs soft limits:** soft limits warn or slow down first; hard limits reject.
- **Avoid hot keys:** a single huge client can concentrate load on one Redis key.

### Client-Side Behavior

Well-behaved clients should honor `Retry-After`, use **exponential backoff with jitter** when retrying (see [Idempotency and Retries](system-design/messaging-and-communication)), and avoid synchronized retries from many clients at once, which can create a retry storm.

## Trade-offs

| Choice               | You gain                | You give up                    |
| -------------------- | ----------------------- | ------------------------------ |
| Strict, exact limits | Predictable enforcement | Memory and latency cost        |
| Approximate limits   | Speed and low memory    | Slight overshoot or undershoot |
| Central store        | One global view         | Latency and a dependency       |
| Local limits         | No network hop          | Inaccurate global limit        |
| Fail-open            | Availability            | Temporary loss of protection   |

## Real-World Usage

- **Public APIs** (payment, social, developer platforms) apply per-key limits and return 429 with retry information.
- **Login endpoints** use tight per-account and per-IP limits to slow brute-force attacks.
- **Internal protection:** a service limits calls per tenant so one customer's spike does not degrade the others.

For a full walkthrough of building one end to end, see [Design a Rate Limiter](system-design/rate-limiter-design).

## Interview Questions

**Q: Explain the token bucket algorithm.**
A bucket holds up to N tokens and refills at a fixed rate. Each request consumes a token and is rejected if none remain. It allows bursts up to the bucket size while enforcing the long-term average rate.

**Q: What is the problem with a fixed window counter?**
At window boundaries a client can use the full limit at the end of one window and again at the start of the next, sending up to twice the limit in a short time.

**Q: How would you rate limit across many servers?**
Keep counters in a shared store like Redis and use atomic operations or Lua scripts to avoid race conditions. Shard keys by client, and consider local caching and replicas for latency and availability.

**Q: What happens if your Redis rate limiter goes down?**
Decide between fail-open (allow requests, keep availability) and fail-closed (reject, keep protection) per endpoint. For login or payments, fail-closed is often safer; for general APIs, fail-open.

**Q: What status code and headers should you return?**
HTTP 429 Too Many Requests with a `Retry-After` header, and ideally headers showing the limit, remaining quota, and reset time.

**Q: How is rate limiting different from load shedding?**
Rate limiting caps what each client can send. Load shedding drops traffic when the system itself is overloaded, no matter who sent it.

## Summary

- Rate limiting protects availability, ensures fairness, and controls cost.
- Token bucket (bursts allowed) and sliding window counter (cheap and smooth) are the most practical choices.
- Fixed windows are simple but allow boundary bursts; sliding logs are exact but memory-hungry.
- In distributed setups, use a shared store with atomic operations, and decide fail-open vs fail-closed explicitly.
- Return 429 with `Retry-After`, and make clients back off with jitter.

## Continue Learning

- [API Gateway](system-design/api-gateway) — where limits are usually enforced.
- [Caching and CDN](system-design/caching-and-cdn) — Redis as a shared fast store.
- [Messaging and Communication](system-design/messaging-and-communication) — idempotency and safe retries.
- [Design a Rate Limiter](system-design/rate-limiter-design) — a full design walkthrough.
- [Circuit Breaker Pattern](system-design/circuit-breaker) — protecting callers from failing services.
