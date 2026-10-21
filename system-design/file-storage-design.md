---
title: "System Design Case Study — File Storage and Sync (Dropbox / Google Drive)"
description: "A complete system design for a cloud file storage and sync service: chunking and deduplication, resumable uploads with pre-signed URLs, the metadata service, device sync with change feeds, conflict handling, sharing, versioning, durability, and garbage collection."
author: ["name": "Rajendra Pancholi", "email": "rpancholi522@gmail.com"]
created: "2026-10-17"
updated: "2026-10-17"
thumbnail: "/images/databases.png"
tags: [system-design, file-storage, dropbox, google-drive, chunking, deduplication, sync, case-study]
keywords: ["Design Dropbox", "Design Google Drive", "File sync system design", "Chunking and deduplication", "Resumable upload pre-signed URL", "Object storage design", "System design interview file storage"]
---

# System Design Case Study — File Storage and Sync (Dropbox / Google Drive)

A file storage service lets users **upload files, access them from any device, keep devices in sync, and share them with others**. It looks simple, but it combines several hard problems: moving **very large files** reliably over flaky networks, **never losing data**, **saving bandwidth and storage** with chunking and deduplication, and **keeping many devices consistent**, including when users edit offline.

![File Storage](/images/databases.png)

---

## Requirements

### Functional Requirements
- **Upload and download** files of any type, including very large ones (many GB)
- **Automatic sync:** a change on one device appears on all the user's other devices
- **Sharing:** share files and folders with users or via links, with view or edit permissions
- **Version history** and recovery of deleted or overwritten files
- **Offline support:** edits made offline sync when the device reconnects
- Notifications of changes to shared files

### Non-Functional Requirements
- **Durability:** files must never be lost (extreme durability, often described as 11 nines)
- **Reliability** of uploads over unstable networks (resumable)
- **Efficiency:** don't re-upload unchanged data; save storage and bandwidth
- **Consistency:** the file tree (metadata) must be strongly consistent; file content syncs eventually
- **Availability** and **low latency** for metadata operations
- **Security:** encryption in transit and at rest, safe sharing

---

## Capacity Estimation

Assumptions: **50 million users**, **10 million daily active users**, **1 GB average stored per user**, **2 file changes per active user per day**, **1 MB of new data per change** (after chunking and dedup).

```
Logical storage: 50M users × 1 GB = 50 PB
(deduplication and compression might reduce physical storage by tens of percent)

Uploads: 10M × 2 = 20M changes/day
Data/day: 20M × 1 MB = 20 TB/day
Ingress: 20 TB / 86,400 s ≈ 230 MB/s ≈ 1.9 Gbps average (peak 2–3x)

Metadata operations (assume ~100 per active user/day: listings, sync polls, commits):
10M × 100 = 1B/day ≈ 11,600/second

With a 4 MB chunk size, 50 PB / 4 MB ≈ 12.5 billion chunks (before dedup)
→ the chunk index is large and must be sharded
```

| Metric | Estimate |
|---|---|
| Logical storage | ~50 PB |
| New data | ~20 TB/day (~1.9 Gbps average) |
| Metadata ops | ~11,600/s |
| Chunks | ~12.5 billion |

Takeaways: **file bytes are huge, but metadata is small and hot.** Keep them in different systems: object storage for bytes, a transactional database for metadata.

---

## Core Idea: Split Files into Chunks

Instead of treating a file as one blob, the client splits it into **chunks** (commonly around 4 MB), and each chunk is identified by a **hash** of its content (for example SHA-256).

**Why chunk?**

| Benefit | How |
|---|---|
| **Resumable uploads** | If the connection drops, re-upload only the missing chunks, not the whole file |
| **Delta sync** | Edit a small part of a large file and only the changed chunks upload |
| **Deduplication** | Identical chunks (the same file uploaded twice, or shared across files and versions) are stored **once** |
| **Parallelism** | Upload and download many chunks at the same time |
| **Efficient storage layout** | Fixed-size units are easy to place, replicate, and repair |

### Chunk Size Trade-off

- **Small chunks:** better dedup and finer deltas, but more metadata, more requests, more overhead.
- **Large chunks:** less overhead, but weaker dedup and larger re-uploads on small edits.
- Typically a few MB is a good balance.

### Fixed-Size vs Content-Defined Chunking

- **Fixed-size:** simple, but inserting a byte near the start shifts every later chunk, so all hashes change.
- **Content-defined chunking** (using a rolling hash such as Rabin fingerprinting) places chunk boundaries based on the content itself, so an insertion changes only nearby chunks. It gives much better dedup and delta sync for edited files, at the cost of more computation on the client.

---

## Data Model

### Metadata Database (relational, strongly consistent)

**`files` table** (one row per file or folder entry)

| Column | Type |
|---|---|
| `file_id` | UUID |
| `namespace_id` | UUID (the user's root or a shared folder) |
| `parent_id` | UUID (folder) |
| `name` | VARCHAR |
| `is_folder` | BOOLEAN |
| `current_version` | INT |
| `deleted` | BOOLEAN (soft delete for trash) |
| `modified_at` | TIMESTAMP |

**`file_versions` table**

| Column | Type |
|---|---|
| `file_id` | UUID |
| `version` | INT |
| `size_bytes` | BIGINT |
| `chunk_hashes` | ordered list of chunk hashes (or a separate `version_chunks` table) |
| `modified_by`, `modified_at` | UUID, TIMESTAMP |

**`permissions` table:** `namespace_id` or `file_id`, `principal_id`, `role` (viewer, editor, owner).

**`change_log` table** (a journal per namespace)

| Column | Type |
|---|---|
| `namespace_id` | UUID |
| `sequence_no` | BIGINT, increasing |
| `file_id`, `change_type` | UUID, ENUM (create, update, delete, move) |

**`devices` table:** `device_id`, `user_id`, `last_cursor` (last change sequence applied).

### Chunk Index and Storage

- **Chunk index:** key = chunk hash → `{storage location, size, reference count}`. A huge key-value table, **sharded by hash**.
- **Chunk bytes:** stored in **object storage** (for example S3-style), not in the database.

---

## Architecture

```
┌───────────────┐   1. chunk + hash          ┌─────────────────────────┐
│ Client app    │ ──────────────────────────►│ Metadata / Sync Service │──► Metadata DB (sharded)
│ (watcher,     │   2. which chunks missing? │  (commit, versions,     │──► Change log / cursors
│  local DB)    │ ◄──────────────────────────│   permissions)          │
│               │   3. upload missing chunks └────────────┬────────────┘
│               │ ─── via pre-signed URLs ──► Object Storage (chunks)   │ events
│               │   4. commit new version                               ▼
│               │ ──────────────────────────► Notification Service ──► other devices
└───────────────┘                              (WebSocket / long poll / push)
```

### Components

| Component | Responsibility |
|---|---|
| **Client app** | Watches the folder, chunks and hashes files, uploads, applies remote changes, keeps a local database of state |
| **Metadata / Sync Service** | The source of truth for the file tree, versions, permissions, and the change log |
| **Block (chunk) service** | Issues pre-signed URLs, tracks which chunks exist, manages reference counts |
| **Object storage** | Durable storage for chunk bytes (replicated or erasure-coded across zones) |
| **Notification service** | Tells other devices that something changed |
| **Sharing service** | Permissions, share links |
| **Background workers** | Garbage collection, thumbnails and previews, virus scanning, indexing for search |

---

## Upload Flow

1. The client detects a new or changed file and **splits it into chunks**, computing each chunk's **hash**.
2. It sends the list of hashes to the Sync Service: *"I want to commit this file; which chunks do you not have?"*
3. The server checks the chunk index and replies with the **missing** hashes (deduplication: chunks that already exist are skipped, even if uploaded by another file or user).
4. The client uploads **only the missing chunks, in parallel, directly to object storage using pre-signed URLs**. Large file bytes never pass through the application servers.
5. If the connection drops, the client **retries only the chunks that failed**.
6. When all chunks are stored, the client calls **commit** with the file's metadata and ordered chunk list and its **base version**.
7. The Sync Service, in **one database transaction**, creates the new file version, updates the file's current version, and appends an entry to the **change log**.
8. The Notification Service tells the user's other devices (and collaborators) that there is something new.

**Order matters:** chunks are stored **before** the metadata commit. If the client crashes in between, you only have orphan chunks (cleaned up later), never a file pointing at missing data.

### Handling Very Large Files

Upload chunks in parallel, resume after interruptions (the client remembers progress or asks the server what exists), and use **multipart upload** facilities of the object store. Throttle bandwidth so syncing doesn't saturate the user's network.

---

## Download Flow

1. The client asks the Sync Service for the file's current version metadata (after a permission check).
2. It receives the **ordered chunk list** and **short-lived signed URLs** (or a CDN URL) for each chunk.
3. It downloads chunks **in parallel**, verifies each **hash** (detecting corruption), and reassembles the file.
4. Popular or shared files can be served through a **CDN** to reduce load and latency. See [Caching and CDN](system-design/caching-and-cdn).
5. Devices cache chunks locally, so they download only what changed.

---

## Syncing Devices

Each device keeps a **cursor**: the sequence number of the last change it has applied.

```
Device A edits report.docx → commit → change_log gets entry #1042
Notification service tells Device B: "namespace has new changes"
Device B asks: "give me changes after #1041" → receives entry #1042
Device B fetches the changed chunks and updates its local copy; cursor = 1042
```

- **Pull with a cursor** is robust: a device that was offline for a week simply asks for everything after its last cursor.
- The **notification** is only a hint to pull now. It can be sent over a **long-lived connection** (WebSocket, long polling) on desktops, or **push notifications** on mobile to save battery. If the hint is lost, the device still catches up on its next periodic poll.
- The change log is **per namespace** and ordered, so devices apply changes in the same order.

---

## Conflicts

Two devices edit the same file at the same time (often while one was offline). Each commit includes the **base version** it was editing.

- The first commit succeeds and creates version N+1.
- The second commit sees its base version (N) is no longer current. This is a **conflict**.

Resolution options:

| Strategy | Notes |
|---|---|
| **Keep both** ("conflicted copy") | Rename the losing version, for example `report (Rahul's conflicted copy).docx`. Safe, never loses data, and the common choice for arbitrary file types |
| **Automatic merge** | Possible for text or structured documents (operational transformation or CRDTs, as in collaborative editors) |
| **Last-writer-wins** | Simple but silently discards edits, so avoid for user files |

The metadata commit uses **optimistic concurrency** (compare the base version), so conflicts are detected reliably. See [Distributed Systems Essentials](system-design/distributed-systems-essentials).

---

## Deduplication Details

- **Chunk-level dedup:** the chunk index maps hash → stored chunk with a **reference count**. Uploading an existing chunk just increments the count.
- **Scope:** dedup **within one account** is safe. Dedup **across all users** saves more space but leaks information (a user can learn whether someone else already stores a particular file by seeing that the upload was skipped), so providers treat cross-user dedup carefully, for example by limiting it or protecting it cryptographically.
- **Encryption interacts with dedup:** if every user encrypts chunks with their own key, identical files produce different ciphertext and can't be deduplicated. It is a trade-off between privacy and storage savings.

---

## Metadata Storage and Scaling

- Use a **relational database** (MySQL or PostgreSQL) for the file tree, versions, and permissions: commits need **transactions** and strong consistency.
- **Shard by namespace (user or shared folder)** so most operations touch a single shard. Shared folders that cross users need care (give shared folders their own namespace).
- **Cache** hot metadata (folder listings, permissions) in Redis, and invalidate on change. See [Databases](system-design/databases).
- The **chunk index** is a separate key-value store sharded by hash (consistent hashing).
- The change log is append-only and read by cursor, which is efficient to shard and scan.

---

## Durability and the Storage Layer

- **Object storage** keeps data across multiple availability zones using **replication** (for example 3 copies) or **erasure coding** (splitting data plus parity fragments so lost pieces can be rebuilt with less storage overhead than full copies).
- **Checksums** verify integrity on write and read, and background **scrubbing** detects and repairs silent corruption.
- **Cross-region replication** protects against region loss (see [High Availability](system-design/high-availability) and [Observability and Deployment](system-design/observability-and-deployment) for disaster recovery).
- **Tiering:** keep recent, frequently used chunks on fast storage and move old versions and rarely accessed data to cheaper, colder tiers.
- Replication is not a backup: keep **version history and a trash period** so accidental deletion or ransomware overwrite can be undone.

---

## Garbage Collection and Deletion

- Deleting a file marks it deleted (soft delete) and moves it to **trash** for a retention period; **old versions** are pruned by policy.
- When a version is permanently removed, decrement the **reference count** of its chunks. A chunk with a count of zero is deleted **after a grace period**, in a background job.
- The grace period protects against races, for example a chunk uploaded and about to be referenced by a commit still in flight.
- **Orphan chunks** (uploaded but never committed) are found and removed by periodic sweeps.

---

## Sharing and Permissions

- **Share with specific users:** grant viewer or editor roles on a file or folder; permissions inherit down the folder tree.
- **Share links:** unguessable tokens (long random strings) with optional **expiry, password, and view-only** settings, revocable at any time.
- Every metadata and download request is **authorized on the server**; signed chunk URLs are short-lived so they can't be reused later. See [Authentication and Security](system-design/auth-and-security).
- Shared folders appear in each member's tree and generate change-log entries for all members.

---

## Security

- **TLS** for all transfers; **encryption at rest** for chunks, with keys in a key management service. Optionally **client-side (end-to-end) encryption** for users who want the provider to be unable to read their data, at the cost of losing server-side features such as previews, search, and cross-user dedup.
- **Malware scanning** of uploaded files, and abuse detection.
- **Audit logs** for sharing and access, and rate limiting to prevent abuse.

---

## Bottleneck Resolutions

| Bottleneck | Solution |
|---|---|
| Large file uploads over unreliable networks | Chunking, parallel and resumable uploads, pre-signed URLs to object storage |
| Bandwidth and storage cost | Chunk-level deduplication, compression, delta sync with content-defined chunking |
| Application servers handling file bytes | Clients upload and download directly to object storage and the CDN |
| Metadata hot spots | Shard by namespace, cache, keep transactions small |
| Huge chunk index | Shard the hash → location store; reference counts |
| Many devices needing updates | Per-namespace change log with cursors, plus push hints |
| Concurrent edits | Optimistic concurrency with conflict copies |
| Data loss risk | Multi-zone replication or erasure coding, checksums and scrubbing, versions, trash |
| Orphan and unreferenced chunks | Reference counting and delayed garbage collection |

---

## Interview Questions

**Q: Why split files into chunks?**
For resumable and parallel transfers, syncing only changed parts, and deduplicating identical chunks across files and versions, which saves bandwidth and storage.

**Q: How do you avoid re-uploading data the server already has?**
The client sends chunk hashes first, and the server replies with the missing ones; only those are uploaded.

**Q: Why use pre-signed URLs?**
So clients upload and download directly to object storage, keeping large file traffic away from application servers, while the URLs are scoped and short-lived.

**Q: How do devices stay in sync?**
Each commit appends an entry to a per-namespace change log. A notification prompts devices to pull changes after their cursor, and a device that was offline simply catches up from its last cursor.

**Q: How do you handle two devices editing the same file?**
Commits include the base version. If the base is stale, the commit is a conflict and the system keeps both versions (a conflicted copy), or merges automatically where the format allows.

**Q: Why is metadata separate from file data, and what do you use for each?**
Metadata is small, hot, and needs transactions and strong consistency (relational database, sharded). File data is huge and immutable-ish, so it goes in cheap, durable, scalable object storage.

**Q: How do you delete chunks safely when they might be shared?**
Reference counting: each version referencing a chunk increments the count, deleting a version decrements it, and chunks at zero are removed by a background job after a grace period.

**Q: What are the privacy concerns of deduplication?**
Cross-user dedup can reveal whether someone else already stores a file; encryption with per-user keys prevents dedup. Providers balance savings against privacy.

**Q: How do you ensure durability?**
Replicate or erasure-code across availability zones, verify checksums, scrub continuously, replicate across regions, and keep versions and trash for human mistakes.

---

## Summary

| Component | Choice |
|---|---|
| File representation | Ordered list of content-hashed chunks (about 4 MB), ideally content-defined |
| Chunk storage | Object storage, replicated or erasure-coded, reference-counted |
| Upload path | Hash first, upload missing chunks via pre-signed URLs, then transactional metadata commit |
| Metadata | Relational DB sharded by namespace; chunk index sharded by hash |
| Sync | Per-namespace change log, device cursors, push or long-poll hints |
| Conflicts | Optimistic concurrency on base version; keep both copies |
| Dedup | Chunk-level; careful with cross-user scope and encryption |
| Cleanup | Reference counts, grace-period garbage collection, trash and version retention |
| Security | TLS, encryption at rest (optionally client-side), signed short-lived URLs, audit logs |

## Continue Learning

- [Caching and CDN](system-design/caching-and-cdn) — serving downloads at the edge.
- [Databases](system-design/databases) — sharding metadata and transactional commits.
- [Distributed Systems Essentials](system-design/distributed-systems-essentials) — optimistic concurrency, conflicts, and CRDTs.
- [Design Netflix](system-design/netflix-design) — object storage and CDN delivery for video.
- [Authentication and Security](system-design/auth-and-security) — permissions, signed URLs, and encryption.
