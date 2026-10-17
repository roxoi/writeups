---
title: "System Design Case Study — Distributed Rate Limiter"
description: "A complete system design for a distributed API rate limiter: requirements, capacity estimation, placement at the gateway, token bucket on a Redis cluster, atomicity, hot keys, multi-region limits, fail-open vs fail-closed, and safe rollout of rules."
author: ["name": "Rajendra Pancholi", "email": "rpancholi522@gmail.com"]
created: "2026-10-17"
updated: "2026-10-17"
thumbnail: "/images/rate-limiter-design.png"
tags: [system-design, rate-limiter, redis, token-bucket, api-gateway, case-study]
keywords: ["Design a rate limiter", "Distributed rate limiter system design", "Rate limiter Redis cluster", "Token bucket Lua script", "Rate limiter interview question", "API gateway rate limiting"]
---

# System Design Case Study — Distributed Rate Limiter

"Design a rate limiter" is one of the most common system design interview questions, and a component almost every production API needs. A rate limiter decides, for every incoming request, **allow or reject**, based on how many requests the client has already made.

This case study focuses on the **system around the algorithm**: where the limiter lives, how counters are stored and scaled, how races and failures are handled, and how rules are rolled out safely. For the algorithms themselves (token bucket, sliding window, and so on) see [Rate Limiting](system-design/rate-limiting).

![Rate Limiter](/images/rate-limiter-design.png)

## Requirements

### Clarifying Questions (ask these first)

- Is it **client-side or server-side**? (Server-side. We can't trust clients.)
- What do we limit by: **user, API key, IP address, endpoint**, or a combination?
- Is it for a **single server or a distributed fleet**? (Distributed.)
- Do limits need to be **exact**, or is a small error acceptable? (Approximate is fine; a few extra requests never hurt.)
- Should it **inform** users when throttled? (Yes: HTTP 429 with retry information.)

### Functional Requirements

- Limit requests per client identity against configurable **rules** (for example, 100 requests per minute per user for `/search`)
- Support **different limits** per endpoint, per customer tier, and for anonymous traffic
- Allow short **bursts** while enforcing a sustained average rate
- Return a clear rejection (`429 Too Many Requests`) with `Retry-After` and quota headers
- Change rules **without redeploying**

### Non-Functional Requirements

- **Low latency:** the check must add only a few milliseconds to each request
- **High availability:** the limiter must never become the reason the API goes down
- **Scalable:** hundreds of thousands of checks per second
- **Accurate enough:** small over- or under-admission is acceptable
- **Memory efficient:** state for tens of millions of clients
- **Observable:** know who is being blocked and why

## Capacity Estimation

Assumptions: **50 million active users**, **500,000 API requests per second at peak**, and each client tracked under about **2 rules** (a per-user limit and a per-endpoint limit).

```
Limiter operations: ~500,000 checks/second (one per request)

State:
50M users × 2 rules = 100M counters
~150 bytes per key (token bucket fields plus Redis overhead)
100M × 150 B ≈ 15 GB

Throughput per Redis shard (assume ~80,000 scripted ops/s, with headroom):
500,000 / 80,000 ≈ 6.3  →  use 8 shards (≈ 2 GB of data each)
```

| Metric               | Estimate                     |
| -------------------- | ---------------------------- |
| Checks/second (peak) | ~500,000                     |
| Active counters      | ~100 million                 |
| Memory               | ~15 GB                       |
| Redis shards         | ~8 primaries (plus replicas) |

Two takeaways: **memory is modest** (idle keys expire, so only active clients cost memory), and the **real constraint is operations per second**, which we meet by sharding.

## Where Should the Limiter Live?

| Option                                      | How it works                                                                                               | Pros                                                                                          | Cons                                                                                                  |
| ------------------------------------------- | ---------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| **In each app server**                      | A library keeps counters in local memory                                                                   | No network hop, very fast                                                                     | With N servers each sees only part of the traffic, so limits are inaccurate; counters lost on restart |
| **Separate limiter service**                | Services call it over the network for each request                                                         | Central logic and reuse                                                                       | Extra network hop and one more service to keep highly available                                       |
| **At the API gateway / edge (recommended)** | Middleware in the [API gateway](system-design/api-gateway) checks a shared counter store before forwarding | One place for every service; rejects abusive traffic **before** it consumes backend resources | The gateway must be highly available and fast                                                         |

**Decision:** enforce at the gateway, with counters in a shared **Redis cluster**. Individual services may add their own fine-grained limits if needed.

## High-Level Design

```
                      ┌──────────────────────────────────────┐
Client ──► Load       │  API Gateway (stateless, many nodes) │
          Balancer ──►│   1. identify client                 │──► Backend
                      │   2. look up rule (local cache)      │    services
                      │   3. check counter ──────────┐       │    (only if allowed)
                      │   4. allow / reject (429)    │       │
                      └───────────────▲──────────────┼───────┘
                                      │              ▼
                          Rules store │      ┌───────────────┐
                    (DB + config svc) │      │ Redis Cluster │  token buckets,
                       pushed/polled ─┘      │ (sharded +    │  one key per
                                             │  replicated)  │  client per rule
                                             └───────────────┘
```

**Components**

| Component              | Responsibility                                                                                                                 |
| ---------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| **Gateway middleware** | Identifies the client, finds the matching rule, calls the counter store, and allows or rejects                                 |
| **Rules store**        | Holds rule definitions (database or config service), with an admin API to edit them                                            |
| **Local rules cache**  | Each gateway node keeps rules in memory and refreshes them (polling or push), so the hot path never queries the rules database |
| **Redis cluster**      | Holds the counters. Sharded by client key, with replicas for failover                                                          |
| **Metrics pipeline**   | Counts allowed and blocked requests per rule for dashboards and alerts                                                         |

## Rules and Client Identification

### Example Rules

```yaml
- id: search-per-user
  match: { endpoint: '/search', identity: user_id }
  algorithm: token_bucket
  capacity: 20 # burst size
  refill_per_second: 5 # sustained rate (300/min)

- id: anonymous-per-ip
  match: { identity: ip }
  algorithm: token_bucket
  capacity: 60
  refill_per_second: 1

- id: premium-tier
  match: { tier: premium, identity: api_key }
  capacity: 1000
  refill_per_second: 200
```

### Identifying the Client

- **Authenticated requests:** use user ID or API key (most reliable).
- **Anonymous requests:** fall back to IP address. IP limits are weak: many users share one IP behind NAT, and attackers rotate IPs. Trust the client IP only from your own load balancer's header (for example `X-Forwarded-For` as set by your proxy), never from a header the client can forge.
- **Counter key:** combine rule and identity, for example `rl:{user:123}:search`.
- If several rules match, **all must pass**, and the most restrictive one decides the response.

### Rule Distribution

Rules change rarely and are read on every request, so cache them in each gateway node's memory and refresh every few seconds (or on a push notification). Version the rules so a bad edit can be rolled back.

## Algorithm Choice

**Token bucket** is the default for API limiting:

- Allows **bursts** up to the bucket size while enforcing the long-term rate
- Needs only **two values per client** (token count and last-refill time), so it is memory-cheap
- Easy to explain and tune with two parameters

Use a **sliding window counter** instead when you need a strict "N per minute" quota without burst allowance. Details and formulas are in [Rate Limiting](system-design/rate-limiting).

## Request Flow

1. A request reaches a gateway node.
2. The gateway **authenticates** it and identifies the client (user ID, API key, or IP).
3. It finds the matching rule(s) in its **local cache**.
4. It calls Redis to **atomically refill and consume** a token for that client's key (one script call).
5. **Allowed:** forward to the backend, and add quota headers to the response.
6. **Rejected:** return **HTTP 429** immediately, with `Retry-After`. The backend never sees the request.
7. The gateway records an **allowed/blocked metric** for the rule.

**Response headers** (names vary by provider; a standard is still emerging):

```
HTTP/1.1 429 Too Many Requests
Retry-After: 12
X-RateLimit-Limit: 300
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1760000000
```

## Atomicity and Race Conditions

If two gateway nodes handle requests from the same client at the same moment, a naive "read the counter, then write it back" sequence lets both read the same value and both be allowed, so the client exceeds its limit.

**Fix: do the check and update as one atomic step inside Redis.**

- Run the **token bucket as a Lua script**: Redis executes a script atomically, so refill, check, and consume happen without interleaving. (The script is shown in [Rate Limiting](system-design/rate-limiting).)
- **Read the time from Redis itself** (the `TIME` command inside the script) rather than from the gateway servers, so **clock skew** between gateways cannot distort refills.
- **Co-locate keys for the same client.** In Redis Cluster, a script may touch only keys in the same hash slot. Using a hash tag, `rl:{user:123}:search` and `rl:{user:123}:global` land on the same shard, so one script can check **all of a client's rules atomically**.
- Always set a **TTL** on every key (about twice the time to refill a full bucket), so idle clients disappear and memory stays bounded.

## Scaling the Counter Store

- **Shard** by client key using Redis Cluster's hash slots (or consistent hashing). All of one client's keys stay together, and load spreads evenly across clients.
- **Replicas** for each shard allow failover. Replication is asynchronous, so a failover may lose the last few counter updates, which only means a client briefly gets a few extra requests. That is acceptable for rate limiting.
- **Latency:** keep Redis in the **same region and zone** as the gateways, use pooled connections, and set a **short timeout** (a few milliseconds) on limiter calls.

### Hot Keys and Heavy Clients

One very busy client (or a large tenant) sends all its checks to a single key on a single shard.

- **Token leasing / batching:** a gateway node asks Redis for a **batch of tokens** (say 10) for that client and serves the next requests locally, going back to Redis only when the batch runs out. This cuts Redis calls roughly 10x for heavy clients, at the cost of slight over-admission (a few tokens may be held by one node).
- **Local pre-check:** drop obviously abusive clients immediately using a small in-memory deny list with a short TTL, so they stop reaching Redis at all.
- **Per-node sub-limits:** split a very large tenant's limit across several keys.

## Multi-Region Deployment

Users and gateways exist in several regions. Options:

| Approach                            | How it works                                                              | Trade-off                                                                                            |
| ----------------------------------- | ------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------- |
| **Independent regional limiters**   | Each region enforces its own share of the limit (for example 50% each)    | Fast and simple, but the global limit is only approximate, and an uneven traffic split wastes budget |
| **Single global counter**           | All regions call one home region                                          | Exact, but adds cross-region latency and a dependency on one region                                  |
| **Regional with asynchronous sync** | Regions enforce locally and periodically exchange usage to adjust budgets | Good balance; small overshoot is possible                                                            |

**Recommendation:** regional limiters with asynchronous reconciliation. Exact global enforcement is rarely worth the added latency and failure coupling. Keep a client **pinned to one region** where possible.

## Failure Handling: Fail-Open vs Fail-Closed

The limiter is a **protective** component. If it fails, it must not take the API down with it.

| Situation                                                     | Behavior                                                                                                               |
| ------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| Redis slow or unreachable (general APIs)                      | **Fail open**: allow the request, optionally enforce a conservative **local in-memory** limit as a backstop            |
| Redis unreachable (sensitive endpoints: login, OTP, payments) | **Fail closed**: reject, or apply a strict local limit, because letting attackers through is worse than a brief outage |
| Single shard down                                             | Replica is promoted; only the clients on that shard are briefly affected                                               |
| Rules store unavailable                                       | Keep using the last known rules from the local cache                                                                   |

Additional protections:

- Wrap every limiter call in a **timeout and a circuit breaker** (see [Circuit Breaker Pattern](system-design/circuit-breaker)) so a slow Redis never stalls the gateway.
- **Alert** when the limiter falls back, so a silent fail-open doesn't go unnoticed.

## Rolling Out and Operating Rules

Rules are powerful: a mistake can block legitimate customers. Operate them carefully:

- **Shadow (dry-run) mode:** evaluate a new rule and **log** what it would have blocked without actually blocking. Review the numbers, then enforce.
- **Gradual rollout:** enforce for a small percentage of traffic first.
- **Allow lists:** exempt trusted internal services and monitoring.
- **Dashboards and alerts:** allowed vs blocked per rule, top blocked clients, and Redis latency. A sudden spike in blocks may be an attack or a bad rule or a buggy client.
- **Audit trail:** record who changed which rule and when, with easy rollback.

## Abuse Considerations

- **Layer the defenses:** coarse IP limits and DDoS protection at the edge (CDN/WAF), per-user and per-API-key limits at the gateway, and fine-grained limits inside sensitive services.
- **Distributed attacks** use many IPs, so per-IP limits alone don't stop them. Add per-account limits and bot detection.
- **Login and OTP endpoints:** limit per account **and** per IP, with progressive delays or temporary lockouts.
- **Expensive operations:** charge more tokens for costly calls (search, report generation) so limits reflect real cost.
- Return a **uniform** 429 response so attackers can't learn internal rules too precisely.

## Admin API

```
// Create or update a rule
PUT /admin/rules/{ruleId}
Body: { match, algorithm, capacity, refillPerSecond, mode: "enforce" | "shadow" }
Response 200: { ruleId, version }

// List rules and versions
GET /admin/rules

// Inspect a client's current state (debugging)
GET /admin/limits/{identity}?rule={ruleId}
Response: { tokensRemaining, lastRefill }

// If the limiter is run as a separate service
POST /check
Body: { identity, ruleId, cost }
Response: { allowed, remaining, retryAfterMs }
```

## Bottleneck Resolutions

| Bottleneck                             | Solution                                                                              |
| -------------------------------------- | ------------------------------------------------------------------------------------- |
| Counter store throughput (~500k ops/s) | Shard Redis by client key across ~8+ nodes                                            |
| Race conditions between gateways       | Atomic Lua script, single key per client and rule                                     |
| Clock skew                             | Use Redis `TIME` inside the script                                                    |
| Extra latency on every request         | Same-zone Redis, pooled connections, short timeouts, token leasing                    |
| Hot key / heavy client                 | Token batching, local deny list, split limits                                         |
| Redis outage                           | Fail open (or closed for sensitive endpoints) with local backstop and circuit breaker |
| Memory growth                          | TTL on every key; only active clients are stored                                      |
| Rule changes causing incidents         | Shadow mode, gradual rollout, versioning, audit log                                   |
| Multi-region consistency               | Regional limits with asynchronous reconciliation                                      |

## Interview Questions

**Q: Where would you place the rate limiter, and why?**
At the API gateway (or edge middleware), so one policy covers every service and abusive traffic is rejected before it uses backend resources. Counters live in a shared Redis cluster.

**Q: Why not keep counters in each server's memory?**
With many servers, each sees only part of a client's traffic, so the global limit can't be enforced accurately, and counters are lost on restart. A shared store gives one view.

**Q: How do you prevent two gateways from both allowing the last request?**
Make the check-and-decrement atomic with a Lua script (or a single atomic command) on the shard that owns the client's key.

**Q: What happens if Redis goes down?**
Decide per endpoint. Fail open for general traffic (with a local fallback limit) to keep the API available, and fail closed for sensitive endpoints such as login. Use timeouts and a circuit breaker so the limiter can't stall requests.

**Q: How would you handle a client that dominates one Redis key?**
Batch tokens so the gateway reserves several at a time and serves requests locally, keep a short-lived local deny list for clearly abusive clients, or split the limit across multiple keys.

**Q: How do you enforce a limit across multiple regions?**
Use regional limiters with their own share of the budget and reconcile asynchronously; exact global enforcement would add cross-region latency and failure coupling for little benefit.

**Q: How do you roll out a new rule safely?**
Run it in shadow mode first (log only), compare the would-be blocks with real traffic, then enforce gradually, with versioning and quick rollback.

**Q: Why is limiting by IP alone not enough?**
Many legitimate users share one IP behind NAT, and attackers rotate IPs. Prefer authenticated identities, and layer IP limits with per-account limits and edge protection.

## Summary

| Component      | Choice                                                                                        |
| -------------- | --------------------------------------------------------------------------------------------- |
| Placement      | API gateway middleware                                                                        |
| Algorithm      | Token bucket (sliding window counter for strict quotas)                                       |
| Counter store  | Redis Cluster, sharded by client key, replicated                                              |
| Atomicity      | Lua script, time from Redis, keys co-located with hash tags                                   |
| Rules          | Config store with local in-memory cache, versioned, shadow mode                               |
| Hot keys       | Token batching, local deny list                                                               |
| Multi-region   | Regional limits with asynchronous reconciliation                                              |
| Failure policy | Fail open by default, fail closed for sensitive endpoints, circuit breaker and local backstop |
| Response       | HTTP 429, `Retry-After`, quota headers                                                        |
| Observability  | Allowed/blocked metrics per rule, alerts, audit log                                           |

## Continue Learning

- [Rate Limiting](system-design/rate-limiting) — the algorithms and the Redis Lua script in detail.
- [API Gateway](system-design/api-gateway) — where the limiter runs.
- [Caching and CDN](system-design/caching-and-cdn) — Redis as a fast shared store.
- [Circuit Breaker Pattern](system-design/circuit-breaker) — protecting the gateway from a slow limiter backend.
- [High Availability](system-design/high-availability) — failover and redundancy for the counter store.
