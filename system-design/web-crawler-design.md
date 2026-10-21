---
title: "System Design Case Study — Web Crawler (Search Engine Crawling at Scale)"
description: "A complete system design for a distributed web crawler: URL frontier with priority and politeness, fetching and DNS caching, robots.txt, URL normalization and deduplication with Bloom filters, content deduplication, spider traps, distributed partitioning, recrawling, and fault tolerance."
author: ["name": "Rajendra Pancholi", "email": "rpancholi522@gmail.com"]
created: "2026-10-17"
updated: "2026-10-17"
thumbnail: "/images/networking-fundamentals.png"
tags: [system-design, web-crawler, bloom-filter, url-frontier, search-engine, case-study]
keywords: ["Design a web crawler", "Web crawler system design", "URL frontier politeness", "Bloom filter URL deduplication", "Distributed crawler architecture", "robots.txt crawling", "System design interview web crawler"]
---

# System Design Case Study — Web Crawler (Search Engine Crawling at Scale)

A web crawler (or spider) **discovers and downloads web pages automatically**: it starts from a list of known URLs, fetches each page, extracts the links, and follows them to find more pages. Search engines use crawlers to collect pages to index; others use them for archiving, price monitoring, data mining, and security scanning.

Crawling sounds simple (a loop of "fetch page, find links, repeat"), but at scale it raises interesting design problems: **billions of URLs, politeness toward website owners, duplicates, infinite loops, failures, and fresh content**.

![Web Crawler](/images/networking-fundamentals.png)

---

## Requirements

### Functional Requirements
- Start from **seed URLs** and discover new pages by following links
- **Download** pages (primarily HTML) and store them for later processing (such as indexing)
- **Extract links** from each page
- Respect **robots.txt** and be **polite** to websites
- Avoid **duplicate** crawling (same URL, same content)
- **Recrawl** pages periodically to keep content fresh
- Be **extensible**: support other content types (images, PDFs) later

### Non-Functional Requirements
- **Scalable:** crawl billions of pages
- **Robust:** survive bad HTML, huge pages, slow servers, crashes, and crawler traps
- **Polite:** never overload a website
- **Efficient:** use bandwidth and machines well
- **Extensible and maintainable:** pluggable parsers and processors

---

## Capacity Estimation

Assumptions: crawl **1 billion pages per month**, with an average page size of **100 KB** of HTML.

```
Rate: 1B pages / (30 × 86,400 s) ≈ 385 pages/second  (peak ~2x → ~800/second)

Bandwidth: 385 pages/s × 100 KB ≈ 38.5 MB/s ≈ 310 Mbps

Storage per month: 1B × 100 KB = 100 TB raw
With compression (~3x): ~33 TB/month → about 2 PB over 5 years

URL "seen" set: suppose 10 billion distinct URLs discovered
Bloom filter at 1% false-positive rate needs ~9.6 bits per URL:
10B × 9.6 bits ≈ 96 Gbit ≈ 12 GB   (fits in memory across a few nodes)
```

| Metric | Estimate |
|---|---|
| Crawl rate | ~385 pages/s (800 peak) |
| Download bandwidth | ~310 Mbps |
| Raw storage | ~100 TB/month (~33 TB compressed) |
| URL seen-set (Bloom filter) | ~12 GB for 10 billion URLs |

Two takeaways: **throughput is moderate** (hundreds of pages per second), but each fetch is **slow** (network latency of hundreds of milliseconds), so you need **many concurrent fetches** (tens of thousands in flight), and the **URL and duplicate bookkeeping** is the real scale challenge.

---

## High-Level Architecture

```
Seed URLs
    │
    ▼
┌──────────────┐   next URL    ┌──────────────┐   HTML     ┌──────────────┐
│ URL Frontier │ ────────────► │   Fetchers   │ ─────────► │ Content Store│ (object storage)
│ (priority +  │               │ (DNS cache,  │            └──────┬───────┘
│  politeness) │               │  robots.txt) │                   ▼
└──────▲───────┘               └──────────────┘            ┌──────────────┐
       │                                                    │ Parser /     │
       │ new URLs                                           │ Link Extractor│
       │                                                    └──────┬───────┘
┌──────┴───────┐   unseen URLs  ┌──────────────┐  all links       │
│ URL Dedup    │ ◄───────────── │  URL Filter  │ ◄────────────────┘
│ (Bloom + DB) │                │ + Normalizer │
└──────────────┘                └──────────────┘
                 Content Dedup (hash / SimHash) ──► Indexer / downstream
```

### Components

| Component | Responsibility |
|---|---|
| **URL Frontier** | The queue of URLs to crawl, with prioritization and per-host politeness |
| **Fetchers (downloaders)** | Download pages using many concurrent connections |
| **DNS resolver cache** | Fast, cached DNS lookups (DNS can become a bottleneck) |
| **robots.txt cache** | Rules per host, so each site is checked once in a while |
| **Parser / link extractor** | Extracts text and links, handles bad HTML |
| **URL normalizer and filter** | Canonicalizes URLs and removes unwanted ones |
| **URL deduplicator** | Remembers which URLs were already seen |
| **Content deduplicator** | Detects identical or near-identical pages |
| **Content store** | Saves the raw or parsed content, handed to the indexer |
| **Scheduler / recrawl manager** | Decides when to revisit pages |

---

## The Crawl Loop

1. Take a URL from the **frontier**.
2. Check **robots.txt** for the host (cached) to see if crawling is allowed.
3. **Resolve DNS** (cached) and **fetch** the page, with timeouts and size limits.
4. **Store** the content, and **parse** it to extract links.
5. **Normalize** and **filter** the extracted URLs.
6. Check the **seen-set**; add genuinely new URLs to the frontier.
7. Check for **duplicate content** before sending the page to the indexer.
8. Repeat.

Crawling can be a **graph traversal**: the web is a graph, and the frontier decides whether the traversal is breadth-first, depth-first, or priority-driven. Pure DFS risks getting lost deep in one site; **BFS with priorities** is the usual choice.

---

## URL Frontier: Priority and Politeness

The frontier is the brain of the crawler. It must answer: *what should we fetch next*, while ensuring we don't hammer any single website.

### Prioritization

Not all URLs are equal. Rank them by:

- **Importance** (for example link-based authority scores like PageRank, or the importance of the domain)
- **Freshness need:** pages that change often (news homepages) deserve more frequent visits
- **Depth:** shallow pages first
- **Quality signals** and past crawl success

### Politeness

A crawler that opens hundreds of connections to one small website can take it down. Rules:

- **One request at a time per host**, with a **delay between requests** to the same host (a few seconds is common; respect `Crawl-delay` where specified).
- Limit total requests per domain per time window.
- Identify yourself with an honest **User-Agent** and contact info.

### A Two-Level Queue Design (Mercator-Style)

```
                    ┌────────────── Front queues (by priority) ──────────────┐
New URLs ─► Prioritizer ─► [High] [Medium] [Low]
                                 │ (select by weighted choice)
                                 ▼
                      Router (maps each URL by host)
                                 ▼
       ┌───────────── Back queues (one queue per host) ──────────────┐
       │ host A queue │ host B queue │ host C queue │ ...              │
       └──────────────────────────────────────────────────────────────┘
                                 ▼
        A min-heap holds "next allowed fetch time" per host;
        a worker takes the host whose time has arrived, fetches ONE URL, then reschedules the host.
```

- **Front queues** handle **priority**.
- **Back queues** hold URLs **per host**, and a timer structure ensures a host is contacted only when its politeness delay has passed. Each host's queue is served by **one worker at a time**.
- The frontier is **large** (billions of URLs), so it can't live only in memory: keep **hot parts in memory and the bulk on disk** (or in a distributed queue/store), and persist state so a restart doesn't lose the crawl.

---

## Fetching

- Use **asynchronous I/O** or large thread pools so tens of thousands of fetches are in flight at once (each spends most of its time waiting on the network).
- **Timeouts** (connect and read), **maximum page size**, and **maximum redirects** (to avoid redirect loops).
- Handle **HTTP details:** status codes (200, 301/302 redirects, 404, 429, 5xx), **compression**, character encodings, and content types (skip content you don't want).
- **Conditional requests** for recrawls: send `If-None-Match` (ETag) or `If-Modified-Since`, and the server may answer `304 Not Modified` with no body, saving bandwidth.
- Back off on `429 Too Many Requests` or repeated errors from a host.

### DNS Is a Hidden Bottleneck

Millions of distinct hosts mean millions of DNS lookups, and many resolvers are slow or rate-limited. Run a **local caching resolver** (or a dedicated DNS service), respect TTLs, and make lookups asynchronous. See [DNS](system-design/dns).

### robots.txt

`robots.txt` at the root of a site says which paths crawlers may or may not fetch.

- **Fetch and cache** it per host (and refresh periodically, for example daily).
- Check every URL against it before fetching.
- If `robots.txt` is missing, crawling is generally allowed; if it can't be fetched due to server errors, be cautious and retry later.
- It is a convention and a matter of **ethics and legal risk**; a good crawler always honors it.

---

## Parsing and Extracting Links

- Parse HTML **robustly**: real-world HTML is often broken, so use a tolerant parser.
- Extract links from `<a href>` (and optionally images, scripts, and sitemaps).
- Resolve **relative URLs** against the page's base URL.
- **JavaScript-rendered pages:** content that appears only after scripts run is invisible to a plain HTML fetch. A fraction of pages may require a **headless browser**, which is far more expensive, so use it selectively.
- Use **sitemaps** (`sitemap.xml`) when available, as they list URLs and their update times.

---

## URL Normalization and Filtering

The same page can appear under many URLs. Normalize before checking for duplicates:

| Rule | Example |
|---|---|
| Lowercase scheme and host | `HTTP://Example.COM/` → `http://example.com/` |
| Remove default ports | `http://example.com:80/` → `http://example.com/` |
| Remove fragments | `page.html#section2` → `page.html` |
| Resolve `.` and `..` | `/a/b/../c` → `/a/c` |
| Sort query parameters | `?b=2&a=1` → `?a=1&b=2` |
| Drop tracking parameters | `utm_source`, session IDs |
| Normalize trailing slash and index files | `/dir` and `/dir/` and `/dir/index.html` (when known equivalent) |

**Filter out** URLs that are unwanted: unsupported schemes (`mailto:`, `javascript:`), blocked domains, disallowed file types, overly long URLs, and anything robots.txt forbids.

---

## Avoiding Duplicates

### URL-Level Deduplication

Before adding a URL to the frontier, check whether it has been **seen**.

- An exact structure (a hash set or database of billions of URL hashes) works but is large.
- A **Bloom filter** gives a compact first check: "definitely not seen" (add it) or "possibly seen." It has **false positives** (a small fraction of genuinely new URLs are wrongly skipped) but **never false negatives**, so you never re-crawl a URL by mistake. At 1% false positives, 10 billion URLs need about 12 GB. See [Distributed Systems Essentials](system-design/distributed-systems-essentials).
- When exactness matters, use the Bloom filter as a fast filter in front of a persistent key-value store holding URL hashes.

### Content-Level Deduplication

Different URLs often serve the same or nearly the same content (mirrors, print versions, tracking variants).

- **Exact duplicates:** compute a hash (for example SHA-256) of the content and check it against a store of known hashes.
- **Near duplicates:** use **SimHash** or **MinHash** fingerprints, which map similar documents to similar fingerprints, so you can find pages that differ only in small details such as ads or timestamps.
- Save storage and indexing cost by keeping one canonical copy.

---

## Spider Traps and Bad Content

A **spider trap** is a structure that generates an effectively infinite set of URLs, wasting crawl budget: calendars with endless "next month" links, session IDs in URLs, or auto-generated paths (`/a/a/a/a/...`).

Defenses:

- Maximum **URL length** and **path depth**.
- A **per-domain cap** on pages crawled (a crawl budget).
- Detect **repeating path patterns** and abnormally large numbers of URLs with the same content hash.
- Drop **session identifiers** during normalization.
- Prioritize by quality, so low-value pages sink to the bottom of the frontier.
- Handle **malicious or huge pages** by enforcing size and time limits, and avoid executing untrusted scripts outside a sandbox.

---

## Distributing the Crawler

A single machine can't crawl the web, so distribute the work.

### Partition by Host

Assign each **host (domain)** to one crawler node using **consistent hashing** on the hostname.

- **Politeness is easy:** all requests to a host come from one node, which controls the rate.
- **Local caches work well:** robots.txt, DNS, and per-host queues live on the node that needs them.
- **Deduplication** for a host's URLs can be local to that node.

### Exchanging URLs

When a node extracts a link belonging to a host owned by **another node**, it sends that URL to the owner through a **message queue** (see [Messaging and Communication](system-design/messaging-and-communication)). Each node consumes its own input topic, adds URLs to its frontier, and crawls.

### Handling Node Changes

When nodes join or leave, consistent hashing moves only a small share of hosts. The persistent frontier state lets the new owner **resume** the host's queue.

### Geographic Distribution

Place crawlers near the sites they crawl (different regions) to reduce latency, subject to partitioning rules.

---

## Storage

| Data | Store |
|---|---|
| **Raw/parsed page content** | Object storage or a distributed file system (compressed, batched into large files) |
| **URL metadata** (last crawl time, status, content hash, change history, outlinks) | Wide-column or key-value store (Cassandra, Bigtable-style), keyed by URL hash |
| **Frontier state** | Distributed queue/store with persistence |
| **robots.txt, DNS** | In-memory caches with TTLs |
| **Seen-URL set** | Bloom filter plus a persistent store |

See [Databases](system-design/databases) for choosing between these stores.

---

## Freshness: Recrawling

Pages change at different rates, so uniform recrawling wastes effort.

- **Adaptive revisit frequency:** track how often a page changed on past visits, and visit frequently changing pages (news) often and static pages rarely.
- **Importance-weighted:** important pages are revisited more.
- **Conditional GET** and sitemap hints reduce cost.
- Maintain a **recrawl scheduler** that inserts URLs back into the frontier when they are due.
- Handle **deleted pages** (404/410): retry a few times, then remove.

---

## Fault Tolerance and Robustness

- **Persist the frontier and progress** (checkpoints), so a crash resumes rather than restarts.
- Make fetch workers **stateless** and replaceable; the frontier retries URLs whose fetch did not complete.
- **Retry** transient errors with exponential backoff and jitter; give up on persistent failures after a limit and record them.
- **Idempotent processing:** the pipeline can see the same URL or content twice (queues deliver at least once), so processing must tolerate duplicates.
- **Isolate bad hosts:** a slow or failing host must not block workers (timeouts, per-host concurrency limits, a circuit breaker per host).
- **Monitor** crawl rate, error rates by type, frontier size, per-domain stats, and duplicate rates, and alert on anomalies. See [Observability and Deployment](system-design/observability-and-deployment).

---

## Legal and Ethical Considerations

- Honor **robots.txt** and meta robots directives; identify yourself with a clear **User-Agent** and provide a way for site owners to contact you or opt out.
- Respect **copyright and terms of service**, and don't collect private or sensitive data.
- Limit crawl rates to avoid harming sites.

---

## Bottleneck Resolutions

| Bottleneck | Solution |
|---|---|
| Slow network fetches | Asynchronous I/O and tens of thousands of concurrent connections |
| DNS lookups | Local caching resolver, async lookups |
| Overloading websites | Per-host queues, delays, one worker per host at a time |
| Billions of seen URLs | Bloom filter plus persistent key-value store |
| Duplicate pages | Content hashes and SimHash/MinHash |
| Infinite URL spaces (traps) | Depth and length limits, per-domain budgets, pattern detection |
| Huge frontier | Priority queues spread across memory and disk, persisted and partitioned by host |
| One machine's limits | Distribute by host hash with consistent hashing and a queue for cross-node URLs |
| Stale content | Adaptive recrawl scheduling with conditional requests |
| Crashes | Persistent frontier, retries, stateless workers, idempotent processing |

---

## Interview Questions

**Q: How do you ensure a crawler is polite?**
Group URLs by host, allow one in-flight request per host with a delay between requests, honor robots.txt and crawl-delay, and cap per-domain request rates.

**Q: How do you avoid crawling the same URL twice?**
Normalize URLs, then check a seen-set. A Bloom filter gives a compact first check (no false negatives), backed by a persistent store when exactness is needed.

**Q: Why is the Bloom filter acceptable here despite false positives?**
A false positive only causes a rare new URL to be skipped. The web is so large that missing a tiny fraction costs little, while the memory savings are huge. There are no false negatives, so URLs are never re-crawled by mistake.

**Q: How do you detect duplicate or near-duplicate content?**
Hash content for exact duplicates, and use SimHash or MinHash fingerprints to find near duplicates.

**Q: What is a spider trap and how do you handle it?**
A generator of effectively infinite URLs (calendars, session IDs). Limit URL length, depth, and per-domain page counts, strip session IDs, and detect repeated patterns.

**Q: How would you distribute the crawler?**
Partition hosts across nodes with consistent hashing, so each node controls politeness and caches for its hosts, and pass cross-partition URLs through a message queue.

**Q: BFS or DFS?**
Prioritized BFS: DFS can sink into one site or a trap, while BFS with priority spreads across sites and reaches important pages first.

**Q: How do you keep content fresh?**
Learn each page's change frequency, revisit changing pages more often, use conditional GET and sitemaps, and schedule recrawls through the frontier.

**Q: How do you handle JavaScript-heavy pages?**
Render a selected subset with a headless browser (expensive), and prefer APIs or sitemaps where available.

---

## Summary

| Component | Choice |
|---|---|
| Frontier | Two-level queues: priority front queues and per-host back queues with next-fetch-time scheduling |
| Fetching | Async I/O, timeouts, size limits, cached DNS and robots.txt, conditional GET |
| URL handling | Normalization, filtering, Bloom filter plus persistent store |
| Duplicates | URL seen-set; content hash and SimHash |
| Traps | Depth, length, per-domain caps, pattern detection |
| Scaling | Partition by host with consistent hashing; queues between nodes |
| Storage | Object storage for pages, wide-column store for URL metadata |
| Freshness | Adaptive recrawl, conditional requests, sitemaps |
| Robustness | Persistent frontier, retries with backoff, stateless workers, idempotent processing |
| Ethics | robots.txt, polite rates, honest User-Agent |

## Continue Learning

- [DNS](system-design/dns) — resolution and caching that crawlers depend on.
- [Distributed Systems Essentials](system-design/distributed-systems-essentials) — Bloom filters and consistent hashing ideas.
- [Messaging and Communication](system-design/messaging-and-communication) — queues between crawler nodes.
- [Design Search Autocomplete](system-design/typeahead-design) — another search-related system.
- [Rate Limiting](system-design/rate-limiting) — per-host request limiting concepts.
