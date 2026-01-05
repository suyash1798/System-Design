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

## Scale Estimation (Back of Envelope)
- DAU: 300M
- Feed reads: ~170K QPS
- Post writes: ~7K QPS
- Followers: avg 200, max 10M+

Note: Read optimization > write optimization.

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

| User Type                     | Strategy          |
|------------------------------|-------------------|
| Normal users (< 1k followers) | Fanout-on-write   |
| Celebrities (≥ 1k followers)  | Fanout-on-read    |

Why:
- Fanout-on-write causes write amplification for celebs.
- Fanout-on-read causes slow feed reads for normal users.

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

### User Feed Table (Redis / Cassandra)
```
UserFeed {
  userId
  [
    (postId, createdAt, authorId)
  ]
}
```

Note: Store ONLY post IDs, not full posts — prevents duplication and large memory usage.

## Write Path (Post Creation Flow)
1. User creates post
2. Post saved in Post DB
3. Event published to Kafka
4. Feed Service:
   - If author followers < threshold → push postId to followers’ feeds (fanout-on-write)
   - Else → do nothing (handled on read for celebs)

## Read Path (Feed Fetch Flow)
1. Client requests feed
2. Feed Service fetches:
   - Precomputed feed (fanout-on-write)
   - Recent posts from celeb followings (fanout-on-read)
3. Merge + sort by time / rank
4. Batch fetch post details
5. Return paginated feed

## Ranking Strategy
- Initial MVP: reverse chronological
- Later add:
  - Engagement score
  - Recency decay
  - User affinity

Note: Ranking is a separate service.

## Caching Strategy
- Cache user feed in Redis
- Cache post objects separately
- Use TTL + LRU
- Cache top N (e.g., 500 posts/user)

## Pagination
- Cursor-based pagination
- Use (createdAt, postId) as cursor
- Avoid offset-based pagination (slow)

## Failure Handling
- Kafka lag → feed delay acceptable
- Cache miss → fallback to DB
- Feed rebuild job for corrupted feeds
- Graceful degradation (skip ranking if needed)

## Consistency Model
- Eventual consistency
- Feed delay of few seconds acceptable
- Likes/comments updated async