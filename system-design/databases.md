---
title: "Databases in System Design — SQL, NoSQL, Indexing, Transactions, Replication, Sharding, and CAP"
description: "A practical guide to databases in system design: SQL vs NoSQL, ACID and isolation levels, CAP and PACELC, indexing (B+ tree and LSM), replication and quorum, sharding, and how to scale a database."
author: ["name": "Rajendra Pancholi", "email": "rpancholi522@gmail.com"]
created: "2026-04-18"
updated: "2026-10-17"
thumbnail: "/images/databases.png"
tags: [databases, sql, nosql, indexing, transactions, sharding, replication, cap-theorem, system-design]
keywords: ["SQL vs NoSQL system design", "Database sharding explained", "CAP theorem and PACELC", "ACID and isolation levels", "Database replication leader follower", "B+ tree vs LSM tree", "Database indexing", "When to use NoSQL"]
---

# Databases in System Design — SQL, NoSQL, Indexing, Transactions, Replication, Sharding, and CAP

The database layer is often the most critical — and most constrained — component in any large-scale system. Choosing the wrong database model, replication strategy, or partitioning scheme can create bottlenecks that are extremely difficult and costly to fix later.

This guide covers the database landscape end to end: relational vs non-relational models, transactions and consistency, CAP, indexing, replication, sharding, and how to scale a database in the right order.

![Database Types](/images/databases.png)

## SQL (Relational) Databases

A relational database stores data in **tables with rows and columns**, with pre-defined relationships between tables enforced by foreign keys. SQL databases offer full ACID transactions.

### Key Characteristics

- **Structured schema** — table structure must be defined before data is inserted; schema changes require migrations
- **Relationships** — data integrity enforced via primary keys, foreign keys, and join operations
- **ACID transactions** — atomicity, consistency, isolation, and durability
- **SQL query language** — powerful, standardized querying with JOINs, aggregations, subqueries

### Materialized Views

A materialized view is a **pre-computed result set** stored as a physical table, derived from a query. It provides dramatically faster reads for complex aggregations at the cost of staleness — the view must be periodically refreshed.

Use cases: dashboard aggregations, reporting queries, pre-joined datasets.

### Advantages

- Simple, mature, well-understood data model
- Strong data consistency and integrity
- Powerful querying with JOINs across related tables
- ACID transactions with rollback support
- Excellent ecosystem: tooling, ORMs, monitoring

### Disadvantages

- **Horizontal scaling is harder** — a single-node relational database scales up easily, but sharding it is complex
- **Schema rigidity** — adding or changing columns requires migrations, which can be risky on very large tables
- **JOIN cost** — complex JOINs across large tables are expensive
- **Object-relational impedance mismatch** — mapping OOP models to tables usually requires an ORM

### Popular SQL Databases

| Database                                           | Notes                                                                                                                  |
| -------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| [PostgreSQL](https://www.postgresql.org)           | Feature-rich open-source SQL DB; JSONB, full-text search, extensions (e.g., pgvector)                                  |
| [MySQL](https://www.mysql.com)                     | Widely deployed; strong read performance; InnoDB engine                                                                |
| [MariaDB](https://mariadb.org)                     | MySQL fork; open-source friendly                                                                                       |
| [Amazon Aurora](https://aws.amazon.com/rds/aurora) | MySQL/PostgreSQL compatible; cloud-native storage replicated across three availability zones with auto-growing storage |

## NoSQL (Non-Relational) Databases

NoSQL databases trade the rigid relational model for flexible schemas and easier horizontal scaling. "NoSQL" is a broad label: the databases under it differ a lot, including in **how consistent they are** (see ACID and BASE below).

### Document Databases

Store data as **semi-structured documents** (JSON, BSON) where each document can have a different structure. No foreign keys — related data is usually embedded within documents.

**Best for:** Content management, user profiles, product catalogs, applications with variable schemas.

**Examples:** [MongoDB](https://www.mongodb.com), [Amazon DocumentDB](https://aws.amazon.com/documentdb), [CouchDB](https://couchdb.apache.org)

### Key-Value Stores

The simplest model — data is stored and retrieved by a **unique key** with an opaque value. Extremely fast for simple lookups but cannot perform complex queries.

**Best for:** Session storage, shopping carts, feature flags, user preferences, caching.

**Examples:** [Redis](https://redis.io), [Amazon DynamoDB](https://aws.amazon.com/dynamodb). (Memcached is a pure cache, not a durable database.)

### Graph Databases

Stores data as **nodes (entities) and edges (relationships)**, optimized for traversing interconnected data. Relationship queries that need many JOINs in SQL are natural and fast here.

**Best for:** Social networks, fraud detection, recommendation engines, knowledge graphs.

**Examples:** [Neo4j](https://neo4j.com), [Amazon Neptune](https://aws.amazon.com/neptune), [ArangoDB](https://www.arangodb.com)

### Time Series Databases

Optimized for **time-stamped sequential data**, with efficient compression and fast range queries over time windows.

**Best for:** IoT sensor data, application metrics, financial tick data, server monitoring.

**Examples:** [InfluxDB](https://www.influxdata.com), [TimescaleDB](https://www.timescale.com), [Apache Druid](https://druid.apache.org) (real-time analytics)

### Wide-Column Stores

Store data in **column families** rather than rows; each row can have a different set of columns. Designed for massive datasets with high write throughput across distributed nodes. Most are built on LSM trees (see Indexes).

**Best for:** Event logging, messaging history, time-series at scale, write-heavy workloads.

**Examples:** [Apache Cassandra](https://cassandra.apache.org), [Google Bigtable](https://cloud.google.com/bigtable), [ScyllaDB](https://www.scylladb.com)

### Also Worth Knowing

| Type                         | What it is                                                                                                                    | Examples                                      |
| ---------------------------- | ----------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------- |
| **NewSQL / distributed SQL** | SQL and ACID transactions **with** horizontal scaling, using consensus between nodes; higher write latency than a single node | Google Spanner, CockroachDB, YugabyteDB, TiDB |
| **Search engines**           | Inverted index for full-text search and filtering; usually a secondary copy of data, not the source of truth                  | Elasticsearch, OpenSearch                     |
| **Vector databases**         | Store embeddings and find the nearest neighbors (similarity search for AI features)                                           | pgvector, Pinecone, Milvus                    |

## SQL vs NoSQL — Decision Guide

| Criteria         | Lean towards SQL                              | Lean towards NoSQL                              |
| ---------------- | --------------------------------------------- | ----------------------------------------------- |
| Data structure   | Structured with defined relationships         | Semi-structured, or accessed mostly by key      |
| Schema           | Fixed, rarely changing                        | Dynamic, frequently evolving                    |
| Consistency      | Strong consistency and integrity are critical | Eventual consistency acceptable                 |
| Scalability      | One powerful node plus replicas is enough     | Massive scale, built-in horizontal partitioning |
| Query complexity | Complex JOINs, ad-hoc queries, reporting      | Known access patterns, simple lookups           |
| Transactions     | Multi-row, multi-table transactions           | Mostly single-key or single-document operations |
| Team expertise   | SQL expertise available                       | NoSQL expertise available                       |

**How to decide in practice:**

- A relational database (PostgreSQL or MySQL) is a sensible default — it handles far more scale than most products ever need.
- Pick NoSQL for a **specific reason**: massive write throughput, flexible documents, key-based access at huge scale, or graph traversal.
- Real systems often use **several databases** (polyglot persistence): SQL for orders and payments, Redis for cache, Elasticsearch for search, a wide-column store for activity logs.
- In an interview, **justify with access patterns and consistency needs**, never with "NoSQL scales better."

## ACID Transactions

ACID is the set of properties that guarantee reliable processing of database transactions.

| Property        | Meaning                                                                                                        |
| --------------- | -------------------------------------------------------------------------------------------------------------- |
| **Atomicity**   | A transaction either fully completes or fully rolls back — no partial success                                  |
| **Consistency** | The database moves from one valid state to another; the application's integrity rules (constraints) are upheld |
| **Isolation**   | Concurrent transactions do not see each other's half-finished work                                             |
| **Durability**  | Once committed, a transaction's effects persist even through crashes (typically via a write-ahead log)         |

> The "C" in ACID (rules stay valid) is **not** the "C" in CAP (all nodes see the same data). They are different ideas.

ACID databases suit financial systems, inventory, bookings, and any domain where correctness cannot be compromised.

### Isolation Levels

Perfect isolation is expensive, so databases offer levels. These are the anomalies each level can allow:

| Level                | Dirty read | Non-repeatable read | Phantom read                                            |
| -------------------- | ---------- | ------------------- | ------------------------------------------------------- |
| **Read Uncommitted** | Possible   | Possible            | Possible                                                |
| **Read Committed**   | No         | Possible            | Possible                                                |
| **Repeatable Read**  | No         | No                  | Possible per the SQL standard (many engines prevent it) |
| **Serializable**     | No         | No                  | No                                                      |

- **Dirty read:** reading another transaction's uncommitted change.
- **Non-repeatable read:** reading the same row twice and getting different values because someone committed in between.
- **Phantom read:** running the same query twice and getting different _sets_ of rows because someone inserted or deleted.

Defaults differ: PostgreSQL defaults to Read Committed, MySQL InnoDB to Repeatable Read. Higher isolation means more safety but more contention or aborted transactions.

### Concurrency Control

- **MVCC** (multi-version concurrency control): the database keeps multiple versions of a row so readers never block writers. Used by PostgreSQL and InnoDB.
- **Pessimistic locking:** lock the row first (`SELECT ... FOR UPDATE`). Safe for high contention, but blocks others.
- **Optimistic locking:** read a `version` column, and update with `WHERE version = N`; if zero rows changed, someone else won, so retry. Good for low contention.

These prevent problems like **lost updates** (two users read a balance and both write back) — a very common interview scenario.

## BASE and Consistency in NoSQL

BASE is a description of a design philosophy common in large distributed databases: favor availability and scalability over immediate consistency.

| Property                  | Meaning                                                                              |
| ------------------------- | ------------------------------------------------------------------------------------ |
| **Basically Available**   | The system keeps responding even during partial failures                             |
| **Soft State**            | State may change over time as replicas converge                                      |
| **Eventually Consistent** | If no new writes happen, all replicas eventually agree; reads may be stale meanwhile |

**Important correction to the common claim "SQL = ACID, NoSQL = BASE":**

- BASE is a loose idea, not a formal guarantee, and many databases don't fit neatly in either camp.
- **MongoDB** supports multi-document ACID transactions. **DynamoDB** supports transactions and optional strongly consistent reads. **Cassandra** lets you choose consistency per query.
- Conversely, a relational database with asynchronous read replicas serves **eventually consistent** reads from those replicas.

So think in terms of **what consistency you get for each operation**, not a label on the database. See also [Caching and CDN](system-design/caching-and-cdn) for how caching adds eventual consistency.

## CAP Theorem and PACELC

The CAP theorem says that when a **network partition** happens in a distributed system, it must choose between **consistency** and **availability**.

![CAP Theorem](/images/cap-theorem.png)

| Property                    | Meaning                                                                                        |
| --------------------------- | ---------------------------------------------------------------------------------------------- |
| **Consistency (C)**         | Every read sees the most recent write (linearizability), as if there were one copy of the data |
| **Availability (A)**        | Every request to a non-failing node gets a response (not necessarily the latest data)          |
| **Partition Tolerance (P)** | The system keeps working despite messages being lost between nodes                             |

**Common misunderstanding:** "pick any two of three." In reality:

- Network partitions **will** happen in any distributed system, so **P is not optional**.
- The real choice is **only during a partition**: refuse some requests to stay consistent (**CP**), or answer with possibly stale data to stay available (**AP**).
- When there is **no** partition, a system can be both consistent and available.
- A "CA" distributed system does not exist; a single-node database is simply not a distributed system, so CAP doesn't apply to it.

### Typical Behavior

| Choice | Behavior during a partition                                                             | Examples (default configurations)                          |
| ------ | --------------------------------------------------------------------------------------- | ---------------------------------------------------------- |
| **CP** | The minority side stops serving requests (or rejects writes) to avoid inconsistent data | ZooKeeper, etcd, HBase, MongoDB with majority writes       |
| **AP** | All sides keep answering and reconcile later                                            | Cassandra, DynamoDB (eventually consistent reads), CouchDB |

Many databases are **configurable** (Cassandra with `QUORUM`, DynamoDB with strongly consistent reads), so classifying them in a single box is a simplification.

### PACELC: The More Useful Version

CAP only talks about partitions, which are rare. **PACELC** adds the everyday trade-off:

> If there is a **P**artition, choose **A**vailability or **C**onsistency; **E**lse (normal operation), choose **L**atency or **C**onsistency.

Even without failures, keeping replicas strongly consistent costs latency (you must wait for them). Typical placements: Cassandra and DynamoDB are **PA/EL** (favor availability and low latency); Spanner-style systems are **PC/EC** (favor consistency). Use these as rough guides, not strict labels.

## Database Indexes

Indexes are data structures that let the database find rows **without scanning the whole table**, like a book's index.

**Trade-off:** indexes speed up reads, but slow down writes (every INSERT, UPDATE, and DELETE must update each index) and use extra storage.

### B+ Tree (the default)

- A balanced, sorted tree with high fan-out, so even billions of rows need only a few levels (a few disk reads per lookup).
- Data (or pointers to it) lives in the **leaf nodes**, which are linked in order — great for **range queries** (`BETWEEN`, `>`) and `ORDER BY`.
- Used by most relational databases (PostgreSQL, MySQL).

### Hash Index

Maps a key to a location with a hash function. Very fast for **equality** lookups, but cannot do ranges or ordering.

### LSM Tree (write-optimized)

Used by Cassandra, RocksDB, HBase, and many NoSQL stores:

1. Writes go to an in-memory **memtable** (plus a write-ahead log for durability).
2. When full, it is flushed to disk as an immutable sorted file (**SSTable**).
3. Background **compaction** merges files and removes old versions.
4. Reads check the memtable and then SSTables; **Bloom filters** skip files that cannot contain the key.

|          | B+ Tree                                   | LSM Tree                    |
| -------- | ----------------------------------------- | --------------------------- |
| Writes   | Slower (update tree in place, random I/O) | Faster (sequential appends) |
| Reads    | Fast, predictable                         | May check several files     |
| Best for | Read-heavy, balanced workloads            | Write-heavy workloads       |

### Clustered vs Secondary Indexes

- **Clustered index:** the table rows are stored in the index order (in MySQL InnoDB, the primary key). There can be only one.
- **Secondary index:** a separate structure that points back to the row (in InnoDB, to the primary key). A lookup may need a second step to fetch the row.

### Composite and Covering Indexes

- A **composite index** on `(a, b, c)` helps queries filtering on `a`, or `a, b`, or `a, b, c` — the **leftmost-prefix rule**. It does **not** help a query filtering only on `b`. Put equality columns first, range columns later.
- A **covering index** contains every column a query needs, so the database answers from the index alone without touching the table (an index-only scan).

### When an Index Is Not Used

- Applying a function to the column (`WHERE LOWER(email) = ...`) unless there is a matching functional index
- Leading wildcard searches (`LIKE '%abc'`)
- Low-selectivity columns (a boolean with two values), where a full scan may be cheaper
- Mismatched types (comparing a string column to a number)

Use `EXPLAIN` to see whether a query uses an index, and add indexes for the queries you actually run, not for every column.

## Database Replication

Replication copies data across multiple database servers to improve reliability, read performance, and fault tolerance. (Older material says "master-slave"; the standard terms are **leader-follower** or **primary-replica**.)

### Leader-Follower (Primary-Replica) Replication

One **leader** accepts all writes. Changes are streamed to one or more **followers**, which serve reads.

![Leader-Follower Replication](/images/master-slave-replication.png)

**Advantages:**

- Read scalability — add followers to scale reads
- Backups and analytics can run on a follower without hurting the leader
- A follower can be promoted if the leader fails

**Disadvantages:**

- A single write endpoint is a bottleneck and a single point of failure until failover completes
- **Replication lag** — followers may serve stale data
- Failover needs care (see below)

### Replication Lag Problems

With asynchronous replicas, a user may not see their own update:

- **Read-your-writes violation:** a user posts a comment, the next read goes to a lagging replica, and the comment is missing. Fix: read from the leader for data the user just changed (or for a short window after their write), or track the replica's replication position.
- **Non-monotonic reads:** refreshing shows newer data and then older data from a different replica. Fix: route a given user to the same replica.

### Failover

When the leader fails: detect it (timeouts or heartbeats), choose the most up-to-date follower, promote it, and point clients at it. Risks: **lost writes** if replication was asynchronous, **split-brain** if the old leader comes back and still thinks it's the leader, and false detection from a too-aggressive timeout. See [High Availability](system-design/high-availability).

### Multi-Leader Replication

Several nodes accept writes and replicate to each other. Used for **multi-region writes** or offline-capable clients.

![Multi-Leader Replication](/images/master-master-replication.png)

**Advantages:** writes are local to each region; survives a leader or region failure without a promotion step.

**Disadvantages:** **write conflicts** (two leaders change the same row at once) need resolution — last-write-wins (can silently lose data), merge rules, CRDTs, or application logic. Also more complex to reason about.

### Leaderless Replication and Quorums

Systems like Cassandra and Dynamo-style stores have **no leader**. A write goes to several replicas and a read asks several replicas. With **N** replicas, writing to **W** and reading from **R**:

```text
if R + W > N, the read set overlaps the write set,
so a read sees at least one up-to-date copy
```

Example: N = 3, W = 2, R = 2 (quorum). Lower W or R gives lower latency but weaker consistency. This is a strong guideline, not a perfect guarantee (concurrent writes and "sloppy quorums" can still cause stale reads).

### Synchronous vs Asynchronous Replication

| Type                 | Behavior                                                      | Trade-off                                                                     |
| -------------------- | ------------------------------------------------------------- | ----------------------------------------------------------------------------- |
| **Synchronous**      | Write acknowledged only after replicas confirm                | No data loss, but higher latency, and a slow or dead replica can block writes |
| **Asynchronous**     | Acknowledged after the leader writes; replicas catch up later | Lower latency, but recent writes can be lost if the leader fails              |
| **Semi-synchronous** | Wait for **one** replica, the rest are async                  | Common compromise: at least two copies exist, with modest latency             |

## Database Sharding

Sharding (horizontal partitioning) splits data across multiple database servers, each holding a **subset (shard)** of the data with the same schema but different rows. (Table _partitioning_ splits a table within one server; _sharding_ spreads it across servers.)

![Database Sharding](/images/sharding.png)

### Partitioning Criteria

| Strategy             | Description                                                | Pros                            | Cons                                                  |
| -------------------- | ---------------------------------------------------------- | ------------------------------- | ----------------------------------------------------- |
| **Hash-Based**       | Hash a key; use result modulo N to select shard            | Even distribution               | Changing N remaps almost every key                    |
| **Range-Based**      | Split by value ranges (e.g., A–M → Shard 1, N–Z → Shard 2) | Easy range queries              | Hot spots if data is skewed (e.g., newest timestamps) |
| **List / Geo-Based** | Assign specific values to shards (e.g., by country)        | Logical, good for data locality | Manual management, uneven sizes                       |
| **Directory-Based**  | A lookup table maps each key to a shard                    | Maximum flexibility             | The lookup service is another dependency              |
| **Composite**        | Combine strategies                                         | Flexible                        | Complex                                               |

### Choosing a Shard Key

The shard key is the most important sharding decision. A good key has:

- **High cardinality** (many distinct values)
- **Even distribution** of data and traffic (no hot shard)
- **Alignment with the main query** so most queries hit one shard

Bad keys cause hot spots: sharding a social app by `user_id` is usually fine, but a celebrity's data can still overload one shard (a **hot key** problem).

### Consistent Hashing

Plain `hash(key) % N` reshuffles nearly all keys when N changes. **Consistent hashing** maps servers and keys onto a ring; each key belongs to the next server clockwise. When a server is added or removed, only about `K/N` keys move (K = total keys, N = nodes). **Virtual nodes** (each server appears at many points on the ring) keep the distribution even and spread a failed node's load across many nodes.

### Resharding

Adding capacity later is painful. A common trick is to create **many logical shards up front** (say 1,024) and map them to a few physical servers; to grow, move whole logical shards to new servers rather than re-hashing every key.

### Unique IDs Across Shards

Auto-increment IDs collide across shards. Use UUIDs, or time-ordered IDs (Snowflake-style: timestamp + machine ID + sequence) that are unique without coordination.

### Advantages of Sharding

- Massive horizontal scalability — add shards to increase capacity
- Each shard handles a smaller dataset and fewer queries
- Fault isolation — a failure in one shard affects only its data

### Disadvantages of Sharding

- Cross-shard queries and JOINs are expensive or must be merged in the application
- Transactions across shards need distributed protocols (two-phase commit or sagas) and are slow and complex
- Resharding and rebalancing are hard
- Secondary indexes become harder (a "global" index must be maintained separately)
- Higher operational complexity — **shard last, not first**

## Normalization vs Denormalization

**Normalization** organizes data to minimize redundancy by splitting data into multiple related tables (1NF, 2NF, 3NF, BCNF).

**Denormalization** intentionally duplicates data to optimize reads and avoid expensive JOINs.

| Approach     | Read Performance        | Write Performance               | Storage | Consistency               |
| ------------ | ----------------------- | ------------------------------- | ------- | ------------------------- |
| Normalized   | Slower (requires JOINs) | Faster (update in one place)    | Less    | Better                    |
| Denormalized | Faster (fewer JOINs)    | Slower (update multiple copies) | More    | Harder to keep consistent |

In high-read systems (feeds, dashboards), denormalization is a deliberate optimization. In NoSQL stores that lack JOINs, you **model the data around your access patterns** — design the document or table for the queries you will run, duplicating data where needed.

## Scaling a Database: The Usual Order

Before reaching for sharding, work through the cheaper steps:

1. **Fix queries and add the right indexes** (use `EXPLAIN`). Often the biggest win.
2. **Use connection pooling** so the database isn't overwhelmed by thousands of connections.
3. **Add a cache** in front of hot reads — see [Caching and CDN](system-design/caching-and-cdn).
4. **Add read replicas** for read-heavy load, handling replication lag.
5. **Scale vertically** — a larger machine is often cheaper than the engineering cost of distributing.
6. **Archive or partition old data** to keep tables and indexes small.
7. **Shard** only when a single leader can no longer handle the writes or the data size.

See also [Scaling](system-design/scaling).

## Interview Questions

**Q: SQL or NoSQL for this system?**
Decide from access patterns and consistency needs. Use SQL when you need relationships, ad-hoc queries, and multi-row transactions. Use NoSQL when you need huge write throughput, flexible schemas, or key-based access at scale. Many systems use both.

**Q: Is it true that NoSQL databases don't support ACID?**
No. Several (MongoDB, DynamoDB) support transactions, and consistency is often configurable per operation. The accurate view is to ask what guarantee each operation gets.

**Q: Explain CAP. Why is "CA" not a real option?**
During a network partition, a distributed system must choose consistency or availability. Partitions can't be avoided, so P is mandatory; the real choice is C vs A only while a partition exists. PACELC adds that, even without partitions, you trade latency against consistency.

**Q: What are the isolation levels and what anomalies do they prevent?**
Read Uncommitted allows dirty reads. Read Committed prevents dirty reads. Repeatable Read also prevents non-repeatable reads. Serializable prevents phantoms too. Higher levels cost more contention.

**Q: How does an index speed up queries and what does it cost?**
A B+ tree lets the database find rows in a few reads instead of scanning the table, and supports ranges and ordering. It costs extra storage and slows writes because every write updates the index.

**Q: Why use an LSM tree instead of a B+ tree?**
LSM trees turn writes into sequential appends, which is much faster for write-heavy workloads. The cost is slower or more variable reads (several files may be checked) and background compaction.

**Q: How do you handle replication lag?**
For read-your-own-writes, read from the leader for recently changed data or route that user appropriately; for consistent browsing, keep a user on the same replica; and use semi-synchronous replication where data loss is unacceptable.

**Q: How would you choose a shard key, and what can go wrong?**
Pick a high-cardinality key that spreads data and traffic evenly and matches the main query. Bad keys cause hot shards; changing the shard count later is painful, so use many logical shards or consistent hashing.

**Q: What is R + W > N?**
In a leaderless system with N replicas, writing to W and reading from R replicas with R + W > N guarantees the read overlaps at least one replica holding the latest write.

## Summary

- Start with a relational database as the default; choose NoSQL for a specific reason, and don't rely on the "SQL = ACID, NoSQL = BASE" slogan.
- ACID plus isolation levels, MVCC, and optimistic or pessimistic locking decide how safe concurrent updates are.
- CAP matters only during partitions (a choice between C and A); PACELC adds the everyday latency-vs-consistency trade-off.
- B+ trees suit balanced and read-heavy workloads; LSM trees suit write-heavy ones. Know composite, covering, and leftmost-prefix indexes.
- Leader-follower replication scales reads but introduces lag and failover risk; quorums (R + W > N) tune consistency in leaderless systems.
- Shard last, with a carefully chosen key, and expect cross-shard queries and transactions to be hard.

## Continue Learning

- [Caching and CDN](system-design/caching-and-cdn) — reducing database load before scaling it.
- [Scaling](system-design/scaling) — where the database fits in growing a system.
- [High Availability](system-design/high-availability) — failover, split-brain, and redundancy.
- [Messaging and Communication](system-design/messaging-and-communication) — decoupling writes with queues.
- [Design a URL Shortener](system-design/url-shortener-design) — database choice applied to a real design.
