---
title: "System Design Case Study — Notification System (Push, SMS, Email at Scale)"
description: "A complete system design for a scalable notification service: push, SMS, email and in-app channels, priority queues, deduplication, retries and provider failover, user preferences, templates, bulk campaigns, delivery tracking, and rate limiting."
author: ["name": "Rajendra Pancholi", "email": "rpancholi522@gmail.com"]
created: "2026-10-17"
updated: "2026-10-17"
thumbnail: "/images/message-broker.png"
tags: [system-design, notification, push-notifications, kafka, queues, case-study]
keywords: ["Design a notification system", "Push notification architecture", "Notification service system design", "FCM APNs SMS email", "Notification deduplication and retries", "System design interview notification"]
---

# System Design Case Study — Notification System (Push, SMS, Email at Scale)

Almost every product sends notifications: a one-time password, an order update, a friend request, a marketing campaign. A notification system takes **events or requests from many services** and delivers them to users over **several channels** (mobile push, SMS, email, in-app), reliably, once, respecting each user's preferences.

The hard parts are not sending a message; they are **reliability** (never lose an OTP), **fairness** (a marketing blast must not delay a password reset), **duplicates**, **third-party provider limits and failures**, and **user trust** (no spam, easy opt-out).

![Notification System](/images/message-broker.png)

---

## Requirements

### Functional Requirements
- Send notifications over **push** (iOS and Android), **SMS**, **email**, and **in-app**
- Triggered by **events** from other services (order shipped, payment failed) and by **API requests** (OTP)
- **Scheduled** and **bulk campaign** notifications to large audiences
- **User preferences:** opt in or out per channel and per category, quiet hours, time zone
- **Templates** with variables and localization
- **Delivery tracking:** sent, delivered, opened, clicked, failed, bounced
- **Priority levels:** critical (OTP, security alert) vs normal vs low (marketing)

### Non-Functional Requirements
- **Reliable:** important notifications are never silently lost (at-least-once, with deduplication)
- **Timely:** OTPs within seconds; marketing can take minutes
- **Scalable:** handle large campaigns and daily peaks
- **Highly available:** the system must not go down when a provider does
- **Extensible:** new channels and providers can be added easily
- **Compliant:** honor unsubscribes and regulations

---

## Capacity Estimation

Assumptions: **50 million users**, an average of **4 notifications per user per day**.

```
Total/day: 50M × 4 = 200M notifications
Average rate: 200M / 86,400 ≈ 2,300/second

Campaign burst: 20M recipients sent within 30 minutes
20M / 1,800 s ≈ 11,000/second  (design the peak at ~25,000/second)

Channel mix (assumption): 70% push, 20% email, 10% SMS
→ SMS: ~20M/day (a major cost item)

Logs: 200M × ~300 bytes ≈ 60 GB/day → ~1.8 TB for 30 days
```

| Metric | Estimate |
|---|---|
| Notifications/day | ~200 million |
| Average rate | ~2,300/s |
| Peak (campaigns) | ~11,000–25,000/s |
| Delivery log | ~60 GB/day |

Two takeaways: the **peak is driven by campaigns**, not steady traffic, so queues must absorb bursts; and **SMS cost** matters as much as scale.

---

## How Notifications Are Delivered

The system does not send messages itself over the internet. It hands them to **third-party providers**:

| Channel | Providers | Notes |
|---|---|---|
| **iOS push** | Apple Push Notification service (APNs) | Needs a device token per app install |
| **Android push** | Firebase Cloud Messaging (FCM) | Device registration token |
| **SMS** | Twilio, Vonage, regional aggregators | Costly; regulated (sender IDs and templates must be registered in some countries, for example DLT in India) |
| **Email** | Amazon SES, SendGrid, Mailgun | Needs sender reputation, SPF/DKIM/DMARC |
| **In-app** | Your own service | Stored per user and pushed live over WebSocket or SSE |

Each provider has **rate limits, failures, and delivery semantics** of its own, and managing them is a core part of the design. Push delivery is **best effort**: the provider may drop or delay it if the device is offline.

---

## Data Model

### `devices` table
| Column | Type |
|---|---|
| `device_id` | UUID |
| `user_id` | UUID |
| `platform` | ENUM (ios, android, web) |
| `push_token` | VARCHAR |
| `last_seen_at` | TIMESTAMP |
| `is_active` | BOOLEAN |

A user can have **several devices**. Tokens change and expire, so keep them updated and remove invalid ones.

### `preferences` table
| Column | Type |
|---|---|
| `user_id` | UUID |
| `category` | VARCHAR (marketing, order_updates, security, ...) |
| `channel` | ENUM (push, sms, email, in_app) |
| `enabled` | BOOLEAN |
| `quiet_hours` | JSON (start, end, timezone) |

Security and transactional notifications (OTP, receipts) are usually **exempt** from marketing opt-outs, within legal rules.

### `templates` table
| Column | Type |
|---|---|
| `template_id` | VARCHAR |
| `version` | INT |
| `channel` | ENUM |
| `locale` | VARCHAR |
| `subject`, `body` | TEXT (with placeholders like `{{order_id}}`) |

### `notifications` table (the log)
| Column | Type |
|---|---|
| `id` | UUID |
| `idempotency_key` | VARCHAR, **UNIQUE** |
| `user_id` | UUID |
| `category`, `priority` | VARCHAR, ENUM |
| `channel` | ENUM |
| `status` | ENUM (queued, sent, delivered, failed, bounced, suppressed) |
| `provider`, `provider_message_id` | VARCHAR |
| `attempts` | INT |
| `created_at`, `sent_at`, `delivered_at` | TIMESTAMP |

Use a relational database for preferences and templates (small, consistent) and a **wide-column or time-series store** (such as Cassandra) for the high-volume delivery log. See [Databases](system-design/databases).

---

## API Design

```
// Send a notification (called by internal services)
POST /notifications
Headers: Idempotency-Key: {unique key}
Body: { userId, category, priority, templateId, params: {...}, channels?: [...], sendAt? }
Response 202: { notificationId }          // accepted, not yet delivered

// Create a bulk campaign
POST /campaigns
Body: { name, templateId, audience: { segmentId }, schedule, channels, rateLimitPerSecond }
Response 201: { campaignId }

// Register / update a device token (called by the mobile app)
PUT /users/{userId}/devices/{deviceId}
Body: { platform, pushToken }

// Preferences
GET  /users/{userId}/preferences
PUT  /users/{userId}/preferences

// Delivery status
GET /notifications/{notificationId}
Response: { status, attempts: [...] }

// Provider callbacks (webhooks)
POST /webhooks/{provider}
```

The API returns **202 Accepted** because sending is **asynchronous**: the request is validated, saved, and queued, and the caller doesn't wait for the provider.

---

## High-Level Design

```
Services (orders, payments, ...)        Campaign Scheduler
          │  events / API                     │
          ▼                                   ▼
   ┌──────────────────────────────────────────────┐
   │ Notification API / Kafka (events in)         │
   └──────────────────────┬───────────────────────┘
                          ▼
   ┌──────────────────────────────────────────────┐
   │ Notification Service (stateless workers)     │
   │  1. validate + dedupe (idempotency key)      │
   │  2. check user preferences + frequency caps  │
   │  3. render template (locale, params)         │
   │  4. choose channel(s) and priority           │
   └───────┬─────────────┬─────────────┬──────────┘
           ▼             ▼             ▼
     Push queues    SMS queues    Email queues      (separate queues per priority)
           ▼             ▼             ▼
     Push workers   SMS workers   Email workers  ──► rate limit + circuit breaker
           ▼             ▼             ▼
      APNs / FCM    SMS providers   Email providers
           │             │             │
           └──── delivery callbacks (webhooks) ───► Status service ──► Log / analytics
```

### Components

| Component | Responsibility |
|---|---|
| **Notification API** | Accepts requests, authenticates callers, validates, assigns an ID, returns 202 |
| **Event bus (Kafka)** | Buffers incoming requests and decouples producers from delivery (see [Messaging and Communication](system-design/messaging-and-communication)) |
| **Notification Service** | Deduplicates, applies preferences and caps, renders templates, routes to channels |
| **Per-channel, per-priority queues** | Absorb bursts and isolate channels and priorities from each other |
| **Channel workers** | Call the providers, handle retries and provider-specific rules |
| **Scheduler** | Releases scheduled and delayed notifications at the right time |
| **Status service** | Receives provider webhooks and updates delivery status |
| **Preference, template, and device services** | Hold user settings, templates, and push tokens (cached heavily) |

---

## Priorities: A Blast Must Not Delay an OTP

If every notification shares one queue, a 20-million-message campaign makes a password-reset code wait behind millions of marketing messages.

**Solution: separate queues and worker pools by priority and channel.**

| Priority | Examples | Treatment |
|---|---|---|
| **Critical** | OTP, security alerts | Own queue and dedicated workers, never throttled by campaigns, fast retries, provider failover |
| **Normal / transactional** | Order updates, receipts | Separate queue, high throughput |
| **Low / bulk** | Marketing, digests | Separate queue, **rate-limited** and paused automatically if critical traffic or provider limits need capacity |

Give each priority its own consumer group so one cannot starve another.

---

## Reliability: Deduplication, Retries, and Failures

### Preventing Duplicates

Messages can be delivered **more than once** (queue redelivery, caller retries). Sending the same OTP or "payment received" twice is a bad experience.

- Every request carries an **idempotency key** (for example `orderId + eventType + userId + channel`).
- The service checks it with an **atomic insert** into the log (unique constraint) or `SET key NX EX` in Redis before sending. A duplicate is dropped.
- This gives **at-least-once delivery with deduplication**, which behaves like exactly-once for the user. See the delivery guarantees in [Messaging and Communication](system-design/messaging-and-communication).
- Note that dedup can't be perfect across a provider boundary: if a worker crashes after the provider accepted the message but before recording it, a retry may resend. Keep retry windows small and record status quickly.

### Retries

- **Transient errors** (timeout, 5xx, throttling): retry with **exponential backoff and jitter**, up to a limit.
- **Permanent errors** (invalid token, invalid number, hard bounce): do **not** retry; mark failed and update the data (below).
- Messages that exhaust retries go to a **dead-letter queue** for inspection and replay.
- Respect **expiry**: an OTP that arrives after it expired is useless, so attach a time-to-live and drop stale messages.

### Provider Failures and Failover

- Wrap every provider call with **timeouts and a circuit breaker** (see [Circuit Breaker Pattern](system-design/circuit-breaker)).
- For SMS and email, keep **two or more providers**; if the primary fails or throttles, route critical traffic to the backup.
- Respect each provider's **rate limits** with a token-bucket limiter per provider (see [Rate Limiting](system-design/rate-limiting)), so you slow down rather than get blocked.

### Keeping Data Clean

| Signal | Action |
|---|---|
| Push provider says the token is **unregistered or invalid** | Deactivate that device token |
| Email **hard bounce** or **spam complaint** | Add to a **suppression list**; never email that address again (protects sender reputation) |
| SMS to an **invalid number** | Mark the number invalid |
| User **unsubscribes** | Update preferences immediately; honor it everywhere |

---

## Bulk Campaigns

Sending to 20 million users must not overwhelm anything.

1. The campaign defines an **audience** (a segment query, resolved into user IDs in batches).
2. A job **fans out in chunks** (say 10,000 users per task) and publishes them to the low-priority queue, not all at once.
3. A **rate limit** controls the send speed (for example 10,000/second), and workers pause if providers throttle or critical traffic needs capacity.
4. Respect **time zones and quiet hours**: schedule "9 a.m. local time" per user, not one global moment.
5. Apply **frequency caps** (for example at most 3 marketing pushes per day per user) and preferences at send time, not only when the campaign is created.
6. Support **pause, resume, and cancel**, and make the fan-out job **resumable** (checkpoint progress) so a crash doesn't restart from zero.

---

## Scheduling

Delayed and scheduled notifications need a scheduler:

- Store `send_at` in a **delay queue** or time-bucketed table (for example a Redis sorted set scored by timestamp, or a database index on `send_at`).
- A scheduler polls the next due items (a few seconds ahead) and publishes them to the proper queue.
- Make it **horizontally scalable and safe**: several scheduler instances may poll, so claim items atomically (a conditional update or a lease) to avoid sending twice. See [Distributed Systems Essentials](system-design/distributed-systems-essentials).

---

## Templates and Content

- Store **versioned templates** with placeholders and **locale** variants; render at send time with the request's parameters.
- Escape user-supplied values in HTML emails to prevent injection.
- Keep **push payloads small** (providers limit size) and **never put sensitive data** (full OTP text, personal details) into a push preview unless intended; the lock screen is visible to others.
- Email needs plain-text and HTML parts, unsubscribe links, and proper headers.

---

## Delivery Tracking

- **Provider callbacks (webhooks)** report delivered, bounced, or complained. They may arrive **late, out of order, or twice**, so make the handler idempotent and never move a status backwards.
- **Opens and clicks:** a tracking pixel and redirect links for email; app-reported events for push. These are approximate (images blocked, privacy features).
- Stream status events into analytics (Kafka → data warehouse) for delivery rate, latency, and campaign results.
- Expose status via the API for debugging ("why didn't I get the OTP?").

---

## Scaling the System

- **Stateless workers** scale horizontally; partition Kafka topics by `userId` so one user's notifications stay ordered and spread evenly.
- **Cache** preferences, templates, and device tokens (they are read on every message and change rarely), invalidating on update. See [Caching and CDN](system-design/caching-and-cdn).
- **Batch** provider calls where supported (FCM and email APIs accept batches), and reuse HTTP/2 connections to APNs.
- Scale **per channel independently**: SMS and email have different limits and costs than push.
- **Multi-region** deployment with replicated queues for availability.

---

## Security and Privacy

- **Authenticate** and authorize internal callers (service tokens), and rate limit them so one buggy service can't flood users. See [Authentication and Security](system-design/auth-and-security).
- Protect **personal data** (phone numbers, emails, tokens) with encryption and strict access.
- Secure webhook endpoints by **verifying provider signatures**.
- Keep **audit logs**, and comply with opt-out and data-deletion requirements.

---

## Observability

Track and alert on: queue depth and age per priority, **end-to-end latency** (especially for OTP), send success rate per provider, bounce and complaint rates, retry and DLQ volume, provider error rates, and cost per channel. A sudden drop in delivery rate for one provider should trigger automatic failover and an alert. See [Observability and Deployment](system-design/observability-and-deployment).

---

## Bottleneck Resolutions

| Bottleneck | Solution |
|---|---|
| Campaign bursts | Queues, chunked fan-out, rate-limited low-priority pool |
| Marketing delays critical messages | Separate queues and worker pools per priority |
| Duplicate notifications | Idempotency key with unique constraint |
| Provider outage or throttling | Circuit breaker, multiple providers, backoff, per-provider rate limits |
| Hot preference/template lookups | Caching with invalidation |
| Invalid tokens and bad addresses | Feedback loops that deactivate tokens and suppress addresses |
| Scheduler double-sending | Atomic claim or lease on due items |
| Cost of SMS | Prefer push/email, fall back to SMS only when needed, cap and monitor |
| Out-of-order or duplicate webhooks | Idempotent status updates, never regress a status |

---

## Interview Questions

**Q: How do you make sure a notification isn't sent twice?**
Attach an idempotency key to each request and check it atomically (unique constraint or `SET NX`) before sending, so duplicates from retries or queue redelivery are dropped. Delivery is at-least-once with deduplication.

**Q: How do you stop a marketing campaign from delaying OTPs?**
Use separate queues and worker pools per priority and channel, and rate-limit the bulk pool so critical traffic always has capacity.

**Q: What if the SMS provider is down?**
A circuit breaker stops calls to it, and critical messages fail over to a second provider. Others are retried with backoff or queued.

**Q: How do you handle invalid or expired push tokens?**
Provider responses tell you a token is unregistered. Deactivate it so you stop sending, and refresh tokens when the app reports a new one.

**Q: How would you send 20 million notifications for a campaign?**
Resolve the audience in batches, fan out in chunks to a low-priority queue, control the send rate, respect time zones, preferences, and frequency caps, and make the job pausable and resumable.

**Q: How do you schedule a notification for a future time?**
Store it with `send_at` in a delay queue or time-indexed store; a scheduler claims due items atomically and publishes them to the right queue.

**Q: Why return 202 instead of 200?**
Sending is asynchronous: the request is accepted and queued, not yet delivered. The caller can check status later.

**Q: How do you track delivery?**
Provider webhooks and app events update the status, handled idempotently since they can arrive late or twice.

---

## Summary

| Component | Choice |
|---|---|
| Ingestion | API returning 202 plus Kafka for events |
| Processing | Stateless service: dedupe, preferences, templates, routing |
| Isolation | Separate queues and worker pools per channel and priority |
| Reliability | Idempotency keys, retries with backoff and jitter, DLQ, TTL on messages |
| Providers | APNs, FCM, SMS and email vendors, with circuit breakers, rate limits, and failover |
| Data | Relational DB for preferences and templates, wide-column store for logs, Redis cache |
| Campaigns | Chunked, rate-limited fan-out, time zones, frequency caps |
| Tracking | Provider webhooks (idempotent), analytics pipeline |
| Data hygiene | Token cleanup, suppression lists, instant unsubscribes |

## Continue Learning

- [Messaging and Communication](system-design/messaging-and-communication) — queues, Kafka, and delivery guarantees.
- [Event-Driven Architecture](system-design/event-driven) — how services publish events that trigger notifications.
- [Rate Limiting](system-design/rate-limiting) — per-user caps and provider throttling.
- [Circuit Breaker Pattern](system-design/circuit-breaker) — protecting the system from failing providers.
- [Design WhatsApp](system-design/whatsapp-design) — push notifications for offline users in a chat system.
