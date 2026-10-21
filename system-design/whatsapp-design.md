---
title: "System Design Case Study — WhatsApp (Instant Messaging at Scale)"
description: "A complete system design for building a WhatsApp-like instant messaging service: real-time messaging, read receipts, last seen, push notifications, media storage, and multi-region architecture."
author: ["name": "Rajendra Pancholi", "email": "rpancholi522@gmail.com"]
created: "2026-04-18"
updated: "2026-10-17"
thumbnail: "/images/whatsapp-system-design.png"
tags: [system-design, whatsapp, messaging, websockets, case-study, scalability]
keywords: ["WhatsApp system design", "Design instant messaging service", "Real-time chat system design", "WebSockets messaging architecture", "Push notification system design", "End-to-end encryption Signal protocol", "Message ordering and delivery guarantees"]
---

# System Design Case Study — WhatsApp (Instant Messaging at Scale)

WhatsApp connects over 2 billion users across 180 countries, processing 100 billion messages per day. Designing a system at this scale requires careful decisions about real-time communication protocols, message delivery guarantees, media storage, and multi-region infrastructure.

![WhatsApp System Design](/images/whatsapp-system-design.png)

## Requirements

### Functional Requirements

- One-on-one chat between users
- Group chats (up to 100 participants)
- File sharing: images, videos, documents
- Sent, Delivered, and Read receipts (double-tick system)
- Online presence and last seen timestamps
- Push notifications for offline users

### Non-Functional Requirements

- High availability with minimal latency (< 100ms message delivery)
- Messages must not be lost — at-least-once delivery
- Horizontally scalable to support 50M+ daily active users
- End-to-end encryption (architectural consideration)

### Extended Requirements

- Voice and video calls (not covered in depth here)
- Message search
- Broadcast lists

## Capacity Estimation

- **50 million daily active users (DAU)**
- Each user sends 40 messages/day to ~4 different people

```
Daily messages: 50M × 40 = 2 billion messages/day
RPS: 2 billion / 86,400 = ~24,000 messages/second
```

**Media files (5% of messages):**

```
100 million media files/day × 100 KB average = 10 TB/day
```

**Total storage for 10 years:**

```
(10 TB media + 0.2 TB text) × 365 × 10 = ~38 PB
```

**Bandwidth:**

```
10.2 TB/day ÷ 86,400s = ~120 MB/second ingress
```

| Metric             | Estimate   |
| ------------------ | ---------- |
| Daily active users | 50 million |
| Messages/second    | ~24,000    |
| Storage/day        | ~10.2 TB   |
| Storage/10 years   | ~38 PB     |
| Bandwidth          | ~120 MB/s  |

## Data Model

### `users` table

| Column            | Type             |
| ----------------- | ---------------- |
| `id`              | UUID             |
| `phone_number`    | VARCHAR (unique) |
| `name`            | VARCHAR          |
| `last_seen`       | TIMESTAMP        |
| `profile_pic_url` | VARCHAR          |

### `messages` table

| Column         | Type      | Description                         |
| -------------- | --------- | ----------------------------------- |
| `id`           | UUID      | Primary key                         |
| `chat_id`      | UUID      | Associated chat or group            |
| `sender_id`    | UUID      | Sender user ID                      |
| `type`         | ENUM      | text, image, video, document, audio |
| `content`      | TEXT      | Message body or media URL           |
| `sent_at`      | TIMESTAMP | When sent by sender                 |
| `delivered_at` | TIMESTAMP | When delivered to recipient device  |
| `seen_at`      | TIMESTAMP | When recipient opened the chat      |

### `chats` table (private chats between two users)

| Column       | Type      |
| ------------ | --------- |
| `id`         | UUID      |
| `created_at` | TIMESTAMP |

### `users_chats` table (N:M mapping)

| Column    | Type |
| --------- | ---- |
| `user_id` | UUID |
| `chat_id` | UUID |

### `groups` table

| Column       | Type      |
| ------------ | --------- |
| `id`         | UUID      |
| `name`       | VARCHAR   |
| `created_by` | UUID      |
| `created_at` | TIMESTAMP |

### `users_groups` table (N:M mapping)

| Column      | Type                 |
| ----------- | -------------------- |
| `user_id`   | UUID                 |
| `group_id`  | UUID                 |
| `joined_at` | TIMESTAMP            |
| `role`      | ENUM (admin, member) |

**Database Choice:** Apache Cassandra or HBase for the messages table (high write throughput, time-range queries, natural horizontal sharding). PostgreSQL for user and chat metadata.

## API Design

```
// Get all chats/groups for a user
GET /users/{userID}/chats
Response: [{ chatID, lastMessage, unreadCount, participants }]

// Get messages in a chat
GET /chats/{chatID}/messages?before={timestamp}&limit=50
Response: [{ messageID, senderID, content, type, sentAt, deliveredAt, seenAt }]

// Send a message
POST /chats/{chatID}/messages
Body: { senderID, content, type }
Response: { messageID, sentAt }

// Create a group
POST /groups
Body: { name, memberIDs }
Response: { groupID }

// Join/leave a group
POST /groups/{groupID}/members
DELETE /groups/{groupID}/members/{userID}
```

## Architecture

### Microservices

| Service                  | Responsibility                                           |
| ------------------------ | -------------------------------------------------------- |
| **User Service**         | Authentication, user profiles, phone number registration |
| **Chat Service**         | WebSocket connections, real-time message delivery        |
| **Notification Service** | Push notifications (FCM/APNs) for offline users          |
| **Presence Service**     | Online status, last seen tracking                        |
| **Media Service**        | Upload, process, and serve media files                   |
| **Group Service**        | Group creation, membership management                    |

Inter-service communication via **gRPC** for low latency. Service discovery via Consul or Kubernetes DNS.

## Real-Time Messaging — WebSocket Architecture

The pull model (HTTP polling) is not viable for messaging — it creates massive unnecessary load and has high latency. The push model via **WebSockets** is the right choice.

![WebSocket Messaging](/images/websockets.png)

### How message delivery works:

1. **Sender** types a message and hits send.
2. **Chat Service** receives the message over the sender's WebSocket connection.
3. Chat Service stores the message in the database (with `delivered_at = null`, `seen_at = null`).
4. Chat Service looks up the **recipient's active WebSocket connection** (via a connection registry in Redis).
5. If recipient is **online**: push the message directly over WebSocket → mark `delivered_at = now()`.
6. If recipient is **offline**: enqueue a notification event in the message queue → Notification Service sends FCM/APNS push notification.
7. When recipient opens the chat → mark `seen_at = now()` → push read receipt back to sender via WebSocket.

### Connection Registry (Redis)

When a user connects via WebSocket, register their connection:

```
SET connection:{userID} {chatServerID} EX 3600
```

When the Chat Service needs to deliver a message, it looks up which Chat Server hosts the recipient's connection, then routes the message to that server.

### Message Queue for Reliability

Even with WebSockets, message delivery must be reliable. Messages are written to a message queue (Kafka) before delivery. The Chat Service consumes from Kafka to ensure messages are processed even if the delivery server restarts.

## Message IDs, Ordering, and Reliable Delivery

Mobile networks are unreliable: a send can time out even though the server received it, and the app will retry. The design must make this safe.

### Client Message ID (Idempotent Sends)

- The sending app generates a **unique `clientMessageId`** (a UUID) for every message and includes it with the send.
- The server **deduplicates** on `(senderId, clientMessageId)`. If the app retries after a timeout, the server recognizes the repeat, does not create a second message, and returns the original result.
- This is the same idempotency idea used for payments (see [Messaging and Communication](system-design/messaging-and-communication)).

### Ordering Within a Chat

Messages must appear in the same order for everyone, but **device timestamps cannot be trusted** (clocks differ and can be wrong).

- Route all messages of a chat to the **same shard or Kafka partition (key = `chatId`)**, so a single writer handles that chat.
- That writer assigns a **per-chat sequence number** that only increases (1, 2, 3, ...). The sequence number, not the timestamp, defines the order.
- Clients display messages sorted by sequence number.
- **Gap detection:** if a client has messages 41 and 43, it knows 42 is missing and requests it (`GET /chats/{chatId}/messages?after=41`). This also lets a device that was offline catch up from its last known sequence.
- Order is guaranteed **per chat**, not across all chats, which is all that users need and is far cheaper than global ordering.

### At-Least-Once Delivery with Acknowledgements

1. The server stores the message and acknowledges to the sender (**one tick**: server received it).
2. The server pushes it to the recipient's device(s). The device sends an **ACK** when it has stored the message (**two ticks**: delivered).
3. If no ACK arrives (device offline or connection dropped), the message stays stored on the server and is **re-sent when the device reconnects**.
4. The receiving app **deduplicates by message ID**, because redelivery can produce duplicates.
5. When the recipient opens the chat, a **read receipt** is sent (blue ticks).

Messages that have been delivered to all of a user's devices no longer need to sit on the server (the server is a store-and-forward relay, with a limited retention period for undelivered messages).

## Multi-Device Support and Group Receipts

A user may be logged in on a phone, a laptop, and a tablet.

- The **connection registry** maps `userId → set of (deviceId, chatServerId)`, instead of a single connection.
- A message is delivered to **each device separately**, and each device keeps its own **cursor** (last sequence number received per chat), so a device that comes online later fetches only what it missed. See the similar cursor-based sync in [Design a File Storage Service](system-design/file-storage-design).
- The per-message `delivered_at` and `seen_at` columns shown earlier work for a 1:1 chat. For **groups**, store receipts **per recipient** in a separate table, for example `message_receipts(message_id, user_id, delivered_at, read_at)`, and show a double tick only when everyone has received it (with per-member detail in the message info screen).

## Read Receipts (Sent / Delivered / Seen)

WhatsApp's iconic tick system:

| State         | Indicator        | Mechanism                                                                            |
| ------------- | ---------------- | ------------------------------------------------------------------------------------ |
| **Sent**      | Single grey tick | Message stored in server DB; `sent_at` set                                           |
| **Delivered** | Double grey tick | Message delivered to recipient device; `delivered_at` set; ACK sent to sender        |
| **Seen**      | Double blue tick | Recipient opens the chat; `seen_at` set; read receipt pushed to sender via WebSocket |

**ACK mechanism:** When the recipient's device receives the message, it sends an acknowledgment back to the Chat Service, which updates `delivered_at` and pushes the delivery receipt to the sender.

## Presence Service — Last Seen

The Presence Service tracks the **last active timestamp** of every user.

**Heartbeat mechanism:** The client periodically sends a heartbeat ping every 30 seconds while active. The Presence Service updates a Redis cache:

```
SET presence:{userID} {timestamp} EX 60
```

If no heartbeat is received within 60 seconds, the user is considered offline and `last_seen` is updated in the database.

**Lazy evaluation alternative:** Track the user's last action timestamp. When another user requests someone's presence, if their last action was more than 30 seconds ago, show them as offline with the last_seen timestamp.

## Push Notifications

When a message recipient is offline, the Chat Service enqueues a notification event:

1. **Chat Service** → publishes event to **Amazon SQS / RabbitMQ**
2. **Notification Service** consumes the event
3. Checks device platform (iOS / Android / Web)
4. Routes to **Apple Push Notification Service (APNs)** or **Firebase Cloud Messaging (FCM)**
5. Push notification delivered to device

**Payload includes:** Sender name, message preview (if end-to-end encryption not enabled), chat ID for deep linking.

![Notification Flow](/images/message-queue.png)

## Media Handling

Media files (images, videos, documents) are not transmitted through the chat server. Instead:

1. **Sender** uploads media directly to the **Media Service**.
2. Media Service stores the file in **Amazon S3** (or equivalent object storage).
3. Media Service returns a URL for the uploaded file.
4. **Sender** sends a message with `type=image` and `content=media_url`.
5. **Recipient** downloads the media from the URL (served via **CloudFront CDN** for low latency).

**Media processing:** Images are compressed and resized. Videos are transcoded to web-compatible formats. Thumbnails are generated for preview.

**WhatsApp deletes media from servers** after it has been downloaded — this reduces storage costs significantly.

## Group Chat Architecture

For group chats (up to 100 members):

1. Sender sends one message to the Group Service.
2. Group Service fans out the message to all group members' Chat Service connections.
3. For offline members, fan-out events are queued in the Notification Service.

**Fan-out challenge at scale:** A group with 100 members means one message generates 99 deliveries. For groups with very large memberships (broadcast channels), this fan-out must be handled asynchronously via Kafka to avoid blocking.

## End-to-End Encryption (E2EE)

With end-to-end encryption, **only the sender's and recipient's devices can read a message**. The server relays ciphertext it cannot decrypt, so even a compromised server or a legal demand cannot expose message content. WhatsApp uses the **Signal Protocol** for this.

### How It Works (Simplified)

1. **Key registration:** each device generates a long-term **identity key pair** plus a batch of one-time **pre-keys** and uploads the **public** parts to the server. Private keys never leave the device.
2. **Session setup:** to message someone for the first time, the sender fetches the recipient's public key bundle from the server and performs a **key agreement** (X3DH-style) to derive a shared secret, even if the recipient is offline.
3. **Per-message keys:** the **Double Ratchet** algorithm derives a **new key for every message**, so compromising one key doesn't expose past messages (**forward secrecy**) or future ones (**post-compromise security**).
4. **Sending:** the sender encrypts the message on the device; the server stores and forwards only ciphertext.
5. **Groups:** the sender encrypts the message **once** with a group "sender key" that each member holds, instead of encrypting separately per member for every message. When membership changes, the keys are rotated.
6. **Multi-device:** each device has its own keys, so a message is encrypted once per recipient device.

### Design Consequences

- The server **cannot** search message content, scan it for spam, or generate content previews in push notifications. Anything that needs the plaintext must run **on the device**.
- **Backups** become a key-management problem: cloud backups of chats need their own encryption (optionally with a user-held key).
- **Metadata** (who talks to whom, when, group membership, IP addresses) is still visible to the server unless extra protections are added.
- Users can **verify** each other's keys (a safety number or QR code) to detect a man-in-the-middle attack.
- Media files are encrypted with a random key before upload to storage, and the **key is sent inside the end-to-end encrypted message**, so the storage service can't read them.

## Advanced Architecture

![WhatsApp Advanced Design](/images/whatsapp-advanced-design.png)

### Data Partitioning

- Shard `messages` table by `chat_id` — ensures all messages for a chat reside on the same shard (efficient retrieval)
- Consistent hashing for shard assignment

### Caching

- Cache recent messages (last 20 per chat) in Redis — reduces database load for the common case
- LRU eviction policy

### API Gateway

Since the system uses both HTTP and WebSocket protocols, an API Gateway that supports both is essential. AWS API Gateway supports HTTP + WebSocket. Alternatively, NGINX or Envoy can handle both.

## Bottleneck Resolution

| Bottleneck                       | Solution                                                            |
| -------------------------------- | ------------------------------------------------------------------- |
| Single Chat Service crash        | Multiple instances; WebSocket session recovery                      |
| Message loss during server crash | Kafka ensures durability; consumer retries                          |
| Media storage cost               | Compress + delete after download; lifecycle policies                |
| Group message fan-out            | Async Kafka-based fan-out for large groups                          |
| Last seen accuracy               | Heartbeat with Redis; fallback to last action timestamp             |
| Notification delivery failure    | Dead-letter queue for failed pushes; retry with exponential backoff |

## Interview Questions

**Q: How do you guarantee messages appear in the right order?**
Route a chat's messages to one shard or partition keyed by chat ID, assign a per-chat sequence number there, and have clients sort by it. Timestamps from devices are not reliable.

**Q: What if the sender's app retries a message after a timeout?**
The app includes a client-generated message ID, and the server deduplicates on it, so a retry never creates a second message.

**Q: How do you deliver a message to a user who is offline?**
Store it on the server, send a push notification, and deliver it when the device reconnects, using a per-chat cursor to fetch what was missed. The recipient acknowledges receipt so the server can remove it.

**Q: How do you support multiple devices per user?**
Keep a registry from user to a set of devices, deliver to each device separately with its own cursor and acknowledgement, and track receipts per recipient for groups.

**Q: How does end-to-end encryption change the design?**
The server only relays ciphertext, so it can't read, search, or preview messages. Keys live on devices (Signal protocol with a double ratchet), groups use sender keys, and features that need plaintext must run on the client.

**Q: How do you scale WebSocket connections?**
Spread connections across chat servers, keep a connection registry in Redis, and route messages to the server that holds the recipient's connection, with heartbeats and resumable reconnects. See [Messaging and Communication](system-design/messaging-and-communication).

## Summary

| Component           | Technology                                             |
| ------------------- | ------------------------------------------------------ |
| Real-time messaging | WebSockets (Chat Service)                              |
| Message broker      | Apache Kafka                                           |
| Primary database    | Apache Cassandra (messages), PostgreSQL (users/groups) |
| Cache               | Redis (connections, presence, recent messages)         |
| Media storage       | Amazon S3 + CloudFront CDN                             |
| Push notifications  | FCM (Android) + APNs (iOS)                             |
| API Gateway         | AWS API Gateway / NGINX                                |
| Service discovery   | Consul / Kubernetes DNS                                |
