# Google Docs – Real-Time Collaborative Editing (System Design)

## Core problem
Build a real-time collaborative document editor where:
- Multiple users edit simultaneously
- No data loss
- All clients converge to the same document
- Typing feels instant

Hard parts: concurrency, ordering, and scale.

## Architecture overview
```
Client (Editor)
   ↓ WebSocket
API Gateway
   ↓
Collaboration Service (authoritative)
   ↓
Persistent Storage (Snapshots + Operation Log)
```

Key ideas:
- WebSockets for real-time bidirectional updates
- One collaboration server per document shard
- In-memory state is authoritative
- Database is for durability, not live coordination

## Client responsibilities
Maintain:
- Confirmed document state
- Buffer of local unacknowledged operations

Send:
- Operations (not full document/chunks)

Handle:
- Applying remote operations
- Transforming operations for consistency
- Retrying operations if ACK not received

## Why not last-write-wins (LWW)
- Loses user input
- Network latency does not reflect real-time order
- Offline edits break convergence
- The editor must never drop characters

## Concurrency control: OT (Operational Transformation)
Why OT:
- Designed for real-time editors
- Central server fits this model
- Preserves user intent

## Operation model
Instead of sending chunks, send operations:
```
Insert(position, text)
Delete(position, length)
```

Each operation includes:
- opId (unique)
- docId
- baseRevision
- userId

## Server-side OT flow
1) Client sends operation with base revision
2) Server checks:
   - If revision is current → apply directly
   - If stale → transform against newer operations
3) Server:
   - Applies transformed operation to in-memory doc
   - Appends operation to log (WAL)
   - ACKs client
   - Broadcasts operation to other clients

Server guarantees intent preservation, deterministic ordering, and no lost updates.

## Client-side OT flow
Client maintains:
- confirmedState
- pendingOps[]

When a remote operation arrives:
1) Transform remote op against pendingOps
2) Apply transformed remote op to editor
3) Transform all pendingOps against the remote op

This prevents cursor jumps and text corruption.

## Conflict example (concurrent inserts)
Initial doc: "Hello"

User A: Insert(1, "X")
User B: Insert(1, "Y")

Server receives B first and transforms A:
```
Insert(1,"X") → Insert(2,"X")
```

Final doc: "H Y X ello" (deterministic order)

No data loss; same result on all clients.

## Routing & scaling
Problem: all users of the same doc must hit the same collaboration server.

Solution:
- Consistent hashing on docId
- Load balancer routes based on hash(docId)
- Presence DB is NOT used for routing

## Presence system
Presence DB stores:
- userId → serverId
- docId → active users
- cursor positions

Used for:
- Showing collaborators
- Cursor movement
- User join/leave events

## Fault tolerance
On server crash:
- Operations are appended to WAL before ACK
- Ops have unique IDs → idempotent retries
- New server rebuilds state from latest snapshot + ops after snapshot

## Reconnect flow
Client sends:
- docId + lastKnownRevision

Server responds with:
- Latest snapshot
- Ops after that revision

Never replay the entire history.

## Storage strategy
- Snapshots (every N operations)
- Append-only operation log
- Compaction to reduce log size

## Non-functional requirements
- Low latency (< 50 ms per keystroke in-region)
- High availability
- Eventual consistency with strong UX guarantees
- Horizontal scalability
- Graceful reconnects

## Key concepts
- Operational Transformation (OT)
- Intent preservation
- Base revision
- Operation log (WAL)
- Idempotent operations
- Consistent hashing
- Authoritative in-memory state
- Snapshot + delta replay

## Summary
This design uses Operational Transformation with a centralized collaboration service that orders and transforms concurrent operations, maintains an authoritative in-memory document state, and ensures all clients converge via transformed operations over WebSockets.