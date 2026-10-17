---
title: "System Design Case Study — URL Shortener (like Bitly or TinyURL)"
description: "A complete system design walkthrough for building a URL shortening service at scale: requirements, capacity estimation, data model, API design, encoding strategies, caching, and bottleneck resolution."
author: ["name": "Rajendra Pancholi", "email": "rpancholi522@gmail.com"]
created: "2026-04-18"
updated: "2026-04-18"
thumbnail: "/images/url-shortner.png"
tags: [system-design, url-shortener, case-study, scalability, caching, database]
keywords: ["URL shortener system design", "Design Bitly", "Design TinyURL", "System design interview URL shortener", "Key generation service design", "Base62 encoding"]
---

# System Design Case Study — URL Shortener (like Bitly or TinyURL)

A URL shortener converts a long, unwieldy URL into a compact alias that redirects to the original. Services like [Bitly](https://bitly.com) and [TinyURL](https://tinyurl.com) handle billions of redirects daily. This case study walks through designing such a system from scratch, covering every dimension of the system design interview framework.

![URL Shortener System Design](/images/url-shortner.png)

## What is a URL Shortener?

A URL shortener creates a short alias for a long URL. When a user visits the short URL, they are redirected to the original long URL.

**Example:**
- Long URL: `https://karanpratapsingh.com/courses/system-design/url-shortener`
- Short URL: `https://bit.ly/3I71d3o`

## Requirements

### Functional Requirements
- Given a long URL, generate a unique shortened URL (alias)
- When a user visits the short URL, redirect them to the original URL
- Short URLs should expire after a configurable timespan (default: some period)
- Users should be able to optionally specify custom aliases

### Non-Functional Requirements
- High availability — the redirect service must never go down
- Low latency — redirects must be near-instantaneous (< 50ms)
- The system should be horizontally scalable
- Short URLs must be globally unique — no collisions

### Extended Requirements
- Analytics: track redirect counts, user geolocation, device type, referrer
- Rate limiting via API keys to prevent abuse
- URL expiration and cleanup

## Capacity Estimation

Assume a `100:1` read/write ratio (reads are redirects; writes are URL creations).

**Traffic:**
- 100 million new URLs created per month
- 10 billion redirects per month

```
Writes: 100 million / (30 days × 24 hrs × 3600s) ≈ 40 URLs/second
Reads:  10 billion / (30 × 24 × 3600s) ≈ 4,000 redirects/second
```

**Storage (10-year retention):**
```
100 million URLs/month × 12 months × 10 years = 12 billion records
12 billion × 500 bytes/record = ~6 TB
```

**Cache (20% of daily reads):**
```
4,000 reads/second × 86,400 seconds = ~350 million reads/day
20% × 350 million × 500 bytes = ~35 GB/day of cache
```

**High-Level Estimate Summary:**

| Metric | Estimate |
|---|---|
| Write RPS | ~40/s |
| Read RPS | ~4,000/s |
| Incoming bandwidth | ~20 KB/s |
| Outgoing bandwidth | ~2 MB/s |
| Storage (10 years) | ~6 TB |
| Cache memory | ~35 GB/day |

## Data Model

Two primary tables are needed:

### `urls` table
| Column | Type | Description |
|---|---|---|
| `hash` | VARCHAR(7) | Short URL key (indexed) |
| `original_url` | TEXT | Original long URL |
| `user_id` | UUID | Owner of the short URL |
| `expiration` | TIMESTAMP | When the URL expires |
| `created_at` | TIMESTAMP | Creation timestamp |

### `users` table
| Column | Type | Description |
|---|---|---|
| `id` | UUID | Primary key |
| `name` | VARCHAR | User's name |
| `email` | VARCHAR | Email (unique) |
| `api_key` | VARCHAR | API key for rate limiting |
| `created_at` | TIMESTAMP | Account creation timestamp |

### Database Choice

Since the data is not strongly relational and we need horizontal scalability, **NoSQL databases** like Amazon DynamoDB, Apache Cassandra, or MongoDB are preferable. If SQL is used, PostgreSQL or Amazon RDS with read replicas would work.

## API Design

### Create Short URL
```
POST /urls
Authorization: Bearer {api_key}

Body: {
  "original_url": "https://very-long-url.com/path?params=values",
  "custom_alias": "my-link",   // optional
  "expiration": "2027-01-01"   // optional
}

Response 201: {
  "short_url": "https://short.ly/abc123d"
}
```

### Redirect
```
GET /{hash}
Response 301 (Permanent) or 302 (Temporary): Location: {original_url}
```

**Note on redirect code:**
- **301 (Permanent):** Browser caches the redirect — reduces load on our servers but prevents analytics tracking on subsequent clicks from same browser.
- **302 (Temporary):** Every click goes through our server — enables accurate analytics tracking but increases server load.

### Delete URL
```
DELETE /urls/{hash}
Authorization: Bearer {api_key}
Response 200: { "success": true }
```

## URL Encoding Strategies

The core challenge is generating a short, unique, collision-resistant hash for each long URL.

### Approach 1: Base62 Encoding

Base62 uses characters A-Z (26) + a-z (26) + 0-9 (10) = 62 characters.

```
62^7 = ~3.5 trillion unique combinations
```

7 characters gives us enough space for 3.5 trillion URLs — far more than we need for 10 years.

**Implementation:** Generate a random 7-character Base62 string. Check for collision. Retry on collision. Simple but has collision risk at high scale.

### Approach 2: MD5 Hash + Base62 Encode

Apply MD5 to the original URL to get a 128-bit hash, then Base62-encode the first 7 characters.

```
MD5(original_url) → 128-bit hash → Base62 encode → 7-char hash
```

**Problem:** Two different URLs may produce the same first 7 characters (hash collision). Requires collision detection and retry, adding latency and complexity.

### Approach 3: Counter-Based Encoding (Recommended)

A distributed counter generates a unique integer for each URL. The integer is Base62-encoded to produce the hash.

```
Counter(0 → 3.5 trillion) → Base62 encode → 7-char unique hash
```

**Guarantees uniqueness** — each counter value is used exactly once.

**Challenge:** A single counter is a single point of failure and a bottleneck. Solve using **Zookeeper** to assign distinct counter ranges to different server instances:

```
Server A: 0 → 1,000,000
Server B: 1,000,001 → 2,000,000
Server C: 2,000,001 → 3,000,000
```

When a server exhausts its range, Zookeeper assigns a new unused range.

### Approach 4: Key Generation Service (KGS) — Best for Large Scale

A dedicated **Key Generation Service** pre-generates millions of unique keys in advance and stores them in a database. When a new URL is created, the API server requests a pre-generated key from KGS.

**KGS Database:**
- `unused_keys` table: keys ready for use
- `used_keys` table: keys already assigned

When a key is issued, it moves from `unused_keys` to `used_keys`. KGS keeps a small in-memory pool of pre-fetched keys for fast access.

**Advantages:**
- No real-time hash generation — keys are immediately available
- No collision — each key is used exactly once
- Decouples key generation from URL creation

**KGS Capacity:**
```
6-char Base62 keys: 62^6 ≈ 56.8 billion unique keys
Storage: 6 bytes × 56.8 billion = ~390 GB
```

This is a one-time fixed database size — not growing like the main URL database.

## High-Level Design

![URL Shortener Basic Design](/images/url-shortener-basic-design.png)

### Creating a New Short URL

1. Client sends `POST /urls` with the long URL and API key.
2. API server validates the API key (rate limiting check).
3. API server requests a unique key from the **Key Generation Service**.
4. KGS returns a pre-generated key and marks it as used.
5. API server stores the mapping `(key → original_url)` in the database and cache.
6. Returns `201 Created` with the short URL to the client.

### Accessing a Short URL

1. Client navigates to `https://short.ly/abc123d`.
2. API server extracts the hash `abc123d`.
3. Check **cache (Redis)** first for the mapping.
4. **Cache hit:** Return HTTP 302 redirect to `original_url`.
5. **Cache miss:** Query the database, populate cache, return HTTP 302 redirect.
6. If hash not found anywhere: Return HTTP 404.

## Detailed Design

### Caching Strategy

- Use **Redis** with **LRU eviction** policy
- Cache 20% of daily active URLs (~35 GB) — accounts for the 80/20 rule (20% of URLs receive 80% of traffic)
- On cache miss: fetch from database, update cache with new entry

### Data Partitioning

Since 6 TB of data exceeds single-node capacity, use **hash-based sharding** on the URL hash. Consistent hashing minimizes re-distribution when adding/removing database nodes.

### Database Cleanup (URL Expiration)

Two strategies for handling expired URLs:

**Active cleanup:** A background cron job periodically scans for and deletes expired URL records.
**Passive cleanup:** When a user accesses an expired URL, the server detects expiration and removes the entry at that point (lazy deletion). Simpler but allows stale entries to persist longer.

A combination of both is optimal: passive for immediate expired-link handling, active cleanup runs nightly to purge old entries.

### Security

- **Private URLs:** Store allowed user IDs per URL; enforce authorization on access
- **Rate limiting:** API key limits requests per minute/hour; enforced at the API gateway layer
- **Abuse prevention:** Flag URLs pointing to known malicious domains (integrations with Google Safe Browsing API)

### Analytics

Store metadata alongside each URL:
- Redirect count
- Last accessed timestamp
- Per-redirect metadata: country, device type, referrer, timestamp

For real-time analytics, emit events to Kafka and process with Apache Spark or AWS Kinesis.

## Advanced Design — Resolving Bottlenecks

![URL Shortener Advanced Design](/images/url-shortener-advanced-design.png)

**Single points of failure and mitigations:**

| Bottleneck | Mitigation |
|---|---|
| Single API server | Run multiple API server instances behind a load balancer |
| Key Generation Service | Run multiple KGS instances; use standby KGS with replicated key database |
| Database | Master-slave replication; multiple read replicas (read-heavy system); database sharding |
| Cache | Redis cluster with replication; multiple cache nodes |
| Redirect latency | CDN or GeoDNS to serve redirects from nearest regional deployment |

## Summary

| Component | Technology Choice |
|---|---|
| API Servers | Node.js/Go instances behind NGINX load balancer |
| Key Generation | Dedicated KGS with Zookeeper for distributed counter |
| Primary Database | Apache Cassandra or Amazon DynamoDB (NoSQL, horizontally scalable) |
| Cache | Redis cluster with LRU eviction |
| Analytics | Apache Kafka + Apache Spark |
| CDN | Amazon CloudFront for static assets |
| API Gateway | Amazon API Gateway (rate limiting, authentication) |

A URL shortener is deceptively simple on the surface but touches nearly every core system design concept: hashing strategies, distributed key generation, caching, database sharding, read/write scalability, and analytics pipelines.
