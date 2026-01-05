# News Feed – System Design

---

## Problem Statement
Design a News Feed system that:
- Shows posts from friends / followers
- Supports likes, comments
- Handles celebs with millions of followers
- Is read-heavy, low latency, highly available

## Requirements

### Functional
- User can create a post
- User can see a ranked feed
- Feed supports pagination (infinite scroll)
- Near-real-time updates (not strict real time)

### Non-Functional
- Read heavy (≈25–30x reads vs writes)
- p99 latency < 200 ms
- Highly available (eventual consistency OK)
- Scalable to hundreds of millions of users
- Freshness: near real-time (seconds)

## Read vs Write Characteristics
- Reads dominate writes (100:1 – 1000:1)
- Reads are latency-sensitive (UX-critical)
- Writes can be asynchronous

## Scale Estimation (Back of Envelope)
- DAU: 300M
- Feed reads: ~170K QPS
- Post writes: ~7K QPS
- Followers: avg 200, max 10M+

> Conclusion: Optimize reads over writes.

## High-Level Architecture
### Core Services
- API Gateway
- Feed Service
- Post Service
- Follow Graph Service
- Ranking Service
- Cache (Redis)
- Storage (Cassandra / DynamoDB)
- Message Queue (Kafka)

## Feed Generation Strategy (MOST IMPORTANT)
### Hybrid Model (Industry Standard)

| User Type                      | Strategy        |
|--------------------------------|-----------------|
| Normal users (< 1k followers)  | Fanout-on-write |
| Celebrities (≥ 1k followers)   | Fanout-on-read  |

Why:
- Fanout-on-write causes write amplification for celebs
- Fanout-on-read causes slow feed reads for normal users

## Celebrity / Hot User Handling
- For high-fanout authors:
  - Do not fanout on write
  - Store post in Post DB; cache recent posts per author (small list)
  - On feed read: identify followed high-fanout authors and merge their recent posts
- Prevents Redis/CPU meltdown during celebrity posts

## Data Model
### Post Table
```
Post {
  postId
  authorId
  content
  mediaUrl
  createdAt
  likeCount
  commentCount
}
```

### Follow Store
```
Follow {
  userId        // follower
  followeeId    // followed user
}
```

### User Feed Table (Redis / Cassandra)
```
UserFeed {
  userId
  [
    (postId, createdAt, authorId)
  ]
}
```

> Store ONLY post IDs, not full posts — prevents duplication & large memory usage.

## Write Path (Post Creation)
1. User creates post
2. Post saved in Post DB
3. Event published to Kafka
4. Feed Service:
   - If author followers < threshold → push postId to followers’ feeds (fanout-on-write)
   - Else → do nothing (handled on read for celebs)

## Read Path (Feed Fetch)
1. Client requests feed
2. Feed Service fetches:
   - Precomputed feed (fanout-on-write)
   - Recent posts from celeb followings (fanout-on-read)
3. Merge + sort by time / rank
4. Batch fetch post details
5. Return paginated feed

> Redis-first always. DB is not scanned to rebuild feeds on read.

## Ranking Strategy
- Initial MVP: reverse chronological
- Later add:
  - Engagement score
  - Recency decay
  - User affinity
- Performance:
  - Redis sorted sets; score ≈ `createdAt + epsilon * engagement`
  - O(log N) insert, O(K) read
- Ranking is a separate service (important callout)

## Caching Strategy
- Cache user feed in Redis
- Cache post objects separately
- Use TTL + LRU
- Cache top N (e.g., 500 posts/user)

## Pagination
- Cursor-based pagination
- Use `(createdAt, postId)` as cursor
- Avoid offset-based pagination (slow)

## Delivery Semantics (Fanout)
- At-least-once delivery; exactly-once is avoided (too expensive)
- Duplicate handling:
  - Idempotent writes: `(userId, postId)` unique
  - Redis `ZADD NX` for dedup at insert
  - Optional read-time dedup by `postId`

## Failure Handling
- Kafka lag → feed delay acceptable
- Cache miss → fallback to DB
- Feed rebuild job for corrupted feeds
- Graceful degradation (skip ranking if needed)

## Redis Failure Handling
- Goals: avoid total outage; degrade gracefully
- Strategy:
  - Serve stale feeds if Redis unstable
  - Pause fanout workers during incidents
  - Rate-limit DB fallback reads
  - Redis auto-failover for node failures
  - Lazy cache repopulation after recovery

> Redis is an optimization layer, not a hard dependency.

## Consistency Model
- Eventual consistency
- Feed delay of few seconds acceptable
- Likes/comments updated asynchronously
