---
title: "System Design Case Study — Netflix (Video Streaming at Global Scale)"
description: "A full system design for a Netflix-like video streaming platform: ingest and encoding pipeline, adaptive bitrate streaming, DRM, the Open Connect CDN model, playback flow, search, recommendations, and capacity planning anchored on real viewing numbers."
author: ["name": "Rajendra Pancholi", "email": "rpancholi522@gmail.com"]
created: "2026-04-18"
updated: "2026-10-17"
thumbnail: "/images/netflix-design.png"
tags: [system-design, netflix, video-streaming, cdn, case-study, transcoding, drm]
keywords: ["Netflix system design", "Video streaming architecture", "Adaptive bitrate streaming", "Open Connect CDN", "Video transcoding pipeline", "DRM video streaming", "System design interview Netflix"]
---

# System Design Case Study — Netflix (Video Streaming at Global Scale)

Netflix serves hundreds of millions of paid memberships (over 325 million reported at the end of 2025) in more than 190 countries, and its own network delivers well over 100 million hours of viewing every day. Building a platform at this scale requires solving fundamental challenges in video encoding, global content delivery, adaptive streaming, content protection, and personalized recommendations.

A key point to understand first: Netflix is **not YouTube**. Netflix licenses or produces its own catalog, so there are **no user uploads, no user comments, and no copyright-fingerprinting of uploads**. The catalog is small (tens of thousands of titles), but the **viewing traffic is enormous**. The hard problem is delivering video, not storing it.

![Netflix System Design](/images/netflix-design.png)

## Requirements

### Functional Requirements

- Users can stream TV shows and movies on any device (web, iOS, Android, Smart TV)
- The content team ingests new titles; the platform encodes and distributes them
- Users can search content by title, actor, genre, or tag
- Resume playback from where the user left off, across devices
- Users can rate titles (thumbs up/down) and see personalized recommendations
- Multiple profiles per account

### Non-Functional Requirements

- High availability — streaming must keep working during partial infrastructure failures
- Low latency — video should start within a couple of seconds on a good connection
- Adaptive quality — video quality adjusts to the user's bandwidth in real time
- Content protection — video must be encrypted and licensed per device
- Scales to tens of millions of concurrent streams

### Extended Requirements

- Geographic content restrictions (licensing agreements by country)
- Subtitles and multiple audio tracks (many languages)
- Metrics and analytics for content performance

> **Variant:** if an interviewer asks for a **YouTube-style** platform instead, add user uploads, comments, and an upload moderation step (copyright fingerprinting, NSFW detection). Those belong to YouTube, not Netflix.

## Capacity Estimation

Anchor the estimate on a real number instead of guessing user counts: Netflix has stated that its Open Connect network delivers **over 125 million hours of viewing per day**. The rest uses stated assumptions.

```
Viewing: 125M hours/day
Average concurrent streams: 125M / 24 ≈ 5.2 million
Peak concurrent streams (assume 2.5x average): ≈ 13 million

Average bitrate (assume ~3 Mbps across all devices and qualities):
Average egress: 5.2M streams × 3 Mbps ≈ 15.6 Tbps
Peak egress: ≈ 15.6 × 2.5 ≈ 39 Tbps

Data delivered per day:
3 Mbps × 3,600 s = 1.35 GB per viewing hour
125M hours × 1.35 GB ≈ 169 PB/day
```

**Storage (the small part):**

```
Assume a library of 50,000 hours of content
Assume ~20 GB per hour across all renditions (resolutions, codecs, audio, subtitles)
50,000 × 20 GB ≈ 1 PB of encoded content at the origin
```

| Metric             | Estimate                                                                                       |
| ------------------ | ---------------------------------------------------------------------------------------------- |
| Daily viewing      | ~125 million hours (published figure)                                                          |
| Concurrent streams | ~5 million average, ~13 million peak                                                           |
| Egress bandwidth   | ~16 Tbps average, tens of Tbps at peak                                                         |
| Data delivered/day | ~170 PB                                                                                        |
| Origin library     | ~1 PB (small)                                                                                  |
| New content        | Thousands of titles/year: low volume, but each source file can be hundreds of GB to several TB |

**Takeaways:** storage is tiny, **bandwidth is the real cost**, and almost all the traffic is reads of the same popular files, which is exactly what caching at the edge solves. No origin or application server could carry tens of Tbps, so **video bytes must never flow through your application servers**.

## Data Model

### `users` / `profiles`

| Column                       | Type                                             |
| ---------------------------- | ------------------------------------------------ |
| `account_id`                 | UUID                                             |
| `email`                      | VARCHAR                                          |
| `subscription_plan`          | ENUM (basic, standard, premium)                  |
| `country`                    | VARCHAR                                          |
| `profile_id`, `profile_name` | UUID, VARCHAR (one account has several profiles) |

### `videos` table (title metadata)

| Column             | Type                                         |
| ------------------ | -------------------------------------------- |
| `id`               | UUID                                         |
| `title`            | VARCHAR                                      |
| `description`      | TEXT                                         |
| `duration_seconds` | INT                                          |
| `genre`            | ENUM                                         |
| `content_rating`   | ENUM                                         |
| `manifest_path`    | VARCHAR (location of the streaming manifest) |
| `created_at`       | TIMESTAMP                                    |

### `video_availability` table (licensing)

| Column            | Type      |
| ----------------- | --------- |
| `video_id`        | UUID      |
| `country`         | VARCHAR   |
| `available_from`  | TIMESTAMP |
| `available_until` | TIMESTAMP |

> Licensing is per country **and per time window**, so a simple list of countries on the video row is not enough.

### `watch_progress` (resume position)

| Column           | Type      |
| ---------------- | --------- |
| `profile_id`     | UUID      |
| `video_id`       | UUID      |
| `offset_seconds` | INT       |
| `updated_at`     | TIMESTAMP |

Key: `(profile_id, video_id)`. This holds **only the latest position** per title, so reads and writes are cheap key lookups. The player sends a heartbeat every 30–60 seconds while playing, plus on pause or exit.

### `viewing_events` (analytics log, append-only)

Every play, pause, seek, and completion is sent as an event to Kafka and stored for analytics and recommendations. It is kept separate from `watch_progress` so the resume path stays small and fast.

### `ratings` table

| Column       | Type                                     |
| ------------ | ---------------------------------------- |
| `profile_id` | UUID                                     |
| `video_id`   | UUID                                     |
| `rating`     | ENUM (thumbs_down, thumbs_up, double_up) |
| `created_at` | TIMESTAMP                                |

### `tags` table

| Column     | Type    |
| ---------- | ------- |
| `video_id` | UUID    |
| `tag`      | VARCHAR |

## API Design

```
// Start playback (control plane)
POST /videos/{videoID}/playback
Body: { profileID, deviceInfo, capabilities (codecs, DRM, max resolution) }
Response 200: {
  manifestURLs: [ranked list of CDN URLs],
  drmLicenseURL, playbackToken, resumeOffsetSeconds
}
Response 403: not available in this country / plan / device

// Media segments: fetched by the player straight from the CDN
GET {cdn-host}/{videoID}/{rendition}/{segment}  (signed, short-lived URL)

// Report progress (heartbeat)
PUT /profiles/{profileID}/progress/{videoID}
Body: { offsetSeconds }
Response 204

// Search
GET /search?q={query}&genre={genre}&cursor={cursor}
Response: [{ video, matchScore }]

// Recommendations (personalized home page)
GET /profiles/{profileID}/home
Response: [{ rowTitle, videos: [...] }]

// Rate a title
PUT /profiles/{profileID}/ratings/{videoID}
Body: { rating }
Response 204

// Ingest a new title (internal tool, content team)
POST /ingest/titles
Response 201: { uploadID, presignedUploadURLs }
```

**Why the playback call is separate from the video bytes:** the API server only authorizes playback and hands the player a manifest, DRM license info, and ranked CDN URLs. The player then downloads video segments **directly from the CDN edge**, never through the application servers. Large uploads similarly go straight to object storage through pre-signed URLs, not through the API.

## Architecture

Netflix's architecture has two distinct halves:

- **Control plane:** everything except the video bytes — login, browsing, search, recommendations, playback authorization. Runs on cloud infrastructure (AWS) and scales like any large web system.
- **Data plane:** the video bytes themselves, delivered by Netflix's own CDN, **Open Connect**.

### Core Services

| Service                        | Responsibility                                                                                |
| ------------------------------ | --------------------------------------------------------------------------------------------- |
| **User / Profile Service**     | Authentication, subscription management, profiles                                             |
| **Playback Service**           | Authorizes playback (plan, country, device), picks CDN servers, returns manifest and DRM info |
| **Ingest / Media Service**     | Receives master files, runs the encoding pipeline                                             |
| **Catalog / Metadata Service** | Titles, artwork, licensing windows                                                            |
| **Search Service**             | Search over the catalog                                                                       |
| **Recommendation Service**     | Personalized rows and ordering                                                                |
| **Analytics Service**          | Viewing events and content performance                                                        |

## Video Processing Pipeline

A studio delivers a high-quality **master file** (for a two-hour 4K film this can be hundreds of GB to several TB). Before streaming, it must be validated and encoded into many formats and quality levels.

![Video Processing Pipeline](/images/video-processing-pipeline.png)

The pipeline is **event-driven** and runs on many workers in parallel, coordinated through queues and workflows (see [Event-Driven Architecture](system-design/event-driven)).

### Stage 1: Ingest and Quality Control

- Receive the master through a secure upload into object storage (resumable, multipart).
- **Validate** the source: format, frame rate, resolution, audio tracks, subtitles, color information, checksums. Files that fail are rejected back to the content team (a technical QC step, not copyright screening).

### Stage 2: Chunking for Parallel Encoding

The video is split into chunks at **shot or scene boundaries** (typically seconds to minutes long) so that many workers can **encode different chunks in parallel**. Encoding a feature film as one long job would take far too long; splitting it lets thousands of cores work at once. Splitting at scene boundaries also avoids visible quality jumps at chunk joins.

### Stage 3: Encoding (Transcoding)

Each chunk is encoded with several codecs for different devices:

- **H.264/AVC** — widest device compatibility
- **H.265/HEVC** — roughly 30–50% better compression than H.264; common for 4K and HDR
- **VP9** — open codec, common on many devices
- **AV1** — newest open codec with the best compression, but higher encoding cost and needs newer devices

Tools: [FFmpeg](https://ffmpeg.org) (open source), [AWS Elemental MediaConvert](https://aws.amazon.com/mediaconvert) (managed).

### Stage 4: Bitrate Ladder and Per-Title Encoding

Each title is encoded at multiple **resolution and bitrate combinations** (a "ladder"). Illustrative H.264 values:

| Resolution | Bitrate (H.264, illustrative) | Bandwidth needed |
| ---------- | ----------------------------- | ---------------- |
| 4K (2160p) | ~15-25 Mbps                   | 25+ Mbps         |
| 1080p      | ~4-8 Mbps                     | 5+ Mbps          |
| 720p       | ~2.5-4 Mbps                   | 3+ Mbps          |
| 480p       | ~1-2 Mbps                     | 1.5+ Mbps        |
| 360p       | ~0.5-1 Mbps                   | 0.7+ Mbps        |

HEVC and AV1 need noticeably less at the same quality. Rather than one fixed ladder for every title, large platforms use **per-title (and even per-shot) encoding**: a flat animated show looks great at a much lower bitrate than a dark, grainy action film, so the ladder is tuned for each title to save bandwidth without losing quality.

### Stage 5: Packaging, Encryption, and Storage

- The encoded video is **packaged into short segments** (a few seconds each) with a **manifest** file listing every quality level and segment.
- Segments are **encrypted** for DRM (next section).
- The result is stored in object storage (for example Amazon S3) as the **origin**, then pushed out to the CDN:

```
s3://videos/{videoID}/{codec}/{resolution}/{segment_number}.m4s
s3://videos/{videoID}/manifest
```

## Adaptive Bitrate Streaming (ABR)

Users have vastly different network conditions, from fiber (100+ Mbps) to mobile data (1–5 Mbps) that fluctuates. **Adaptive Bitrate Streaming** adjusts quality continuously.

**Protocols:** **MPEG-DASH** and **HLS** are the two standard HTTP-based protocols; both work the same way (Netflix mainly uses DASH-style streaming, and HLS is common on Apple devices). Segments are plain HTTP files, so they can be cached by any CDN.

**How it works:**

1. The player requests the **manifest**, which lists all quality levels and segment URLs.
2. It picks a conservative **initial quality** so playback starts quickly.
3. It downloads segments one by one, **measuring throughput and watching its buffer level**.
4. The **player decides** which quality to request for the next segment: if throughput drops or the buffer is draining, switch down; if there's headroom, switch up.
5. Switches happen at segment boundaries, so the viewer sees a smooth change, not a stall.

The intelligence is **in the client**, not the server, so the server side stays simple and cacheable.

**Resume playback:** the playback response includes the saved `resumeOffsetSeconds`; the player simply starts by requesting the segment containing that timestamp.

## Content Protection (DRM)

Studios require that video can't be copied freely, so streams are **encrypted** and decrypted only on authorized devices:

1. Segments are encrypted at packaging time (a common-encryption scheme lets one encrypted file work with several DRM systems).
2. The player, at playback start, requests a **license** (decryption key) from a **license server**, presenting its playback token and device credentials.
3. The license server checks the account, plan, country, and device security level, and may limit the maximum resolution on less secure devices.
4. Major DRM systems: **Widevine** (Android, Chrome), **FairPlay** (Apple), **PlayReady** (Microsoft). Multi-DRM supports all of them.
5. Segment URLs are **signed and short-lived** so they can't be shared.

## Playback Flow and Content Delivery: Open Connect

For most services, a third-party CDN (CloudFront, Fastly, Cloudflare) is the right choice. At Netflix's scale, they built their own CDN, **Open Connect**.

**How Open Connect works:**

- Netflix places **Open Connect Appliances (OCAs)**, purpose-built video cache servers, **inside ISP networks** and at internet exchange points, in close to 1,000 locations worldwide.
- Netflix states that Open Connect delivers **100% of its video traffic**. The origin servers only **fill** the appliances.
- **Fill happens off-peak:** popular content is **pre-positioned** on OCAs during quiet hours, based on predicted demand, so peak-time viewing doesn't compete with fill traffic.
- Because the appliance sits inside or next to the viewer's ISP, video travels a very short network path.

**What happens when a user presses Play:**

1. The player calls the **Playback Service**.
2. It checks the user's plan, country, and device, and returns a signed manifest plus DRM information.
3. It also decides which OCAs should serve this viewer (nearest, healthy, **holding that content**, not overloaded) and returns a **ranked list** of CDN URLs.
4. The player fetches segments from the first OCA. If that OCA is slow or fails, it **switches to the next one** in the list.

**For a standard system design interview, use a managed CDN:**

| CDN                                                    | Notes                              |
| ------------------------------------------------------ | ---------------------------------- |
| [Amazon CloudFront](https://aws.amazon.com/cloudfront) | Tight AWS integration; Lambda@Edge |
| [Cloudflare CDN](https://www.cloudflare.com/cdn)       | Global network; DDoS protection    |
| [Fastly](https://www.fastly.com)                       | Fast purging; programmable edge    |

See [Caching and CDN](system-design/caching-and-cdn) for edge caching, TTLs, and invalidation.

## Search Architecture

The catalog is small (tens of thousands of titles), so the challenge is **query volume and relevance**, not data size.

- Use **Elasticsearch** (or OpenSearch): index `title`, `description`, `tags`, `cast`, `director`, `genre`
- Full-text search with relevance scoring, typo tolerance, and **typeahead** suggestions
- Faceted filtering: genre, year, rating, duration
- **Availability filtering:** only return titles licensed in the user's country
- A small index doesn't need many shards, but needs **replicas** and caching for high read throughput

**Trending content:** count searches and plays over the last N hours in a stream-processing job, refresh rankings every few minutes, and weight by popularity and freshness.

## Recommendation Engine

Recommendations drive most of what people watch. A typical design has two stages:

1. **Candidate generation:** from the whole catalog, quickly narrow to a few hundred titles using collaborative filtering ("members with similar viewing also watched X"), content similarity, and popularity.
2. **Ranking:** a machine learning model orders those candidates for this specific profile and builds the **rows** of the home page.

Signals include viewing history (titles, completion, time of day), searches, ratings, device type, and which rows the profile interacts with or ignores. Even the **artwork** shown for a title can be personalized.

**Practical points:**

- Models are **trained offline** on large batches of viewing events; results are **precomputed and cached** per profile and refreshed every few hours, with lighter real-time adjustments.
- **Cold start:** new users get popularity- and onboarding-based picks; new titles are promoted using content similarity.
- Don't compute recommendations on the request path; serve them from a cache.

## Geo-Blocking and Entitlements

Licensing restricts which titles can be watched in which countries and when.

1. Determine the user's country from IP geolocation and account information (and detect proxies/VPNs where required).
2. **Catalog level:** only show titles available in that country (`video_availability`, checking the license window).
3. **Playback level:** the Playback Service **re-checks entitlement** before issuing a manifest and DRM license; this is the real enforcement point.
4. If not allowed, respond with a clear "not available in your region" error (HTTP 403 is typical; 451 is reserved for legal demands).
5. Edge geo-restrictions on the CDN add a second layer of protection.

## Advanced Architecture

![Netflix Advanced Design](/images/netflix-advanced-design.png)

### Data Partitioning

- **Title metadata and accounts:** a relational database with read replicas (infrequent writes, heavy reads), with a cache in front
- **Watch progress and viewing history:** a wide-column store such as Apache Cassandra (very high write throughput, queried by user/profile key)
- **Search index:** Elasticsearch cluster with replicas
- **Origin video:** object storage (S3), replicated across regions for durability

See [Databases](system-design/databases) for these trade-offs.

### Caching Strategy

- **Metadata cache:** Redis with TTL (title info, artwork URLs); see [Caching and CDN](system-design/caching-and-cdn)
- **Search results cache:** popular queries for a few minutes
- **Recommendation cache:** per profile, refreshed every few hours
- **Video:** at the CDN edge (OCAs)

### Analytics Pipeline

```
Player events (play, pause, seek, error, quality switches)
  → Kafka
  → Stream processing (Spark / Flink) for real-time metrics
  → Time-series / OLAP store (Druid, etc.) → dashboards

Batch:
  → data lake (S3) → batch jobs → ML training for recommendations
```

Quality-of-experience metrics (start time, rebuffering, bitrate) are watched closely, because they show problems before users complain. See [Messaging and Communication](system-design/messaging-and-communication) for Kafka.

### Resilience

At this scale, failures are normal, so the design assumes them: services run across multiple availability zones and regions, calls use timeouts, retries, and circuit breakers, and non-critical features (such as personalized rows) **degrade gracefully** to cached or generic content so that **playback keeps working**. See [High Availability](system-design/high-availability) and [Circuit Breaker Pattern](system-design/circuit-breaker).

## Bottleneck Resolution

| Bottleneck                       | Solution                                                                                                   |
| -------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| Egress bandwidth (tens of Tbps)  | Own CDN with appliances inside ISP networks, pre-positioned content, video never flows through app servers |
| Encoding time for long films     | Split at shot boundaries and encode chunks in parallel on many workers                                     |
| Bandwidth cost                   | Per-title encoding, modern codecs (HEVC/AV1)                                                               |
| Unpredictable network quality    | Adaptive bitrate streaming with client-side switching                                                      |
| Slow start                       | Conservative initial quality, small segments, nearby edge servers                                          |
| CDN server failure or overload   | Ranked list of servers; the player fails over to the next                                                  |
| Search latency and QPS           | Cache hot queries, replicas, typeahead from a small index                                                  |
| Recommendation freshness vs cost | Offline training, precomputed and cached results                                                           |
| Content piracy                   | DRM, signed short-lived URLs, device security levels                                                       |
| Control-plane outage             | Multi-region deployment, graceful degradation, circuit breakers                                            |

## Interview Questions

**Q: Why is bandwidth the main challenge for a streaming service rather than storage?**
The catalog is small (about a petabyte of encodes), but millions of concurrent streams at a few Mbps each mean tens of Tbps of egress. That's solved by caching video at the edge, close to viewers.

**Q: How does adaptive bitrate streaming work?**
Video is split into short segments encoded at several quality levels, described by a manifest. The player measures throughput and buffer level and picks the quality of each next segment, switching at segment boundaries.

**Q: Why not stream video through your application servers?**
Their capacity is tiny compared with the traffic. The API only authorizes playback and returns manifest and CDN URLs; the player downloads segments directly from edge servers.

**Q: How would you speed up encoding a two-hour 4K film?**
Split the master into chunks at scene boundaries and encode them in parallel across many workers, then assemble the results.

**Q: How do you protect content from piracy?**
Encrypt segments, issue decryption keys from a license server after checking account, region, and device, use multi-DRM (Widevine, FairPlay, PlayReady), and serve segments through signed, short-lived URLs.

**Q: How do you implement "resume watching" across devices?**
Store only the latest offset per (profile, title) in a key-value or wide-column store, updated by player heartbeats every 30–60 seconds; the playback call returns it so any device can resume.

**Q: Why build a custom CDN?**
At very large scale, placing appliances inside ISP networks gives lower cost and better quality than renting a third-party CDN. For most systems, a managed CDN is the right choice.

**Q: What if the recommendation service is down?**
Degrade gracefully: show cached or popular rows. Browsing and, above all, playback must continue to work.

## Summary

| Component                  | Technology                                                                             |
| -------------------------- | -------------------------------------------------------------------------------------- |
| Source and origin storage  | Object storage (Amazon S3), multi-region                                               |
| Encoding                   | FFmpeg and cloud encoders; parallel chunked, per-title encoding; H.264, HEVC, VP9, AV1 |
| Streaming protocol         | MPEG-DASH / HLS (adaptive bitrate, client-driven)                                      |
| Content protection         | Multi-DRM (Widevine, FairPlay, PlayReady), signed URLs                                 |
| Video delivery             | Netflix Open Connect (or a managed CDN such as CloudFront)                             |
| Control plane              | Stateless services on cloud infrastructure across regions                              |
| Metadata DB                | Relational database with read replicas and cache                                       |
| Watch progress and history | Apache Cassandra (or similar wide-column store)                                        |
| Search                     | Elasticsearch / OpenSearch                                                             |
| Recommendations            | Candidate generation + ranking models, trained offline, cached per profile             |
| Analytics                  | Kafka + Spark/Flink + Druid                                                            |
| Caching                    | Redis for metadata; edge caches for video                                              |

## Continue Learning

- [Caching and CDN](system-design/caching-and-cdn) — edge caching, TTLs, and how CDNs deliver content.
- [Databases](system-design/databases) — choosing relational vs wide-column stores.
- [Messaging and Communication](system-design/messaging-and-communication) — Kafka for event pipelines.
- [Event-Driven Architecture](system-design/event-driven) — the pipeline behind encoding workflows.
- [High Availability](system-design/high-availability) — designing for failure across regions.
