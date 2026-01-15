# WhatsApp – System Design

## Scope & Priorities
### In scope
- 1–1 chat
- Group chat (small groups; large groups discussed separately)
- Push notifications

### Out of scope
- Audio & video calls
- Deep crypto internals (E2EE details)

### Core priorities
- Low latency → chat UX dies if slow
- Strict ordering → non-negotiable for trust
- Ordering path is effectively CP (Consistency + Partition tolerance): small availability sacrifices are acceptable to preserve order; other components aim for high availability.

## Problem statement
Design a globally scalable real-time messaging system similar to WhatsApp that supports:
- Billions of users; millions of concurrent connections
- Reliable message delivery with ordered messaging
- Offline support and multi-device sync

## Requirements
### Functional
- 1:1 and group messaging
- Message status: sent / delivered / read
- Offline message delivery
- Multi-device support
- Media references (objects in separate store)
- Push notifications
- Presence (online / last seen) – eventual consistency

### Non-functional
- Scale: 1B+ users, 100M+ concurrent connections
- Throughput: millions of messages/sec (peak)
- Send ACK latency < 50 ms (regional); delivery p99 < 200 ms
- FIFO ordering per conversation; no message loss (delay acceptable)
- High availability at service level; CP for ordering/commit path

### Out of scope
- Payments, status/stories
- Audio/video calls
- Cryptography deep dive

## Scale assumptions (rough but realistic)
- DAU: 100M–500M
- Concurrent users: ~5–10% of DAU
- Messages/user/day: 30–100 → 3B–50B/day (~35k–580k msg/sec avg; peaks to millions/sec)
- Message size: ~1 KB payload (~2 KB with metadata)
- Small groups: ≤ 200; large groups: up to 10k+

## Communication protocol
- WebSocket over TCP; long-lived, bi-directional
- Auth during handshake (token-based)
- Auto-reconnect with resume (client sends last_received_seq_no)

## High-level architecture
Core components:
- Client (persistent connection: WebSocket/TCP)
- API Gateway
- Connection Gateway (WebSocket termination)
- Chat Service (stateless)
- Sequencer Service (sharded by conversation/chatId)
- Append-only durable log (Kafka-like/WAL)
- Message Store (source of truth)
- Media Store (Object Storage + CDN)
- Fanout Service (async workers)
- Push Notification Service
- Connection Registry (userId → serverId mapping)

Key principle: separate connections, ordering, and delivery — never mix them.

## Connection management
- Each device maintains its own WebSocket connection
- Identity = (userId, deviceId); IP is not identity
- Connections terminate at Connection Gateways
- Live connection state in-memory at gateways; shared mapping userId → active gateways
- Gateways are stateless & disposable
- Failure: gateway crash → disconnect → client reconnects (LB); missed messages resynced on reconnect

## Message flow (1–1 chat)
1) Client sends message over persistent connection
2) API Gateway → Chat Service
3) Chat Service calls Sequencer (per chat shard)
4) Sequencer assigns monotonic sequenceNumber/messageId
5) Message appended to durable log; persisted in Message Store
6) Sender gets Sent ACK (after durability)
7) Fanout to receiver shard/device
8) Receiver device gets message
9) Receiver sends Delivered ACK

Sender ACK does NOT wait for receiver ACK.

## ACK semantics
| ACK Type  | When it is sent           | Purpose               |
|-----------|---------------------------|-----------------------|
| Sent      | After durable persistence | Guarantees no loss    |
| Delivered | Receiver device ACK       | Delivery confirmation |
| Read      | Client UI event           | UX only               |

Rule: never ACK before durability. There is no single ACK.

## Ordering guarantees (core)
- FIFO per conversation; not global; not per user
- Map conversations to sequencer shards; all messages for a chat go through the same shard
- Sequencer assigns monotonic sequence; fanout occurs after ordering
- Client buffers; delivers only lastSeq + 1; fetch missing messages on gaps

## Sequencer design
What it is:
- Logical service responsible only for ordering

What it isn’t:
- Redis INCR; DB auto-increment; global counter

Model:
- One leader sequencer per chat, sharded by chatId (`hash(conversationId) → shard`)
- Append-only ordering; sequence = log offset

Failure handling:
- Leader–follower replication; only leader assigns IDs
- State persisted in durable log; on crash elect new leader (ZooKeeper/Raft), read last committed offset, continue without gaps
- Chat Service is not the source of truth for ordering

## Storage model (conceptual)
Message fields:
- messageId, conversationId, senderId, sequenceNumber, payload/mediaRef, timestamp, delete tombstone

Retention:
- Stored until devices sync + retention window; media separate with TTL
- Per-chat append-only log (not per-user inbox) enables replay and natural ordering

## Storage design (Cassandra/HBase style)
Optimized for high write throughput, ordered reads, and horizontal scale.

Message table:
- Partition Key: chat_id
- Clustering Key: sequence_no (ASC)
- Columns: message_id, sender_id, content (encrypted), timestamp

User message state table:
- Partition Key: user_id
- Clustering Key: (chat_id, message_id)
- Columns: read_at, deleted_flag

Delete semantics:
- Delete for me → per-user delete flag; client hides message
- Delete for everyone → special control message; clients replace content

## Group messaging & fanout
Strategy:
- 1–1 chat → fanout-on-write (push)
- Small groups (≤ ~200) → fanout-on-write is safe and predictable
- Large groups (≥ ~10k) → prefer fanout-on-read to avoid write/cache explosion; store once per chat and resolve recipients dynamically on read

Note: Fanout-on-read is typical for feeds; use it for extreme-size groups only.

Large group handling:
- Batch fanout when pushing; collapse push notifications
- Rate limits per group; slow-path delivery for huge groups

## Multi-device sync & offline handling
- Each device tracks lastSeenSequence
- Online user → immediate push
- Offline user → store + push notification
- On reconnect/new device → replay from last delivered ID; fetch where `seq > lastSeen`
- Media fetched lazily; ordering preserved across devices

## Presence
- Eventual consistency for presence (online/last seen)
- Publish/subscribe updates; tolerate brief staleness

## Scaling strategy & bottlenecks
Dimensions: users, connections, messages, groups, devices, regions

Horizontal scaling:
- Connection Gateways scale independently
- Chat Service is stateless
- Sequencer shards by conversation
- Fanout via async workers

First bottleneck to break: network egress, not storage

## Key optimizations
- Batch fanout per shard; reuse persistent connections
- Compress payloads; region-local delivery
- Avoid duplicate sends; mandatory client-side dedup

## Backpressure & protection
- Rate limits per user & group; monitor queue depth
- Drop non-critical features first; protect core messaging path

## Failure handling
Connection Gateway failure:
- Client reconnects; messages resynced

Chat Service failure:
- ACK only after durability; retries reuse same messageId; idempotent writes

Sequencer failure:
- Restart from last committed offset; ordering preserved

Fanout failure:
- At-least-once delivery; retry until device ACK; clients dedup via sequence/messageId

Storage failure:
- Multi-AZ replication; apply backpressure → delay ACKs (never skip durability)

## Observability
Track:
- Send ACK latency; delivery lag
- Fanout backlog; sequencer hot shards
- Retry rates

---

## End-to-End Encryption (high-level)
- Encryption happens on clients; server sees ciphertext only
- Key model (simplified): public key shared; private key stays on device
- Sender encrypts with receiver’s public key; receiver decrypts with private key
- Offline messages with E2EE: encrypted payload stored and delivered as-is; server cannot decrypt
- Challenges: multi-device keys, key rotation, secure backup/restore, group key management