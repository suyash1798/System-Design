# URL Shortener – System Design

---

## Problem statement
- Generate short URLs for long URLs
- Redirect users with minimal latency
- Handle very high read traffic reliably

## Requirements
### Functional
- Create a unique short URL for a given long URL
- Redirect short URL → original URL
- Optional: custom alias and expiry (TTL)
- Track basic click analytics (non-blocking)

### Non-Functional
- Low latency for redirects (target p99 < 50 ms)
- High availability for redirect service
- Strong consistency for URL creation
- Durability (URLs must not be lost)
- Horizontal scalability

### Out of Scope
- Abuse/malware detection
- QR codes
- Advanced user authentication

## Scale and capacity assumptions
Two representative scenarios (choose the one that fits the interview):

- Moderate: ~100M new URLs/year → ~3–4 writes/sec (bursty), ~10B redirects/year → ~300–500 RPS avg; Read:Write ≈ 100–1000:1
- Peak: ~100M new URLs/day; Read:Write ≈ 100:1 → Writes ≈ 1.2k RPS; Reads ≈ 120k RPS; Storage ≈ ~100 GB/day (1 KB/URL) ≈ 36 TB/year

Redirect is the hot path in both scenarios.

## APIs
### Create Short URL
POST /shorten

Request body (JSON):
```json
{
  "originalUrl": "https://example.com",
  "customAlias": "myalias",   
  "expirySeconds": 31536000    
}
```

Responses:
- 201 Created
```json
{ "shortUrl": "https://sho.rt/myalias" }
```
- 400 Bad Request (invalid URL)
- 409 Conflict (alias already exists)

### Redirect
GET /{shortCode}

Responses:
- 302 Found with Location: https://example.com
- 404 Not Found (unknown/expired)
- 503 Service Unavailable (cache miss + DB down)

## High-level architecture
- CDN (edge caching of 302s)
- API Gateway (auth, rate-limit, routing)
- Create Service
- Redirect Service
- Key Generation Service (KGS)
- Redis (hot path cache)
- Primary Database (persistent store)
- Async Analytics Pipeline (Queue + Consumers)

## Short code generation
- Dedicated KGS with pre-allocated counter ranges per node to avoid bottlenecks
- Encode counter with Base62
- 7 chars → 62^7 ≈ 3.5 trillion URLs
- Database enforces unique constraint on short code; on rare collision, retry

Why not random?
- Random Base62 works at small scale; counters are simpler to reason about and avoid probabilistic collision handling at large scale

Custom alias:
- Validate format; insert with uniqueness check; on conflict → 409

## Data model
You can use SQL or a KV/NoSQL store; the access pattern is key lookup by shortCode.

Fields (generic):
```
shortCode (PK)
longUrl
userId (optional)
createdAt
expiryAt (optional)
isCustomAlias (bool)
status (active/expired)
```

Notes:
- Strong consistency on writes; read replicas for scale (SQL) or partitioned KV (NoSQL)
- Optional index on expiryAt for cleanup jobs

## Redirect flow (hot path)
1) Client → CDN
2) CDN cache hit → return 302
3) CDN miss → Redirect Service
4) Redis lookup: key shortCode → value longUrl(+expiry)
5) Cache miss → DB → populate Redis → return 302
6) Emit async event for analytics (non-blocking)

If Redis fails: fallback to DB; higher latency but keep availability.

## Caching strategy
### Redis (read-through/cache-aside)
- Key: shortCode; Value: longUrl (+ expiry)
- URLs are immutable → safe for aggressive caching
- Redis TTL aligned with link expiry when present
- Redis = performance layer, database = correctness

### CDN
- Cache 302 responses at edge
- Use Cache-Control: public, max-age=<ttl>
- TTL aligned with link expiry; no per-URL invalidation; accept small stale window

## Partitioning and geo
- Shard by hash(shortCode) for even distribution
- Geo-replication for read latency; avoid geo-sharding write path

## Analytics pipeline
- Redirect service publishes click events to a queue
- Consumers process asynchronously; best-effort delivery
- Redirect path does not depend on analytics availability

## Hot key handling
- CDN absorbs the majority of viral traffic
- Redis handles remaining hot reads
- App servers + network are primary bottlenecks; DB must stay out of hot path

## Consistency and failure handling
- Create API writes to primary DB; strong consistency
- Immediate redirect reads from cache or primary
- Cache miss + DB down → 503
- Unknown/expired URL → 404
- Duplicate key → DB uniqueness + retry
- Services stateless → horizontally scalable

## Trade-offs and improvements
- CDN for extremely hot URLs
- Bloom filter to reduce DB misses
- Rate limiting to prevent abuse
- URL expiration/TTL support + background cleanup