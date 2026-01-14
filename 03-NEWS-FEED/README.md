# News Feed – System Design

## Problem statement
Design a News Feed system that:
- Shows posts from friends/followers
- Supports likes and comments
- Handles celebrities with millions of followers
- Is read-heavy, low latency, and highly available

## Requirements
### Functional
- User can create a post
- User can see a ranked, paginated feed (infinite scroll)
- Near-real-time updates (seconds, not strict real time)
- Follow/unfollow users
- Likes/comments contribute to ranking signals

### Non-Functional
- Read heavy (≈25–30x reads vs writes)
- p99 feed latency target < 200 ms (or stricter if required)
- High availability; eventual consistency acceptable
- Scalable to hundreds of millions of users
- Freshness: near real-time (few seconds acceptable)

## Read vs write characteristics
- Reads dominate writes (100:1 – 1000:1)
- Reads are latency-sensitive (direct UX impact)
- Writes can be asynchronous

## Scale estimation (back of envelope)
- DAU: 300M
- Feed reads: ~170K QPS
- Post writes: ~7K QPS
- Followers: avg 200, max 10M+

Conclusion: prioritize read performance over write cost.

## Core design insight
- Never scan all followers on read.
- Never rely on cache as the only source of truth.
- Use a hybrid fanout model (industry standard).

## High-level architecture
Core services:
- API Gateway
- Feed Service
- Post Service
- Follow Graph Service
- Ranking Service
- Cache (Redis)
- Storage (Cassandra/DynamoDB)
- Message Queue (Kafka)

## Feed generation strategy (MOST IMPORTANT)
### Hybrid model

| User type                     | Strategy         |
|------------------------------|------------------|
| Normal users (< ~1k followers)| Fanout-on-write  |
| Celebrities (≥ ~1k followers) | Fanout-on-read   |

Why:
- Fanout-on-write amplifies writes for celebs (too expensive)
- Fanout-on-read makes normal users’ reads slow (too expensive)

## Celebrity / hot user handling
- For high-fanout authors:
  - Don’t fanout on write
  - Store post in Post DB; cache recent posts per author (small list)
  - On feed read: identify followed high-fanout authors and merge their recent posts
- Prevents Redis/CPU meltdown on celebrity posts.

## Data model
### Post
```
Post {
  postId,
  authorId,
  content,
  mediaUrl,
  createdAt,
  likeCount,
  commentCount
}
```

### Follow
```
Follow {
  userId,      // follower
  followeeId   // followed user
}
```

### UserFeed (Redis/Cassandra)
```
UserFeed {
  userId,
  [ (postId, createdAt, authorId) ]
}
```

### FeedItem (minimum viable)
```
FeedItem {
  feedUserId,
  postId,
  authorId,
  createdAt,
  rankScore,
  sourceType // fanout-write | fanout-read
}
```

Store ONLY post IDs in user feeds (not full posts) to avoid duplication and reduce memory.

## Write path (post creation)
1) User creates post
2) Post saved in Post DB
3) Event published to Kafka
4) Feed Service:
   - If author followers < threshold → push postId to followers’ feeds (fanout-on-write)
   - Else → skip fanout (handled on read for celebs)

## Read path (feed fetch)
1) Client requests feed
2) Feed Service fetches:
   - Precomputed feed page (fanout-on-write)
   - Recent posts from celeb followings (fanout-on-read)
3) Merge + sort (by time/rank)
4) Batch fetch post details
5) Apply lightweight ranking if needed; cache the result (short TTL)
6) Return paginated feed

Redis-first for performance; the DB is not scanned to rebuild feeds during read.

## Ranking strategy
- MVP: reverse chronological
- Evolve to include:
  - Engagement score
  - Recency decay
  - User affinity
- Implementation notes:
  - Redis sorted sets; score ≈ `createdAt + epsilon * engagement`
  - O(log N) insert; O(K) read
- Ranking is a separate service.

## Caching strategy
- Cache user feed pages in Redis (short TTL)
- Cache post objects separately
- Use TTL + LRU
- Cache top N (e.g., 500 posts/user)

## Pagination
- Cursor-based pagination using `(createdAt, postId)`
- Avoid offset-based pagination (slow and inconsistent)

## Delivery semantics (fanout)
- At-least-once delivery; exactly-once generally avoided (too expensive)
- Duplicate handling:
  - Idempotent writes: enforce `(userId, postId)` uniqueness
  - Redis `ZADD NX` (or equivalent) for dedup on insert
  - Optional read-time dedup by `postId`

## Failure handling
- Kafka lag → acceptable feed delay
- Cache miss → fallback to DB
- Feed rebuild job for corrupted feeds
- Graceful degradation (skip ranking if needed)
- Service robustness:
  - Feed Service is stateless; multiple replicas behind LB
  - Auto-scaling + health checks
  - If cache down → serve from Feed DB, rate-limit if needed
  - Fanout worker lag → eventual consistency; users may see posts slightly late

## Redis failure handling
- Degrade gracefully; avoid total outage
- Strategies:
  - Serve stale feeds if Redis unstable
  - Pause fanout workers during incidents
  - Rate-limit DB fallback reads
  - Redis auto-failover for node failures
  - Lazy cache repopulation after recovery

Redis is an optimization layer, not a hard dependency.

## Consistency model
- Eventual consistency
- Feed delay of a few seconds acceptable
- Likes/comments updated asynchronously