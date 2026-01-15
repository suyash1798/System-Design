# System Design Fundamentals – Quick Reference

## OLTP vs OLAP
### OLTP (Online Transaction Processing)
- Day-to-day app operations
- Small reads/writes; high concurrency
- Examples: login, place order, like post
- Typical DBs: MySQL, Postgres

### OLAP (Online Analytical Processing)
- Analytics & reporting
- Large scans and aggregates; read-heavy
- Examples: DAU, revenue reports
- Typical DBs: Redshift, BigQuery, Snowflake

One-liner: OLTP is for transactions; OLAP is for analytics.

## Primary key vs index
### Primary key
- Uniquely identifies a row
- NOT NULL + UNIQUE; only one per table
- Automatically indexed

### Index
- Improves query speed; can be non-unique
- Multiple indexes allowed
- Usually implemented with B+ trees; lookup ≈ O(log n)

One-liner: Primary key enforces identity; indexes improve performance.

## Why B+ trees for DB indexes?
- Disk-friendly (fewer disk seeks)
- Supports range queries (<, >, BETWEEN)
- Sorted data; shallow tree even with millions of rows

One-liner: B+ trees support range queries and work efficiently on disk.

## Transactions & ACID
### Transaction
- A group of operations executed as a single logical unit; either all succeed or all fail.

### ACID properties
- Atomicity: all or nothing; partial updates are rolled back
- Consistency: DB moves from one valid state to another; all constraints must hold
- Isolation: concurrent transactions don’t interfere; appears serial
- Durability: once committed, data survives crashes (WAL/redo logs)

Note: Idempotency is NOT part of ACID.

## Isolation problems (must know)
- Dirty read
- Lost update
- Non-repeatable read
- Phantom read

## WAL (Write-Ahead Logging)
- Used for durability
- Changes written to log before DB pages
- Crash recovery via log replay

## Networking: TCP vs UDP
### TCP
- Reliable, ordered delivery; connection-oriented
- Slower; use cases: HTTP, APIs, DBs

### UDP
- Unreliable; no ordering; connectionless
- Faster; use cases: streaming, gaming, DNS

One-liner: TCP prioritizes correctness; UDP prioritizes speed.

## Why HTTP/3 uses UDP (QUIC)
- Built on QUIC
- Avoids TCP head-of-line blocking
- Stream-level loss handling
- Faster handshake (0-RTT)

One-liner: HTTP/3 uses UDP so QUIC can provide reliability without TCP’s HoL blocking.

## Operating systems: process vs thread
### Process
- Separate memory space; strong isolation
- Expensive context switch; better fault tolerance

### Thread
- Shared memory; lightweight
- Cheaper context switch; faster communication

One-liner: Processes provide isolation; threads provide concurrency.

## Common threading problems
- Race conditions
- Deadlocks
- Starvation
- Hard debugging (non-deterministic bugs)

## Deadlock
### Definition
- Threads are permanently blocked, each waiting for resources held by others.

### Four necessary conditions (ALL required)
- Mutual exclusion
- Hold and wait
- No preemption
- Circular wait

Break any one → no deadlock.

One-liner: Deadlock occurs when mutual exclusion, hold-and-wait, no preemption, and circular wait all hold.