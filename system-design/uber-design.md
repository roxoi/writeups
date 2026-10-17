---
title: "System Design Case Study — Uber (Ride-Hailing at Global Scale)"
description: "A complete system design for building Uber: real-time location tracking, geospatial driver matching with geohash, Quadtree and H3, safe dispatch without double-booking, surge pricing, idempotent payment processing, and handling 100M daily active users."
author: ["name": "Rajendra Pancholi", "email": "rpancholi522@gmail.com"]
created: "2026-04-18"
updated: "2026-10-17"
thumbnail: "/images/uber-system-design.png"
tags: [system-design, uber, geospatial, quadtree, websockets, idempotency, case-study, location-tracking]
keywords: ["Uber system design", "Ride hailing architecture", "Geospatial indexing system design", "Quadtree geohash H3", "Real-time location tracking", "Distributed lock driver dispatch", "Idempotent payments", "System design interview Uber"]
---

# System Design Case Study — Uber (Ride-Hailing at Global Scale)

Uber handles tens of millions of trips per day across the world, matching riders with nearby drivers in seconds, tracking live locations, managing surge pricing, and handling payments — all in real time. This is one of the most technically challenging system designs because it combines real-time geospatial processing, bidirectional communication, and strict consistency requirements around trip state and money.

The numbers below are **design assumptions** for an interview, not Uber's real figures.

![Uber System Design](/images/uber-system-design.png)

## Requirements

### Functional Requirements

**For Customers:**

- View all available cabs in the vicinity with ETA and pricing
- Book a ride specifying source, destination, and cab type
- Track the driver's live location during pickup and trip
- Cancel a ride before pickup
- Rate the trip and driver after completion

**For Drivers:**

- Accept or deny incoming ride requests
- See the customer's pickup location after accepting
- Mark the trip as started and completed
- View earnings history

### Non-Functional Requirements

- High reliability — rides must never be silently lost
- High availability with minimal latency (< 1 second for driver matching)
- Horizontally scalable to support 100M+ DAU globally
- Strongly consistent trip state (a trip cannot simultaneously be accepted by two drivers, and a driver cannot hold two trips)
- **No double charging** — payments must be safe to retry

### Extended Requirements

- Surge pricing based on demand/supply imbalance
- Payment processing via third-party payment gateways
- Fraud detection for payment and account abuse
- Analytics for route optimization and business intelligence

## Capacity Estimation

- **100 million DAU(Daily Active Users)**, 1 million active drivers
- 10 million rides/day

```
If each user performs 10 actions (check fares, request, track, etc.):
Total requests: 100M × 10 = 1 billion/day
RPS: 1B / 86,400 ≈ 12,000 requests/second

Location updates (drivers send GPS every 5 seconds):
1M drivers × (86,400 / 5) ≈ 17 billion location updates/day
Location update RPS: ~200,000/second (if all 1M drivers are online at once)
```

**Storage:**

```
1 billion actions × 400 bytes = ~400 GB/day
10 years: 400 GB × 365 × 10 ≈ 1.5 PB
```

**Location data is a different story.** At roughly 50 bytes per ping, persisting every update would be about 860 GB/day. So the design **keeps only the latest position hot** (in memory) and stores history in a separate write-optimized store, often sampled or kept only for trips.

| Metric                                | Estimate        |
| ------------------------------------- | --------------- |
| DAU                                   | 100 million     |
| Active drivers                        | 1 million       |
| Rides/day                             | 10 million      |
| Request RPS                           | ~12,000         |
| Location update RPS                   | ~200,000 (peak) |
| Storage/10 years (transactional data) | ~1.5 PB         |

The request rate is modest; the **location write rate** is what drives the architecture.

## Data Model

### `customers` table

| Column                      | Type      |
| --------------------------- | --------- |
| `id`                        | UUID      |
| `name`                      | VARCHAR   |
| `email`                     | VARCHAR   |
| `phone`                     | VARCHAR   |
| `default_payment_method_id` | UUID      |
| `created_at`                | TIMESTAMP |

### `drivers` table

| Column                | Type                                    |
| --------------------- | --------------------------------------- |
| `id`                  | UUID                                    |
| `name`                | VARCHAR                                 |
| `license_number`      | VARCHAR                                 |
| `rating`              | FLOAT                                   |
| `is_available`        | BOOLEAN                                 |
| `last_known_location` | POINT (lat, lng) — updated infrequently |
| `created_at`          | TIMESTAMP                               |

> The **live** location is **not** written to this table on every ping. At 200,000 updates/second a relational database would collapse. Live positions live in an in-memory geospatial index; the database holds only a periodically refreshed last-known location.

### `cabs` table

| Column                | Type                                |
| --------------------- | ----------------------------------- |
| `id`                  | UUID                                |
| `driver_id`           | UUID                                |
| `registration_number` | VARCHAR                             |
| `type`                | ENUM (UberGo, UberX, UberXL, Black) |
| `capacity`            | INT                                 |

### `trips` table

| Column             | Type                                                          |
| ------------------ | ------------------------------------------------------------- |
| `id`               | UUID                                                          |
| `customer_id`      | UUID                                                          |
| `driver_id`        | UUID                                                          |
| `cab_id`           | UUID                                                          |
| `source_lat`       | FLOAT                                                         |
| `source_lng`       | FLOAT                                                         |
| `destination_lat`  | FLOAT                                                         |
| `destination_lng`  | FLOAT                                                         |
| `status`           | ENUM (requested, accepted, in_progress, completed, cancelled) |
| `fare`             | DECIMAL                                                       |
| `started_at`       | TIMESTAMP                                                     |
| `ended_at`         | TIMESTAMP                                                     |
| `distance_km`      | FLOAT                                                         |
| `surge_multiplier` | FLOAT                                                         |
| `request_key`      | VARCHAR, UNIQUE (client-supplied idempotency key)             |

### `ratings` table

| Column     | Type                      |
| ---------- | ------------------------- |
| `id`       | UUID                      |
| `trip_id`  | UUID                      |
| `rater_id` | UUID (customer or driver) |
| `ratee_id` | UUID                      |
| `rating`   | INT (1-5)                 |
| `feedback` | TEXT                      |

### `payments` table

| Column                    | Type                                                    |
| ------------------------- | ------------------------------------------------------- |
| `id`                      | UUID                                                    |
| `trip_id`                 | UUID                                                    |
| `amount`                  | DECIMAL (or integer in the smallest currency unit)      |
| `currency`                | VARCHAR                                                 |
| `status`                  | ENUM (pending, authorized, completed, failed, refunded) |
| `provider`                | ENUM (stripe, paypal, cash)                             |
| `provider_transaction_id` | VARCHAR                                                 |
| `idempotency_key`         | VARCHAR, UNIQUE                                         |

## API Design

```
// Request a ride (customer)
POST /rides
Headers: Idempotency-Key: {client-generated unique key}
Body: { source: {lat, lng}, destination: {lat, lng}, cabType, paymentMethod }
Response 202: { rideID, estimatedFare, estimatedETA }

// Cancel a ride (customer)
DELETE /rides/{rideID}
Body: { reason? }
Response 200: { success }

// Accept a ride (driver)
POST /rides/{rideID}/accept
Response 200: { customerLocation, customerName, estimatedPickupTime }
Response 409: ride already taken or cancelled

// Deny a ride (driver)
POST /rides/{rideID}/deny
Response 200: { success }

// Start a trip (driver)
POST /trips/{tripID}/start
Response 200: { tripID, startedAt }

// End a trip (driver)
POST /trips/{tripID}/end
Response 200: { tripID, fare, endedAt }

// Update driver location (driver, called every 5 seconds)
PUT /drivers/{driverID}/location
Body: { lat, lng, heading, speed }
Response 200: { success }

// Rate a trip
POST /trips/{tripID}/rating
Body: { rating: 1-5, feedback? }
Response 201: { success }
```

**Why the `Idempotency-Key`?** Mobile networks drop responses. If the app retries `POST /rides` after a timeout, the server must not create a second ride. It stores the key (the `request_key` column) and returns the original result on a repeat. Accept, start, and end calls are made safe by the **state machine** described below.

## Architecture

### Core Services

| Service                         | Responsibility                                                                           |
| ------------------------------- | ---------------------------------------------------------------------------------------- |
| **Customer Service**            | Customer authentication, profiles                                                        |
| **Driver Service**              | Driver authentication, profiles, availability                                            |
| **Ride Service**                | Ride requests, dispatch orchestration, ETA calculation                                   |
| **Matching / Location Service** | Ingests driver locations and answers "who is nearby?" from an in-memory geospatial index |
| **Trip Service**                | Trip lifecycle management (state machine)                                                |
| **Payment Service**             | Fare calculation, payment processing, refunds                                            |
| **Notification Service**        | Push notifications (ride requests, confirmations, alerts)                                |
| **Analytics Service**           | Surge pricing signals, business intelligence                                             |

Most services are **stateless** and scale by adding instances behind a [load balancer](system-design/load-balancing). The **Matching/Location** index is the exception: it holds in-memory state and is **sharded by geography** (explained below).

## The Core Problem: Real-Time Driver Matching

When a customer requests a ride, the system must **find the nearest available drivers** within seconds. This is a geospatial query over 1 million+ live driver locations that change every 5 seconds.

### Approach 1: SQL Geospatial Queries

```sql
SELECT * FROM drivers
WHERE is_available = TRUE
  AND ST_DWithin(
    last_known_location,
    ST_Point(-122.4016, 37.7564)::geography,
    5000  -- within 5km radius in meters
  )
ORDER BY ST_Distance(last_known_location, ST_Point(-122.4016, 37.7564)::geography)
LIMIT 10;
```

PostgreSQL with the PostGIS extension supports this. It works at small scale but becomes slow with millions of rows that are updated constantly.

### Approach 2: Geohashing

Each driver's GPS coordinates are encoded into a **geohash string** (a hierarchical alphanumeric code). Drivers with nearby locations share a common geohash prefix.

```
San Francisco (37.7564, -122.4016) → Geohash: 9q8yy9mf

Nearby drivers share prefix "9q8yy" — a longer shared prefix means a smaller area
```

**Implementation:**

- Keep a **set of driver IDs per geohash cell** in Redis (for example `cell:9q8yy → {driver1, driver2, ...}`)
- To find nearby drivers: look up the rider's cell **and its 8 neighboring cells**, then filter by exact distance
- When a driver moves into a new cell, remove them from the old cell's set and add them to the new one
- Redis also has built-in geospatial commands (`GEOADD`, `GEOSEARCH`), which use geohash internally and are a good fit for a simple version

**Limitation:** cells at the boundary can miss nearby drivers just across the line, which is why the 8 neighbors are searched too. Fixed-size cells are also wasteful: many drivers in a dense city cell, almost none in a rural one.

### Approach 3: Quadtree (Adaptive Cells)

A **Quadtree** recursively divides 2D geographic space into four quadrants. Each leaf node stores the drivers within that cell. A cell is **subdivided when it exceeds a driver-count threshold**.

![Quadtree](/images/quadtree.png)

**Properties:**

- Dense urban areas get many small cells (high precision)
- Rural areas get large cells (few drivers)
- Finding nearby drivers = walk down the tree to the rider's cell, collect drivers there and in adjacent cells

**In-memory:**

- Millions of driver positions at about 100 bytes each is only a few GB, which fits in memory
- Updates are cheap: when a driver crosses a cell boundary, move them between leaves
- Range queries cost O(log N + K), where K is the number of drivers returned

### What Real Systems Use

In practice, large ride-hailing companies use **hierarchical grid systems** such as **H3** (hexagonal cells, open-sourced by Uber) or **S2** (Google's spherical cells). Hexagons have equal distance to all neighbors, which makes "nearby" searches and demand/supply aggregation (surge) cleaner than squares. The idea is the same as geohash: convert location to a cell ID, then look up drivers by cell ID and neighboring cells.

### Making the Index Scale: Shard by Region

The index sees ~200,000 updates/second, so a single in-memory structure on one machine is not enough, and every stateless Ride Service instance cannot hold a full copy and consume every update.

- **Partition the map into regions or cell ranges**, and assign each partition to a Matching/Location shard.
- Each shard consumes only the location updates for its own area and answers queries for its own area.
- A query near a shard boundary asks the neighboring shard too.
- Replicate each shard (a standby that can take over) so a crash doesn't lose the live index; since positions refresh every 5 seconds, a rebuilt index becomes accurate again within seconds.

### From "Nearest" to "Best"

Straight-line distance is not the same as arrival time (rivers, one-way streets, traffic). Production matching takes the nearest 10–20 candidates and ranks them by **estimated time to pickup** from a routing/ETA service, plus driver rating and acceptance history. Large systems also match **batches of riders and drivers together** rather than greedily one request at a time, to reduce total waiting time.

## Real-Time Location Tracking

### Driver → Server (Location Updates)

Every 5 seconds, the driver's app sends a location update:

- **WebSocket or a persistent connection** (low overhead per message, bidirectional — also used to push ride offers)
- Alternatively, HTTP POST with a short interval (simpler but more overhead)

The Location Service receives the update and:

1. **Updates the driver's position in the in-memory geospatial index** (the hot path for matching), with a TTL so a driver who stops sending updates for ~60 seconds is treated as offline
2. **Publishes the update to Kafka** (partitioned by driver ID) for location history, analytics, and ETA models, consumed asynchronously
3. If the driver is on an active trip: **pushes the position to that trip's customer** (below)

It does **not** write each ping to the primary relational database.

### Server → Customer (Live Tracking)

Fan-out for live tracking is **per active trip**, so it should not rely on Kafka topics per driver (millions of tiny subscriptions don't work well there):

- When a trip is active, the Location Service publishes the driver's position to a **per-trip channel** (for example Redis Pub/Sub channel `trip:{tripID}`).
- The gateway server that holds the customer's WebSocket connection **subscribes to that channel** and forwards each update to the app.
- The customer app renders the driver's live position on the map.

```
Driver app → WebSocket → Location Service ─┬→ in-memory geo index (matching)
                                           ├→ Kafka (history, analytics)
                                           └→ Redis Pub/Sub "trip:{id}" → gateway → Customer WebSocket
```

See [Messaging and Communication](system-design/messaging-and-communication) for WebSocket scaling and [Design WhatsApp](system-design/whatsapp-design) for a similar connection-routing problem.

## Ride Dispatch Flow

![Uber Working](/images/uber-working.png)

1. **Customer** submits a ride request (source, destination, cab type) with an idempotency key.
2. **Ride Service** creates the trip in `requested` state and asks the Matching Service for the best candidate drivers (nearest by ETA, top 3).
3. For **Driver 1** (best match), Ride Service **reserves the driver** (see below), then the **Notification Service** sends the ride offer.
4. **Driver 1** has 15 seconds to accept or deny.
5. If accepted: the trip moves to `accepted`, the driver is marked unavailable, and the customer is notified with driver ETA and live location.
6. If denied or timeout: the reservation is released and the offer goes to Driver 2, then Driver 3.
7. If no driver accepts after 3 attempts: notify the customer (offer retry or widen the search).

**Timers must survive crashes.** "Wait 15 seconds, then try the next driver" should not be a timer inside one server's memory. Store the pending offer (with its deadline) in Redis or the database and use a delay queue or key-expiry so another instance can continue if the server dies.

### Race Condition Prevention

Multiple riders may request at the same time, and two Ride Service instances might both pick the same driver as the best match. We need **only one rider to get that driver**.

**Layer 1: a short-lived reservation in Redis**

```
SET driver_offer:{driverID} {rideID} NX EX 20
```

- `NX` means "set only if the key does not exist," and `EX 20` sets a 20-second expiry, all in **one atomic command**. If the command succeeds, this ride holds the driver; if it returns nil, someone else already does, so move to the next candidate.
- The TTL is slightly longer than the 15-second accept window, so a crashed dispatcher cannot hold the driver forever.
- (Older material writes `SETNX key value EX 30`; `SETNX` does not accept an expiry, so use `SET ... NX EX` instead.)

**Release safely.** Never delete the key blindly: if your lock already expired and another ride acquired it, you would delete _their_ lock. Delete only if the value is still yours, atomically with a Lua script:

```lua
if redis.call("GET", KEYS[1]) == ARGV[1] then
  return redis.call("DEL", KEYS[1])
else
  return 0
end
```

**Layer 2: the database decides.** A Redis lock is an **optimization, not the source of truth**. Locks can expire while a process is paused, and Redis can fail over and lose a lock. So the authoritative step is an atomic **conditional update** in the database:

```sql
-- Driver accepts: succeeds for exactly one transaction
UPDATE trips
SET status = 'accepted', driver_id = :driver
WHERE id = :trip AND status = 'requested';

UPDATE drivers
SET is_available = FALSE
WHERE id = :driver AND is_available = TRUE;
-- rows affected = 1 → we won; 0 → conflict, return 409 / try the next driver
```

Do both updates in a single transaction. This is **optimistic concurrency** (compare-and-set): even if two dispatchers slip through the Redis layer, only one update matches the `WHERE` condition.

## Trip State Machine

A trip moves through a fixed set of states, and **only valid transitions are allowed**:

```
requested ──► accepted ──► in_progress ──► completed
    │            │
    └────────────┴──► cancelled
```

- Every transition is a conditional update (`WHERE status = <expected previous state>`), so duplicate or out-of-order requests (a retried "end trip," a cancel arriving after accept) are rejected or ignored, never applied twice.
- Add a **unique constraint** so a driver or customer can have at most one active trip.
- The authoritative trip state lives in a **strongly consistent store** (a relational database, sharded by region) and not in an eventually consistent cache.

## Surge Pricing

Surge pricing dynamically increases fares when demand exceeds supply in a geographic area.

```
Surge Multiplier = f(demand_rate, supply_rate, historical_patterns)
```

**Implementation:**

1. **Analytics Service** continuously monitors ride request rate and available driver count **per cell** (geohash/H3)
2. When `demand / supply > threshold` in a cell, it computes a surge multiplier (e.g., 1.5x, 2x, 3x), usually **smoothed over time** so prices don't flicker
3. The multiplier is published to Redis with a TTL
4. Ride Service reads the multiplier when computing fare estimates
5. Customers see the surge indicator before confirming, and the multiplier shown is **stored on the trip** (`surge_multiplier`) so the price doesn't change after they accept

Surge also incentivizes drivers to move to high-demand areas (shown on the driver's heatmap).

## Payment Processing

Rather than building payment infrastructure from scratch:

1. Integrate with a third-party processor: **Stripe**, **Braintree**, or **PayPal**
2. The customer's payment method is **tokenized** and stored with the processor (never stored raw in Uber's database)
3. Optionally **pre-authorize** (place a hold for the estimated fare) when the ride is requested, to catch failing cards early
4. After trip completion, Ride Service calculates the final fare (surge, tolls, etc.)
5. Payment Service **captures/charges** the stored payment method via the processor's API
6. A **webhook** from the processor confirms success or failure
7. The payment record is stored in the `payments` table

### Never Charge Twice: Idempotency

The most dangerous bug here is a **retry that creates a second charge**. A request can time out _after_ the processor already charged the card; a naive retry would charge again.

- Generate a **deterministic idempotency key** per charge attempt (for example derived from the trip ID, like `trip-{id}-charge`).
- Send it to the processor (Stripe-style APIs accept an `Idempotency-Key` header): repeating the call with the same key returns the original result instead of charging again.
- Store the key in the `payments` table with a **UNIQUE** constraint, so your own system can't create two payment rows for the same charge.
- Create the payment row in `pending` **before** calling the processor, then update it from the result, so a crash in between can be reconciled.

### Retries and Webhooks

- **Retry** timeouts and server errors with exponential backoff and jitter, **always reusing the same idempotency key**. Do **not** blindly retry hard declines (insufficient funds); move to a failed state instead.
- **Webhooks** can arrive **more than once and out of order**. Verify the signature, **deduplicate by event ID**, and make the handler idempotent (ignore an update that would move the payment backward).
- Run a periodic **reconciliation job** that compares your records with the processor's to catch anything lost.
- A persistently failed payment becomes an **outstanding balance** on the customer's account, collected on the next ride or by blocking new rides until settled.
- Process payment work **asynchronously** from a queue (with a dead-letter queue for manual review) so a slow processor never blocks trip completion. See [Event-Driven Architecture](system-design/event-driven) for the outbox pattern that keeps "trip completed" and "charge requested" consistent.

## Notifications

```
Ride request → Notification Service → driver's open WebSocket (if connected)
                                    → FCM (Android) / APNs (iOS) as fallback or wake-up
```

Notification payload includes:

- Pickup location
- Customer name and rating
- Estimated fare
- Cab type requested
- Time limit to accept (15 seconds)

For time-critical notifications (ride requests to drivers), use **high-priority** FCM messages, which can wake a device from Doze mode. Send through a queue with retries so a slow push provider doesn't block dispatch.

## Detailed Design

### Data Partitioning

**Geographic sharding** is a natural fit:

- Partition by **geographic region** (city or metro area), using a **region → shard directory**
- Most data for a trip (drivers, trips, customers in that region) is co-located, and cross-region queries are rare
- Trips that cross a region boundary, and riders who travel, are handled by keeping a global user/identity store while trip data lives in the region where it starts
- **Within** a region, consistent hashing spreads cache keys and matching shards evenly. See [Databases](system-design/databases) for sharding trade-offs.

### Caching Strategy

| Data                      | Store                                                     | Policy                                                                               |
| ------------------------- | --------------------------------------------------------- | ------------------------------------------------------------------------------------ |
| Driver locations (active) | In-memory geo index / Redis GEO                           | Updated every 5s; expires after ~60s without an update                               |
| Active trip state         | **Database is authoritative**; Redis is only a read cache | Short TTL; invalidate on every state change. Don't use LRU eviction as the only copy |
| Surge multipliers         | Redis                                                     | TTL 60s; refreshed by Analytics Service                                              |
| User profiles             | Redis                                                     | LRU; TTL 1 hour                                                                      |

See [Caching and CDN](system-design/caching-and-cdn) for patterns and invalidation.

### Analytics

Driver GPS data → Kafka → Apache Spark:

- **Route optimization:** identify high-traffic corridors
- **Demand prediction:** ML models predict demand by area and time of day
- **Driver supply analysis:** identify supply shortfalls before they occur

## Advanced Architecture

![Uber Advanced Design](/images/uber-advanced-design.png)

### Bottleneck Resolutions

| Bottleneck                                       | Solution                                                                                      |
| ------------------------------------------------ | --------------------------------------------------------------------------------------------- |
| Driver matching latency                          | In-memory geospatial index (cells/quadtree), sharded by region                                |
| Location update throughput (~200k/s)             | In-memory index for hot data; Kafka for the history pipeline; **no per-ping database writes** |
| Race conditions on driver dispatch               | Redis `SET NX EX` reservation **plus** database conditional update                            |
| Duplicate rides on retry                         | Idempotency key on `POST /rides`                                                              |
| Double charges                                   | Deterministic idempotency key, UNIQUE constraint, webhook deduplication                       |
| Ride Service availability                        | Stateless instances behind a load balancer; matching shards replicated                        |
| Payment failures                                 | Retry queue with backoff + DLQ for manual review                                              |
| Notification delivery                            | Queue-backed pipeline with retries and WebSocket-first delivery                               |
| Cascading failures (e.g., payment provider slow) | Timeouts and [circuit breakers](system-design/circuit-breaker); degrade non-critical features |
| Overload / abuse                                 | [Rate limiting](system-design/rate-limiting) at the edge                                      |

## Interview Questions

**Q: How do you prevent two riders from getting the same driver?**
Reserve the driver atomically in Redis with `SET key value NX EX ttl`, and enforce correctness with a database conditional update (`WHERE is_available = TRUE`) in the same transaction as assigning the trip. The Redis lock is an optimization; the database is the authority.

**Q: How do you avoid charging a customer twice if a payment request times out?**
Use an idempotency key (derived from the trip) on every charge request, reuse the same key on retries, enforce uniqueness in your own payments table, and deduplicate webhooks by event ID.

**Q: Why not store every location update in the database?**
About 200,000 updates per second would overwhelm a relational database, and only the latest position matters for matching. Keep the live position in memory, stream history through Kafka to a write-optimized store.

**Q: Geohash, quadtree, or H3?**
All convert location into cells for fast lookup. Geohash uses fixed rectangular cells, a quadtree adapts cell size to density, and H3/S2 give uniform hierarchical cells that make neighbor searches and aggregation cleaner. Search neighboring cells in all cases.

**Q: How do you scale the matching service?**
Shard the geospatial index by region so each shard handles a slice of the updates and queries, replicate shards for failover, and rank candidates by ETA rather than straight-line distance.

**Q: How do you stream a driver's location to the rider?**
Push each update to a per-trip channel (such as Redis Pub/Sub); the gateway server holding the rider's WebSocket subscribes to that channel and forwards updates. Kafka handles the analytical pipeline, not per-user fan-out.

**Q: What happens if the dispatcher crashes while waiting for a driver to accept?**
The reservation expires through its TTL, and the pending offer and its deadline are stored outside the process, so another instance can resume or move on to the next driver.

## Summary

| Component                | Technology                                                                                       |
| ------------------------ | ------------------------------------------------------------------------------------------------ |
| Location tracking        | WebSockets (bidirectional); per-trip Redis Pub/Sub for live fan-out                              |
| Geospatial matching      | In-memory index (geohash cells / Quadtree / H3), sharded by region                               |
| Location event stream    | Apache Kafka (history and analytics)                                                             |
| Dispatch safety          | Redis `SET NX EX` reservation + database compare-and-set                                         |
| Primary database         | PostgreSQL sharded by region (trips, users, payments); Cassandra or similar for location history |
| Cache                    | Redis (surge, profiles, trip read cache)                                                         |
| Payment processing       | Stripe / Braintree with idempotency keys and webhook deduplication                               |
| Notifications            | WebSocket first, FCM + APNs fallback, queue-backed                                               |
| Analytics                | Kafka + Apache Spark                                                                             |
| Resilience               | Circuit breakers, rate limiting, retries with backoff and jitter                                 |
| Service discovery / mesh | Consul or Kubernetes DNS; Envoy-based mesh (e.g., Istio)                                         |

## Continue Learning

- [Caching and CDN](system-design/caching-and-cdn) — caching surge multipliers and trip state safely.
- [Databases](system-design/databases) — sharding by region and conditional updates.
- [Messaging and Communication](system-design/messaging-and-communication) — Kafka, idempotency, and WebSocket scaling.
- [Event-Driven Architecture](system-design/event-driven) — the outbox pattern for trip and payment events.
- [Circuit Breaker Pattern](system-design/circuit-breaker) — protecting dispatch from a slow payment provider.
