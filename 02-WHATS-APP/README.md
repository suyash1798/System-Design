# WhatsApp – System Design

---

## Scope & Priorities
### In Scope
- 1–1 chat
- Group chat (max ~200 users)
- Push notifications

### Out of Scope
- Media (images/videos)
- Read receipts
- Last seen / online status

### Core Priorities
- Low latency → chat UX dies if slow
- Strict ordering → non-negotiable for trust
- Ordering path is effectively CP (Consistency + Partition tolerance): short availability sacrifices are acceptable to preserve order; other non-ordering components aim for high availability.

## Problem Statement
Design a globally scalable real-time messaging system similar to WhatsApp that supports:
- Billions of users
- Millions of concurrent connections
- Reliable message delivery
- Ordered messaging
- Offline support
- Multi-device sync

## Requirements

### Functional
- 1:1 messaging
- Group messaging
- Message status: sent / delivered / read
- Offline message delivery
- Multi-device support
- Media sharing (images, videos, docs)
- Push notifications

### Non-Functional
- Scale: 1B+ users, 100M+ concurrent connections
- Throughput: millions of messages/sec (peak)
- Send ACK latency < 50ms (regional)
- Delivery latency p99 < 200ms
- FIFO ordering per conversation
- No message loss (delay acceptable)
- High availability at the service level; CP for the ordering/commit path

### Out of Scope
- Payments
- Status / stories
- Audio & video calls
- Cryptography deep dive (E2EE internals)

## Scale Assumptions (Rough but Realistic)
- DAU: ~100M+
- Concurrent users: ~5–10% of DAU
- Messages/user/day: 30–60
- Message size (text): ~0.5–1 KB
- Max group size: 200; avg group size: 50–100
- P99 latency: sender ACK < 50–100 ms; delivery < 200 ms

## High-Level Architecture
### Core Components
- Client (persistent connection: WebSocket/TCP)
- API Gateway
- Connection Gateway (WebSocket termination)
- Chat Service
- Sequencer Service (sharded by conversation/chatId)
- Append-only durable log (Kafka-like/WAL)
- Message Store (long-term)
- Media Store (Object Storage + CDN)
- Fanout Service
- Push Notification Service

### Key Principle
Separate connections, ordering, and delivery — never mix them.

## Connection Management
### Key Ideas
- Each device maintains its own WebSocket connection
- Identity = (userId, deviceId) — IP is not identity
- Connections terminate at Connection Gateways

### State Handling
- Live connection state → in-memory at gateways
- Lightweight shared mapping: userId → active gateways
- Gateways are stateless & disposable

### Failure Handling
- Gateway crash → connection drops; client reconnects via Load Balancer
- Missed messages are synced on reconnect

## Message Flow (1–1 Chat)
1. Client sends message over persistent connection
2. API Gateway → Chat Service
3. Chat Service calls Sequencer (per chat shard)
4. Sequencer assigns monotonic messageId
5. Message appended to durable log
6. Sender gets Sent ACK (single tick)
7. Message fanout to receiver shard/device
8. Receiver device gets message
9. Receiver sends Delivered ACK (double tick)

Note: Sender ACK does NOT wait for receiver ACK.

## ACK Semantics (Very Important)

| ACK Type  | When it is sent           | Purpose               |
|-----------|---------------------------|-----------------------|
| Sent      | After durable persistence | Guarantees no loss    |
| Delivered | Receiver device ACK       | Delivery confirmation |
| Read      | Client UI event           | UX only               |

Rule: never ACK before durability. There is no single ACK.

## Message Ordering (Core Concept)
### Ordering Scope
- FIFO per conversation
- Not global; not per user

### How Ordering Works
- Conversations are mapped to sequencer shards
- All messages for a conversation go through the same shard
- Sequencer assigns monotonic sequenceNumber
- Fanout happens after ordering

### Client Handling
- Buffer messages
- Deliver only lastSeq + 1
- Fetch missing messages if gaps appear

## Sequencer Design
### What Sequencer Is
- A logical service, not a database
- Responsible only for ordering

### What Sequencer Is NOT
- ❌ Redis INCR
- ❌ DB auto-increment
- ❌ Global counter

### Correct Model
- One leader sequencer per chat, sharded by chatId
- Fixed number of sequencer shards; `hash(conversationId) → shard`
- Append-only ordering; sequence = log offset

### Sequencer Failure Handling
- Leader–follower replication; only leader assigns IDs
- State persisted in durable log
- On crash: new leader elected (ZooKeeper/Raft), reads last committed messageId, continues without gaps
- Chat Service is NOT source of truth for ordering

## Storage Model
### Message Schema (Conceptual)
- `messageId`
- `conversationId`
- `senderId`
- `sequenceNumber`
- `payload` / media reference
- `timestamp`
- `delete` tombstone flag

### Retention
- Stored until all active devices sync, plus retention window
- Media stored separately with TTL

- Per-chat append-only log (not per-user inbox) for easy replay and natural ordering

## Group Messaging & Fanout
### Strategy
- 1–1 chat → fanout-on-write (push)
- Group chat (≤ 200 users) → fanout-on-write is safe and predictable

Note: Fanout-on-read is for feeds, not chat.

### Large Group Handling
- Batch fanout
- Push notification collapse
- Rate limits per group
- Slow-path delivery for huge groups

## Multi-Device Sync & Offline Handling
- Each device tracks `lastSeenSequence`
- Online user → immediate push
- Offline user → store + push notification
- On reconnect/new device → replay from last delivered ID; fetch messages where `seq > lastSeen`
- Media fetched lazily; ordering preserved across devices

## Scaling Strategy & Bottlenecks
### Dimensions
- Users, Connections, Messages, Groups, Devices, Regions

### Horizontal Scaling
- Connection Gateways scale independently
- Chat Service is stateless
- Sequencer sharded by conversation
- Fanout via async workers

### First Bottleneck to Break
- Network egress, not storage

## Key Optimizations
- Batch fanout per shard
- Reuse persistent connections
- Compress payloads
- Region-local delivery
- Avoid duplicate sends
- Client-side dedup (mandatory)

## Backpressure & Protection
- Rate limits per user & group
- Queue depth monitoring
- Drop non-critical features first
- Protect core messaging path

## Failure Handling
### Connection Gateway Failure
- Client reconnects; messages resynced

### Chat Service Failure
- ACK only after durability; retries use same `messageId`
- Idempotent writes

### Sequencer Failure
- Restart from last committed offset; ordering preserved

### Fanout Failure
- At-least-once delivery; retry until device ACK

### Storage Failure
- Multi-AZ replication
- Backpressure → delay ACKs (never skip durability)

## Observability
Track:
- Send ACK latency
- Delivery lag
- Fanout backlog
- Sequencer hot shards
- Retry rates

## Rapid-Fire Interview Truths (Memorize)
- Clients generate IDs? ❌ Only client UUID for retries, not ordering
- Out-of-order delivery allowed? ❌ No — strict FIFO per conversation
- Kafka mandatory? ❌ No — any durable append-only log/WAL works