---
title: "System Design Case Study — Ticket Booking System (BookMyShow / Ticketmaster)"
description: "A complete system design for a ticket booking platform: preventing double booking, seat holds with expiry, atomic multi-seat reservation, payment handling, flash-sale traffic with a virtual waiting room, search, caching, and failure handling."
author: ["name": "Rajendra Pancholi", "email": "rpancholi522@gmail.com"]
created: "2026-10-17"
updated: "2026-10-17"
thumbnail: "/images/system-design-arch.png"
tags: [system-design, ticket-booking, concurrency, seat-reservation, flash-sale, case-study]
keywords: ["Design BookMyShow", "Ticket booking system design", "Prevent double booking", "Seat hold with TTL", "Flash sale system design", "Virtual waiting room", "System design interview booking"]
---

# System Design Case Study — Ticket Booking System (BookMyShow / Ticketmaster)

A ticket booking system sells a **limited inventory** (seats) to a **huge number of simultaneous buyers**. Browsing is easy to scale, but the heart of the problem is correctness under contention: **two people must never get the same seat**, while thousands compete for the same few seats the moment a popular show goes on sale.

The same design patterns apply to **hotel and flight booking, appointment slots, and e-commerce flash sales** (selling limited stock), so this is a very reusable case study.

![Ticket Booking](/images/system-design-arch.png)

---

## Requirements

### Functional Requirements
- Browse and **search** events, movies, and shows by city, date, genre, and venue
- View a show's **seat map** with live availability
- **Select seats** and temporarily **hold** them while paying
- **Pay** and receive a confirmed booking with a ticket (QR code)
- **Cancel** a booking and get a refund according to policy
- Handle **general admission** events (no seat numbers, just a count)

### Non-Functional Requirements
- **No double booking** (the hard, non-negotiable requirement): strongly consistent seat state
- **High availability** for browsing, and **low latency** for seat selection
- **Scales for spikes:** popular releases and sales create sudden, enormous demand
- **Fairness** during high demand, and **resistance to bots**
- **Durable and auditable:** payments and bookings are never lost

---

## Capacity Estimation

Assumptions: **20 million DAU** browsing; **1 million bookings/day**; a blockbuster on-sale with **2 million users** arriving in the first minute for **100,000 seats**.

```
Normal browsing: 20M users × 20 requests = 400M/day ≈ 4,600 requests/second
Bookings: 1M/day ≈ 12/second on average (tiny)

Flash sale spike:
2M users / 60 seconds ≈ 33,000 requests/second on one show's seat availability
Only 100,000 seats to sell → at most 100,000 successful bookings, 
so ~95% of arrivals will be told "sold out" or must wait

Seat data: 50,000 shows/day × 200 seats ≈ 10M seat rows/day × ~100 bytes ≈ 1 GB/day
```

| Metric | Estimate |
|---|---|
| Browsing traffic | ~4,600 RPS average |
| Bookings | ~12/s average |
| Flash-sale spike | ~33,000 RPS concentrated on one show |
| Seat data | ~1 GB/day (small) |

Two takeaways: **storage is trivial**; the difficulty is **contention on a tiny set of rows** (one show's seats), where most of the load is people who will not get a seat. The design should protect the booking path from the crowd.

---

## Core Concepts

### Three Seat States

```
AVAILABLE ──(hold)──► HELD ──(payment success)──► BOOKED
                       │
                       └──(timeout / payment fails / user leaves)──► AVAILABLE
```

A **hold** reserves the seat for a short time (for example 5–10 minutes) while the user pays. It prevents someone else from taking the seat mid-payment, without permanently blocking it if the user disappears.

### Browsing Is Eventually Consistent; Booking Is Strongly Consistent

- The **seat map** a user sees can be a few seconds stale (cached). That is acceptable.
- The **hold and confirm steps are authoritative** and strongly consistent. If a seat turns out to be taken, the user is told "seat just taken, pick another."

This split lets the read side scale massively with caches while the write side stays correct.

---

## Data Model

### `shows` table
| Column | Type |
|---|---|
| `show_id` | UUID |
| `event_id`, `venue_id`, `screen_id` | UUID |
| `starts_at` | TIMESTAMP |
| `status` | ENUM (scheduled, on_sale, sold_out, cancelled) |

### `seats` table (one row per seat per show)
| Column | Type |
|---|---|
| `show_id` | UUID |
| `seat_id` | VARCHAR (for example "C12") |
| `price` | DECIMAL |
| `status` | ENUM (available, held, booked) |
| `held_by` | UUID (booking ID) |
| `hold_expires_at` | TIMESTAMP |

Primary key: `(show_id, seat_id)`.

### `bookings` table
| Column | Type |
|---|---|
| `booking_id` | UUID |
| `user_id` | UUID |
| `show_id` | UUID |
| `status` | ENUM (pending, confirmed, cancelled, expired) |
| `total_amount` | DECIMAL |
| `idempotency_key` | VARCHAR, UNIQUE |
| `created_at` | TIMESTAMP |

### `booking_seats` table
`booking_id`, `show_id`, `seat_id`.

### `payments` table
`payment_id`, `booking_id`, `amount`, `status`, `provider_transaction_id`, `idempotency_key` (UNIQUE).

### Database Choice

Use a **relational database** (PostgreSQL or MySQL) for seats, bookings, and payments: you need **transactions and conditional updates**. Shard by **show ID** so all seats of a show live together (a booking touches only one show). Use read replicas, caches, and a search engine for the catalog.

---

## API Design

```
// Search shows
GET /shows?city=Mumbai&date=2026-11-01&genre=movie&cursor=...
Response: [{ showId, title, venue, startsAt, minPrice }]

// Seat map (cacheable, slightly stale)
GET /shows/{showId}/seats
Response: [{ seatId, price, status }]

// Hold seats (authoritative, atomic, all-or-nothing)
POST /shows/{showId}/holds
Headers: Idempotency-Key: {key}
Body: { seatIds: ["C12","C13"] }
Response 201: { bookingId, expiresAt, total }
Response 409: { unavailableSeats: ["C13"] }

// Pay and confirm
POST /bookings/{bookingId}/pay
Headers: Idempotency-Key: {key}
Body: { paymentMethod }
Response 202: { status: "processing" }

// Get booking / ticket
GET /bookings/{bookingId}
Response: { status, seats, qrCode }

// Cancel
DELETE /bookings/{bookingId}
```

---

## The Core Problem: Holding Seats Without Double Booking

### Naive Approach (Broken)

```text
1. Read the seat → "available"
2. (another user does the same)
3. Write "held by me"
```

Two requests can both read "available" and both write, so **both think they got the seat**. This is the classic race condition (read-then-write without atomicity).

### Solution: An Atomic Conditional Update

Let the database decide in **one atomic statement** with a condition in the `WHERE` clause:

```sql
-- Hold seats: all-or-nothing inside one transaction
BEGIN;

UPDATE seats
SET status = 'held',
    held_by = :bookingId,
    hold_expires_at = now() + interval '10 minutes'
WHERE show_id = :showId
  AND seat_id IN ('C12', 'C13')
  AND (status = 'available'
       OR (status = 'held' AND hold_expires_at < now()));   -- expired holds are reusable

-- rows affected must equal the number of seats requested
-- if fewer → ROLLBACK and return 409 (someone else got at least one seat)
COMMIT;
```

- Only **one** transaction can change a given seat from available to held, because the update checks the current state at the moment of writing. The loser's update matches zero rows.
- **All-or-nothing:** if the user asked for 2 seats and only 1 was free, the transaction rolls back so they don't end up with a partial, useless hold.
- **Expired holds are treated as available** directly in the condition, so correctness doesn't depend on a cleanup job running on time.
- **Avoid deadlocks** when two users request overlapping seats in different orders: always **sort seat IDs** and lock/update them in the same order.

### Pessimistic vs Optimistic Locking

| Approach | How | Use when |
|---|---|---|
| **Atomic conditional update** (above) | `UPDATE ... WHERE status = 'available'` | Default choice: simple, no long locks |
| **Pessimistic lock** | `SELECT ... FOR UPDATE` on the seat rows, then update | Complex multi-step logic inside one transaction |
| **Optimistic lock (version column)** | Update `WHERE version = :seen`, retry on conflict | Low contention (not the case for hot seats) |

For hot shows, contention is **high**, so prefer short atomic updates and keep transactions tiny.

### Confirming After Payment

```sql
BEGIN;
UPDATE seats
SET status = 'booked'
WHERE show_id = :showId AND seat_id IN (...)
  AND held_by = :bookingId
  AND hold_expires_at > now();          -- the hold must still be valid

-- rows affected = number of seats → mark booking confirmed, create ticket
COMMIT;
```

If the hold has expired and someone else took the seat, the confirm fails and the payment must be **refunded automatically**.

---

## The Payment Window Problem

What if the user's payment succeeds at the same moment the hold expires?

**Best approach: don't let the hold expire during an active payment.**

1. When the user presses Pay, move the booking to `payment_in_progress` and **extend the hold** (to a bounded maximum, for example another few minutes).
2. A background job releases only holds that are expired **and** not in an active payment.
3. If the payment succeeds after all, confirm normally. If the confirm fails (the seat was lost), **refund automatically** and tell the user.
4. Payment **webhooks may arrive late or twice**: process them idempotently by booking ID and provider transaction ID, and reconcile with the provider regularly. The same idempotency rules as in [Design Uber](system-design/uber-design) (payment section) apply.

---

## Expiring Holds

Two complementary mechanisms:

- **Lazy expiry:** treat holds with `hold_expires_at < now()` as available during the hold query (as above). This is always correct.
- **Background cleanup:** a periodic job (or a delayed message when the hold is created) marks expired bookings as `expired` and sets their seats back to `available`, so seat maps and counts stay accurate, and users are notified.

Optionally keep **holds in Redis** (`SET hold:{show}:{seat} {bookingId} NX EX 600`) for very high throughput, with the database as the final authority at confirmation. Be careful: a Redis-only lock can be lost on failover, so the database conditional update must still decide. See [Distributed Systems Essentials](system-design/distributed-systems-essentials).

---

## General Admission and Flash Sales (Counters)

When there are no seat numbers (concert standing area, or **e-commerce stock**), inventory is a **counter**:

```sql
UPDATE inventory
SET available = available - :qty
WHERE show_id = :showId AND available >= :qty;
-- rows affected = 1 → reserved;  0 → sold out
```

The condition makes the decrement atomic and prevents overselling (the count never goes below zero). For **extreme throughput** on one counter, options include splitting stock across several counters (shards) and drawing from them, or using a fast in-memory store with periodic reconciliation to the database. Reserve with a **hold and expiry** just as for seats.

---

## Surviving the Flash Sale: Protect the Booking Path

When 2 million users hit a 100,000-seat sale, most will lose. Without protection, they overwhelm the seat database for everyone. Layer defenses:

### 1. Virtual Waiting Room (Queue)

Before users reach seat selection, put them in an **online queue**:

- Users get a **queue position** and are admitted at a **controlled rate** that the booking service can handle (for example 2,000 buyers at a time).
- Admission uses a **signed token** (with expiry) that the booking API requires.
- Fair ordering (first come, first served, or a randomized lottery for the arrival window to avoid rewarding the fastest bots).
- The queue itself must be cheap and scalable: serve it from the **CDN/edge or a separate lightweight service**, backed by a counter or a sorted set, never from the seat database.

### 2. Cache and Offload Reads

- The **seat map** is served from a cache (Redis) with a **short TTL** (a couple of seconds) or built from a stream of seat changes; push updates to viewers over SSE or WebSocket for live availability. See [Caching and CDN](system-design/caching-and-cdn).
- Static catalog data (event details, posters) comes from the **CDN**.

### 3. Rate Limiting and Bot Defense

- Rate limit per user, IP, and device, and cap seats per booking (for example 6).
- Use CAPTCHAs or proof-of-work challenges, device fingerprinting, and account-level checks to curb scalping bots. See [Design a Rate Limiter](system-design/rate-limiter-design).

### 4. Isolate Hot Shows

- Run the on-sale for a blockbuster on **dedicated capacity** (its own database shard and service instances) so it can't starve other shows.
- Keep the booking transaction **tiny and fast**.

### 5. Fail Gracefully

- Return "sold out" quickly when inventory hits zero (a cached flag), without touching the database.
- Show clear status and retry guidance instead of timeouts.

---

## Search and Catalog

- The catalog is **read-heavy and rarely changes**. Store it in the database, index it in **Elasticsearch/OpenSearch** for search (city, date, genre, title), and cache heavily.
- Availability filters ("shows with seats left") can use a **slightly stale** count, with the exact check at hold time.

---

## Architecture

```
Client ─► CDN ─► Load Balancer ─► API Gateway (auth, rate limit)
                                       │
        ┌──────────────┬───────────────┼────────────────┬──────────────┐
        ▼              ▼               ▼                ▼              ▼
   Catalog/Search   Seat-map      Waiting-room      Booking       Payment
   (ES + cache)     (Redis cache  service           Service       Service ─► Payment provider
                     + live push)                   (holds,           │
                                                     confirm)         ▼ webhooks
                                                       │
                                                       ▼
                                     Seat/Booking DB (sharded by show, replicated)
                                                       │
                                            Kafka (events via outbox)
                                       ┌───────────────┼───────────────┐
                                       ▼               ▼               ▼
                                 E-ticket/QR      Notifications    Analytics
```

- **Outbox pattern:** when a booking is confirmed, write the event to an outbox table in the same transaction, and publish it to Kafka, so tickets and notifications are never lost or duplicated (see [Event-Driven Architecture](system-design/event-driven)). Notifications use the [notification system](system-design/notification-system-design).
- **Ticket validation:** the QR code contains a **signed ticket ID**; at the venue, scanners verify the signature and mark the ticket used (atomic update) so a screenshot can't be reused twice.

---

## Cancellations and Refunds

- Cancellation moves the booking to `cancelled`, **releases the seats** (`booked → available`), and triggers a refund through the payment provider (idempotent, with reconciliation).
- Apply the cancellation policy (time windows and fees), and send notifications.
- Use a **state machine** for bookings with valid transitions only (`pending → confirmed → cancelled`), enforced by conditional updates, so duplicate or out-of-order requests are harmless.

---

## Failure Handling

| Failure | Handling |
|---|---|
| App server crashes after hold | Hold expires automatically; seat becomes available again |
| User closes the page mid-booking | Hold expires; background cleanup |
| Duplicate "pay" or "hold" requests | Idempotency keys and unique constraints |
| Payment succeeds but the confirm fails | Automatic refund and user notification |
| Payment webhook lost | Reconciliation job queries the provider for pending payments |
| Database primary fails | Replica promotion; in-flight requests retry safely thanks to idempotency |
| Cache outage | Fall back to the database for authoritative operations (rate limited) |

---

## Bottleneck Resolutions

| Bottleneck | Solution |
|---|---|
| Hot rows (one show's seats) | Short atomic updates, shard by show, dedicated capacity for blockbusters |
| 2M users for 100k seats | Virtual waiting room with controlled admission |
| Read load on seat map | Short-TTL cache, CDN, live push instead of polling |
| Double booking | Atomic conditional updates (never read-then-write) |
| Hold/payment race | Extend hold during payment, auto-refund on lost seat |
| Bots and scalpers | Rate limits, CAPTCHA, per-account seat caps |
| Lost or duplicated events | Outbox pattern and idempotent consumers |
| Stale "available" counts | Accept staleness for display, authoritative at hold time |

---

## Interview Questions

**Q: How do you prevent two users from booking the same seat?**
Use an atomic conditional update (`UPDATE ... WHERE status = 'available'`) inside a transaction so only one request can change the seat state, and treat zero affected rows as "seat taken."

**Q: Why hold seats instead of booking immediately?**
Payment takes time; a hold reserves the seat for a short window so others can't take it mid-payment, and it expires automatically if the user leaves.

**Q: How do you handle a user selecting several seats?**
Update them all in one transaction and verify the affected-row count equals the number requested, otherwise roll back. Sort seat IDs to avoid deadlocks.

**Q: What if payment succeeds after the hold expired?**
Prevent it by extending the hold while a payment is in progress. If the seat is still lost, confirm fails and the payment is refunded automatically.

**Q: How do you handle millions of users hitting a sale at once?**
A virtual waiting room admits users at a controlled rate, reads are served from caches and the CDN, rate limiting and bot protection filter abuse, and hot shows get isolated capacity.

**Q: How would you design for general admission or flash-sale stock?**
Use an atomic decrement with a condition (`available >= qty`) so stock can't go negative, with holds and expiry, and shard the counter if a single row is too hot.

**Q: Where do you use eventual consistency, and where strong consistency?**
Browsing, search, and seat-map display can be slightly stale and cached. Seat holds, confirmations, and payments must be strongly consistent.

**Q: How do you ensure confirmation emails and tickets are never lost?**
Write the event to an outbox table in the same transaction as the booking, publish it from there, and make consumers idempotent.

---

## Summary

| Component | Choice |
|---|---|
| Seat state | AVAILABLE → HELD (with expiry) → BOOKED |
| Preventing double booking | Atomic conditional updates in a transaction, never read-then-write |
| Multi-seat holds | All-or-nothing, with sorted seat order |
| Hold expiry | Lazy expiry in queries plus background cleanup |
| Payment | Idempotency keys, extended hold during payment, auto-refund, reconciliation |
| Flash sale | Virtual waiting room, caches, rate limits, isolated capacity |
| Database | Relational, sharded by show ID, with replicas for the catalog |
| Catalog and search | Elasticsearch, cache, CDN |
| Events | Outbox pattern to Kafka for tickets and notifications |

## Continue Learning

- [Databases](system-design/databases) — transactions, isolation, and conditional updates.
- [Distributed Systems Essentials](system-design/distributed-systems-essentials) — locks, leases, and fencing.
- [Design Uber](system-design/uber-design) — dispatch locking and idempotent payments.
- [Design a Rate Limiter](system-design/rate-limiter-design) — protecting the booking API from bots and spikes.
- [Event-Driven Architecture](system-design/event-driven) — the outbox pattern for reliable events.
