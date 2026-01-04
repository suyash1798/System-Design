# WhatsApp – System Design
---

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
- High availability (AP system)
- FIFO ordering per conversation
- No message loss (delay acceptable)

### Out of Scope
- Payments
- Status / stories
- Audio & video calls
- Cryptography deep dive (E2EE internals)

## High-Level Architecture
### Core Components
- Connection Gateway (WebSocket)
- Chat Service
- Sequencer (Ordering Service)
- Fanout Service
- Push Notification Service
- Message Store
- Media Store (Object Storage + CDN)

### Key Principle
Separate connections, ordering, and delivery — never mix them.

## Connection Management
### Key Ideas
- Each device maintains its own WebSocket connection
- Identity = (userId, deviceId)
- IP address is never part of identity
- Connections terminate at Connection Gateways

### State Handling
- Live connection state → in-memory
- Lightweight shared mapping → userId → active gateways
- Gateways are stateless & disposable

### Failure Handling
- Gateway crash → connection drops; client reconnects via Load Balancer
- Missed messages are synced on reconnect

## Message Lifecycle (Golden Path)
1. Client sends message with clientMsgId
2. Chat Service assigns serverMsgId
3. Message is durably persisted
4. Server sends Sent ACK
5. Sequencer assigns sequenceNumber
6. Fanout to recipients
7. Receiver device sends Delivered ACK
8. Read ACK handled separately

Rule: never ACK before durability.

## ACK Semantics (Very Important)

| ACK Type  | When it is sent              | Purpose                 |
|-----------|------------------------------|-------------------------|
| Sent      | After durable persistence    | Guarantees no loss      |
| Delivered | Receiver device ACK          | Delivery confirmation   |
| Read      | Client UI event              | UX only                 |

Note: There is no single ACK.

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
- Fixed number of sequencer shards
- `hash(conversationId) → shard`
- Append-only ordering
- Sequence = log offset

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

## Multi-Device Sync
- Each device tracks `lastSeenSequence`
- On reconnect or new device:
	- Fetch messages where `seq > lastSeen`
	- Media fetched lazily
	- Ordering preserved across devices

## Delete Semantics
### Delete for Me
- Local only; no server change

### Delete for Everyone
- Server marks message as deleted
- Tombstone event broadcast
- Ordering preserved
- Content hidden, not removed

## Group Messaging & Fanout
### Fanout Strategy
- Fanout-on-read, not fanout-on-write

### Why
- Avoid N writes for large groups
- Store one copy per conversation
- Deliver dynamically to devices

### Large Group Handling
- Batch fanout
- Push notification collapse
- Rate limits per group
- Slow-path delivery for huge groups

## Scaling Strategy
### Dimensions
- Users, Connections, Messages, Groups, Devices, Regions

### Horizontal Scaling
- Connection Gateways scale independently
- Chat Service is stateless
- Sequencer sharded by conversation
- Fanout via async workers

## Failure Handling
### Connection Gateway Failure
- Client reconnects; messages resynced

### Chat Service Failure
- ACK only after durability
- Client retries with same `messageId`
- Idempotent writes

### Sequencer Failure
- Restart from last committed offset
- Ordering preserved

### Fanout Failure
- At-least-once delivery
- Retry until device ACK

### Storage Failure
- Multi-AZ replication
- Backpressure → delay ACKs (never skip durability)

## Backpressure & Protection
- Rate limits per user & group
- Queue depth monitoring
- Drop non-critical features first
- Protect core messaging path

## Observability
Track:
- Send ACK latency
- Delivery lag
- Fanout backlog
- Sequencer hot shards
- Retry rates