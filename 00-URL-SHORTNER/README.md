# URL Shortener – System Design

## 1. Problem Statement
- Generate short URLs for long URLs
- Redirect users with minimal latency
- Handle very high read traffic reliably

## 2. Requirements

### Functional
- Create a unique short URL for a given long URL
- Redirect short URL → original URL
- Track basic click analytics (non-blocking)

### Non-Functional
- Low latency for redirects (<50ms target)
- High availability for redirect service
- Strong consistency for URL creation
- Durability (URLs should not be lost)

### Out of Scope
- Abuse / malware detection
- QR codes
- Advanced user authentication

## 3. Traffic Estimation (Order of Magnitude)
- URL creation: ~100M/year → ~3–4 writes/sec (bursty)
- Redirects: ~10B/year → ~300–500 RPS avg
- Read : Write ≈ 100–1000 : 1

> Note: Redirect is the hot path.

## 4. High-Level Architecture
- API Gateway (auth, rate-limit, routing)
- URL Creation Service
- Redirect Service
- Key Generation Service
- Redis (cache)
- Database (persistent store)
- Async Analytics Pipeline (Queue + Consumers)

## 5. Short URL Generation Strategy
- Dedicated Key Generation Service
- Uses counter ranges per node to avoid bottlenecks
- Counter value encoded using Base62
- Database enforces unique constraint on `short_url`
- On collision (rare), retry with next ID

### Why not random?
- Random Base62 works at small scale
- Predictable counters are easier to reason about at very large scale

## 6. Redirect Flow (Hot Path)
1. Client hits redirect endpoint
2. Redirect service checks Redis (`short_url` → `long_url`)
3. Cache hit → immediate redirect
4. Cache miss → fetch from DB → backfill Redis → redirect
5. Async event sent for analytics

### If Redis fails
- Fallback to DB
- Higher latency but system remains available

## 7. Caching Strategy
- Redis as read-through cache
- Key: `short_url`
- Value: `long_url`
- URLs are immutable → safe for aggressive caching
- Handles traffic spikes and hot keys

## 8. Database Design
Schema:
```
short_url (PK)
long_url
user_id (optional)
created_at
expires_at (optional)
```

- Access pattern: key-based lookup
- Strong consistency on writes
- Read replicas for scalability

## 9. Partitioning Strategy
- Shard by `hash(short_url)` for even distribution
- Geo-replication handled separately for latency
- Avoid geo-sharding on write path

## 10. Analytics Pipeline
- Redirect service publishes click events to a queue
- Analytics consumers process events asynchronously
- Best-effort delivery
- Redirect flow never depends on analytics availability

## 11. Failure Handling Summary
- Redis down → fallback to DB
- Analytics down → drop events
- Duplicate key → DB constraint + retry
- Services are stateless → horizontally scalable

## 12. Trade-offs & Future Improvements
- CDN for extremely hot URLs
- Bloom filter to reduce DB misses
- Rate limiting to prevent abuse
- URL expiration support