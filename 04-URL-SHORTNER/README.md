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
- Low latency for redirects (target p99 < 50–200 ms depending on scale)
- High availability for redirect service
- Strong consistency for URL creation and redirects
- Durability (URLs must not be lost)
- Horizontal scalability

### Out of Scope
- Abuse/malware detection
- QR codes
- Advanced user authentication

## Scale and capacity assumptions
Two representative scenarios:
- Moderate: ~100M new URLs/year → ~3–4 writes/sec (bursty), ~10B redirects/year → ~300–500 RPS avg; Read:Write ≈ 100–1000:1
- Peak: ~100M new URLs/day; Read:Write ≈ 100:1 → Writes ≈ 1.2k RPS; Reads ≈ 120k RPS; Storage ≈ ~100 GB/day (1 KB/URL) ≈ 36 TB/year

Alternative DAU view:
- DAU ≈ 100M; 0.1 URL created/user/day; ~10 redirects/URL/day → ~10M creates/day, ~100M redirects/day; Avg ≈ 1.1k RPS; Peak ≈ 10–12k RPS.

Redirect is the hot path in all scenarios.

## APIs
### Create Short URL
POST /shorten

Request body (JSON):
```json
{
  "originalUrl": "https://example.com",
  "customAlias": "myalias",
  "expirySeconds": 31536000,
  "userId": "123"
}
```

Responses:
- 201 Created → returns short URL
```json
{ "shortUrl": "https://sho.rt/myalias" }
```
- 400 Bad Request (invalid URL)
- 409 Conflict (alias already exists)

### Redirect
GET /{shortCode}

Responses:
- 301 Moved Permanently or 302 Found with Location: https://example.com
- 404 Not Found (unknown/expired)
- 503 Service Unavailable (cache miss + DB down)

## High-level architecture
- CDN (edge caching of 301/302)
- API Gateway (auth, rate-limit, routing)
- Create Service
- Redirect Service
- Key Generation Service (KGS)
- Redis (hot path cache)
- Primary Database (persistent store)
- Async Analytics Pipeline (Queue + Consumers)

## Short code generation
- Dedicated KGS with pre-allocated counter ranges per node to avoid bottlenecks
- Use monotonically increasing numeric IDs encoded with Base62
- 7 chars → 62^7 ≈ 3.5 trillion URLs (ample keyspace)
- Range leasing per instance (e.g., A: 1–10M, B: 10M–20M); on restart, lease a new unused range
- Database enforces unique constraint on short code; on rare collision, retry

Why not random?
- Random Base62 works at small scale; counters are simpler to reason about and avoid probabilistic collision handling at large scale

Custom alias:
- Validate format; insert with uniqueness check; on conflict → 409

## Data model
Use SQL or a KV/NoSQL store; access pattern is key lookup by shortCode.

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
- For NoSQL tables: partition key MUST be shortCode; use DB-level TTL for expired links

## Redirect flow (hot path)
1) Client → CDN
2) CDN cache hit → return 301/302
3) CDN miss → Redirect Service
4) Redis lookup: key shortCode → value longUrl(+expiry)
5) Cache miss → DB → populate Redis → return 301/302
6) Emit async event for analytics (non-blocking)

If Redis fails: fallback to DB; higher latency but remain available.

## Caching strategy
### Redis (read-through/cache-aside)
- Key: shortCode; Value: longUrl (+ expiry)
- URLs are immutable → safe for aggressive caching
- Redis TTL aligned with link expiry when present
- Redis = performance layer; DB = correctness

### CDN
- Cache 301/302 responses at edge
- Use Cache-Control: public, max-age=<ttl>
- TTL aligned with link expiry; no per-URL invalidation; accept small stale window

## Partitioning and geo
- Shard by hash(shortCode) for even distribution
- Geo-replication for read latency; avoid geo-sharding on the write path

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
- Cache down → DB still serves redirects; rate-limit if needed
- DB replica failure → read from other replicas
- ID generator crash → lease new range on restart

## Trade-offs and improvements
- CDN for extremely hot URLs
- Bloom filter to reduce DB misses
- Rate limiting to prevent abuse
- URL expiration/TTL support + background cleanup
- Write-through updates only if updates are allowed (usually URLs are immutable)

## Key takeaways
- Redirect is the hot path; creation is cold
- Cache absorbs traffic spikes; DB stays out of hot reads
- TTL controls data size automatically; accept small stale windows at the edge
- Strong consistency for redirects; eventual consistency acceptable for analytics
- Range-based ID allocation avoids bottlenecks and collisions