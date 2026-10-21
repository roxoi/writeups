---
title: "System Design Case Study — Search Autocomplete (Typeahead Suggestions)"
description: "A complete system design for a search autocomplete service: prefix lookup with tries and precomputed top-K, the offline ranking pipeline, serving architecture, caching at every layer, client-side debouncing, freshness and trending, personalization, and typo tolerance."
author: ["name": "Rajendra Pancholi", "email": "rpancholi522@gmail.com"]
created: "2026-10-17"
updated: "2026-10-17"
thumbnail: "/images/caching-cdn.png"
tags: [system-design, autocomplete, typeahead, trie, search, caching, case-study]
keywords: ["Design search autocomplete", "Typeahead system design", "Trie top-K suggestions", "Autocomplete architecture", "Prefix search system design", "System design interview autocomplete"]
---

# System Design Case Study — Search Autocomplete (Typeahead Suggestions)

As you type into a search box, a list of suggestions appears after every keystroke. Autocomplete (also called typeahead) saves users typing, guides them to popular queries, and corrects mistakes. It looks small, but it is one of the **most latency-sensitive and highest-traffic** services in a search product: every search triggers several suggestion requests, and each must return in a **few tens of milliseconds** to feel instant.

![Search Autocomplete](/images/caching-cdn.png)

---

## Requirements

### Functional Requirements
- Given a **prefix** the user has typed, return the **top K suggestions** (typically 5–10)
- Suggestions are ranked by **popularity and relevance** (and optionally personalization and location)
- Support **multiple languages**
- **Filter** offensive, illegal, or unwanted suggestions
- Reflect **trending** queries reasonably quickly

### Non-Functional Requirements
- **Very low latency:** responses well under 100 ms end to end (the service itself should answer in a few milliseconds)
- **High availability:** if suggestions fail, search must still work
- **Scalable** to a huge request rate
- **Eventually fresh:** new popular queries may take minutes to hours to appear (instant updates are not required)

---

## Capacity Estimation

Assumptions: **100 million daily active users**, **10 searches per user per day**, and about **4 suggestion requests per search** (after the client reduces requests, see below).

```
Suggestion requests/day: 100M × 10 × 4 = 4 billion
Average: 4B / 86,400 ≈ 46,000 requests/second
Peak (3x): ≈ 140,000 requests/second

Data: assume the 100 million most popular queries are kept
100M × ~25 bytes ≈ 2.5 GB of raw query text

Top-K cache per prefix (K = 10) multiplies this several times → tens of GB,
which fits in memory when sharded across a few nodes
```

| Metric | Estimate |
|---|---|
| Suggestion requests | ~46,000/s average, ~140,000/s peak |
| Distinct queries kept | ~100 million |
| Raw text | ~2.5 GB |
| With top-K per prefix | tens of GB (in memory, sharded) |

Takeaways: the **data is small** (fits in memory), but the **request rate is very high** and the latency budget is tiny. Everything is designed around **precomputation and caching**.

---

## Why Not Query the Database?

```sql
SELECT query FROM searches WHERE query LIKE 'pyth%' ORDER BY frequency DESC LIMIT 10;
```

Even with an index on `query`, this is too slow and too heavy at 140,000 requests/second, because the database must collect and sort all matches on every keystroke. The key insight: **suggestions for a given prefix change slowly, so compute them in advance** and make serving a simple lookup.

---

## Data Structures

### Option 1: Trie (Prefix Tree)

A **trie** stores strings as paths, where each node represents a character, so all strings sharing a prefix share a path.

```
          (root)
          /     \
         p       j
         |       |
         y       a
         |       |
         t       v
         |       |
         h       a
        / \
       o   (python)
       |
       n  (python, count=9M)
```

To find suggestions for "pyth", walk down `p → y → t → h`, then you need the best queries **under** that node.

**Naive approach:** traverse the entire subtree under the node and sort by frequency. Too slow for popular short prefixes (the subtree for "a" is huge).

**Optimization: store the top K at every node.** Each node keeps the **precomputed top-K suggestions** for its prefix. A lookup is then just: walk the prefix (O(length of the prefix)) and return the stored list, with **no traversal or sorting at request time**.

**Cost:** more memory, because every node stores K entries. Reduce it by:
- Storing **query IDs or pointers** rather than copies of the strings
- **Compressing** the trie (a radix tree merges single-child chains)
- Limiting depth (suggest only after 1–2 characters; precompute up to, say, 15–20 characters)
- Keeping only queries above a minimum frequency

### Option 2: Prefix → Top-K Key-Value Map

Even simpler to operate: for **every prefix** of every popular query, store `prefix → [top K suggestions]` in a key-value store (Redis or an in-memory map).

```
"p"     → [python, pizza, pinterest, ...]
"py"    → [python, pytorch, pycharm, ...]
"pyt"   → [python, pytorch, pytest, ...]
```

Lookup is a single key access: O(1). It uses more space than a compressed trie but is very easy to build, shard, replicate, and cache. Many production systems use this form.

### Option 3: Search Engine Features

Elasticsearch and OpenSearch have a **completion suggester** that uses an efficient in-memory structure (a finite-state transducer) for prefix lookups with weights, and support fuzzy matching. It is a good choice when you already run a search cluster, and the data size is moderate.

### Comparison

| Option | Lookup | Memory | Build and operate | Best for |
|---|---|---|---|---|
| Trie with top-K per node | O(prefix length) | Moderate with compression | Custom code | Maximum efficiency |
| Prefix → top-K map | O(1) | Higher | Simple, easy to shard | Most production systems |
| Search engine suggester | Fast | Moderate | Managed, feature-rich (fuzzy) | Existing search infrastructure |

---

## Architecture

The system has two separate halves: an **offline (or near-line) pipeline** that builds the suggestion data, and an **online serving path** that just reads it.

```
                   ONLINE (serving)
User types ─► CDN / browser cache ─► Load Balancer ─► Suggestion Service ─► in-memory trie / KV store
                                                         (stateless, sharded replicas)

                   OFFLINE / NEAR-LINE (building)
Search logs ─► Kafka ─► Aggregation job (counts, time decay) ─► Ranker ─► Build top-K per prefix
                                                                              │
                                                              versioned snapshot ─► deploy to serving nodes
                                    Trending stream (small, fast updates) ───────► merged at serving time
```

---

## The Data Pipeline

### 1. Collect Query Logs

Every completed search is logged as an event (query text, timestamp, language, country) and sent through **Kafka** (see [Messaging and Communication](system-design/messaging-and-communication)). Use only **completed** searches, not every partial keystroke.

### 2. Aggregate Frequencies

A batch or streaming job (Spark or Flink) counts how often each query appears over a window (for example the last few days to weeks).

**Recency matters.** A query popular last year but dead now should fade. Use **time decay**, for example an exponentially weighted score:

```text
score = Σ (count_in_day × decay^(days_ago))      with decay like 0.9–0.95 per day
```

### 3. Clean and Filter

- **Normalize:** lowercase, trim whitespace, collapse spacing and punctuation, handle Unicode forms consistently.
- **Remove** offensive, illegal, private, or low-quality queries using blocklists and classifiers (suggestions are highly visible, so this matters a lot for brand safety).
- **Drop** very rare queries (they add memory but little value, and might expose personal information).

### 4. Build the Top-K Structure

For each query, add its score to every prefix. For each prefix keep the top K. This can be done by a distributed job: for every query emit `(prefix, query, score)` for each prefix, group by prefix, and keep the K highest.

### 5. Publish a Versioned Snapshot

Write the new trie or key-value snapshot to storage, then have serving nodes **load the new version and swap it atomically** (like blue-green: the old version stays until the new one is fully loaded). Run it on a schedule (hourly or daily). Versioning makes **rollback** easy if a bad build occurs.

### 6. Freshness for Trending Queries

A daily rebuild is too slow for breaking news. Add a **trending fast path**:

- A streaming job tracks queries whose counts are **spiking** relative to their baseline in the last few minutes.
- These are written to a **small, separate trending index**.
- At request time, the suggestion service **merges** trending items into the base top-K (with a boost), so a new term can appear within minutes without rebuilding the whole structure.

---

## Serving Architecture

### Request Flow

1. The client sends `GET /suggest?q=pyth&lang=en&limit=8`.
2. A cache (browser, CDN, or edge) may answer immediately.
3. Otherwise the **Suggestion Service** looks up the prefix in its in-memory structure (and merges trending and personal items), then returns the list.

```
GET /suggest?q=pyth&limit=8&lang=en
Response 200:
Cache-Control: public, max-age=60
[ { "text": "python tutorial", "score": 0.97 },
  { "text": "python download", "score": 0.91 }, ... ]
```

### Scaling the Serving Tier

- **Stateless service** nodes holding the data **in memory**; add replicas for throughput.
- If the data doesn't fit on one node, **shard by prefix**: for example by the first one or two characters or by a hash of the prefix, using consistent hashing. Beware **uneven load**: prefixes such as "a", "s", and "th" are much hotter than "zq", so shard by traffic and not only by key range, and replicate hot shards.
- Replicate across availability zones, and across regions near users.
- Keep hot prefixes (short ones) in the **cache** to absorb the largest share of traffic.

### Caching at Every Layer

Short prefixes are repeated by huge numbers of users, so caching is extremely effective. See [Caching and CDN](system-design/caching-and-cdn).

| Layer | What it does |
|---|---|
| **Browser** | Caches responses with `Cache-Control` so retyping or backspacing reuses results; the client also keeps its own map of prefix → results |
| **CDN / edge** | Caches popular prefixes for tens of seconds to minutes |
| **Service memory / Redis** | Holds the top-K lists |

A TTL of a minute or so is fine because suggestions don't need to be real-time exact.

---

## Client-Side Optimizations

The client matters as much as the server:

- **Debounce:** wait about 100–200 ms after the last keystroke before sending a request, so fast typing sends fewer requests. This is what keeps the estimate at ~4 requests per search.
- **Cancel stale requests:** if the user types another character, abandon the previous request, and ignore out-of-order responses (show only the response matching the current text).
- **Start at a minimum length** (1–2 characters, depending on the product).
- **Local cache** of recent prefix results and the user's own recent searches.
- **Prefetch** the likely next-character results for popular prefixes when cheap.
- Keep **payloads tiny** (text and score only).

---

## Ranking, Personalization, and Context

Base ranking is **popularity with recency**, but real products blend more signals:

- **Personalization:** the user's own search history and interests (store a small per-user list and merge it into the global results).
- **Location and language:** suggestions for "pizza" in Mumbai differ from those in Chicago. Maintain separate structures (or boosts) per language and region.
- **Context:** the category or page the user is in (for example searching within a shop or within a video platform).
- **Freshness:** trending boosts, as above.
- **Diversity and deduplication:** avoid near-identical suggestions ("python tutorial" and "python tutorials") dominating all slots.

Keep personalization **separate** from the global index: merge a small personal list at request time instead of building a structure per user.

---

## Typo Tolerance and "Did You Mean"

Prefix matching fails if the user mistypes early ("pyhton"). Options:

- **Fuzzy matching** with edit-distance limits (for example one or two edits), using **n-gram indexes**, **Levenshtein automata**, or the fuzzy options of a search engine's suggester.
- A separate **spell-correction** service (using query logs: common corrections of frequent mistakes) that suggests "Did you mean: python".
- Apply fuzzy matching only when exact-prefix results are few, since it is more expensive.

---

## Availability and Failure Handling

- **Degrade gracefully:** if the suggestion service is slow or down, the search box simply shows no suggestions and still lets users search. Use short client timeouts (for example 200–300 ms) and don't block the UI.
- **Fallback to stale data:** serving nodes keep the last good snapshot if a new build fails to load.
- Use a **circuit breaker** and rate limiting to protect the service from traffic spikes (see [Circuit Breaker Pattern](system-design/circuit-breaker) and [Rate Limiting](system-design/rate-limiting)).
- Roll out new snapshots to a few nodes first (a canary) and compare quality metrics before the rest.

---

## Quality and Safety

- **Measure quality:** click-through rate on suggestions, the fraction of searches that use a suggestion, keystrokes saved, and latency percentiles.
- **Safety:** continuously filter inappropriate suggestions; provide a way to **remove** a specific suggestion quickly (a blocklist that is applied at serving time, not only at build time).
- **Privacy:** don't suggest queries that may contain personal data (rare queries, long unique strings), and anonymize logs.
- Protect against **manipulation**: bots can spam queries to push suggestions up, so deduplicate by user and IP and require a minimum number of distinct users before a query becomes eligible.

---

## Bottleneck Resolutions

| Bottleneck | Solution |
|---|---|
| Very high request rate | Debouncing, multiple cache layers, stateless replicated services |
| Latency per request | Precomputed top-K per prefix held in memory: a lookup, not a computation |
| Trie too large for one node | Compress, limit depth and rare queries, shard by prefix traffic |
| Hot prefixes | Replicate hot shards, cache short prefixes at the edge |
| Staleness | Frequent rebuilds plus a small streaming trending index merged at serve time |
| Bad or offensive suggestions | Filters, blocklists applied at build time and serve time |
| Failed build or bad data | Versioned snapshots, atomic swap, quick rollback |
| Out-of-order responses on the client | Cancel stale requests and match the response to the current text |

---

## Interview Questions

**Q: How do you make autocomplete fast?**
Precompute the top-K suggestions for every prefix offline and keep them in memory, so serving is a single lookup. Add caching at the browser, CDN, and service layers, and debounce on the client.

**Q: Why store top-K at each trie node?**
Without it, you must traverse the whole subtree under the prefix and sort on every request, which is too slow. Storing the top K makes lookup O(prefix length) at the cost of extra memory.

**Q: How do you keep suggestions fresh?**
Rebuild the main structure on a schedule from aggregated logs with time decay, and add a small streaming trending index that is merged at request time so new popular queries appear within minutes.

**Q: How would you shard the data?**
By prefix (consistent hashing on the first characters or a hash of the prefix), taking traffic skew into account: replicate hot prefixes more.

**Q: How do you handle personalization?**
Keep the global index shared and merge a small per-user list (recent searches and interests) into the results at request time, rather than storing a structure per user.

**Q: How do you reduce load from the client?**
Debounce keystrokes, cancel in-flight requests, start suggesting after a minimum length, cache results locally, and use `Cache-Control` headers.

**Q: What if the service goes down?**
Suggestions are optional: the UI should time out quickly and continue working without them. Servers keep the last good snapshot, and a circuit breaker prevents overload.

**Q: How do you prevent offensive or manipulated suggestions?**
Filter at build time and serve time with blocklists and classifiers, require a minimum number of distinct users per query, and dedupe bot traffic.

---

## Summary

| Component | Choice |
|---|---|
| Lookup structure | Prefix → top-K map or compressed trie with top-K per node |
| Ranking | Frequency with time decay, plus location, language, and personalization |
| Pipeline | Query logs → Kafka → aggregation job → top-K build → versioned snapshot |
| Freshness | Scheduled rebuild plus a streaming trending index merged at serve time |
| Serving | Stateless in-memory nodes, sharded by prefix with replicated hot shards |
| Caching | Browser, CDN, and service memory |
| Client | Debounce, cancel stale requests, local cache, minimum prefix length |
| Resilience | Graceful degradation, last-good snapshot, circuit breaker, canary rollout |
| Safety | Blocklists, filters, abuse protection, privacy rules |

## Continue Learning

- [Caching and CDN](system-design/caching-and-cdn) — caching strategies at each layer.
- [Messaging and Communication](system-design/messaging-and-communication) — Kafka pipelines for query logs.
- [Design Twitter](system-design/twitter-design) — trending topics and search in a social product.
- [Design Netflix](system-design/netflix-design) — search for a streaming catalog.
- [System Design Cheat Sheet](system-design/cheat-sheet) — estimation numbers and quick comparisons.
