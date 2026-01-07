# Rate Limiter – System Design

---

## Why rate limiting exists
Rate limiting protects systems from:
- Resource starvation (CPU, DB connections, thread pools)
- Cascading failures in downstream services
- Abuse, bots, brute-force attacks
- Infrastructure cost spikes
- Unfair usage by a small set of users

Without rate limiting, a single noisy user can degrade or crash the system.

## Where to place a rate limiter
### Primary: API Gateway
- Centralized enforcement; stops bad traffic early
- One implementation for all services

### Other layers
- Load Balancer → coarse IP limits
- Application service → last line of defense

Note: Real systems use multiple layers (defense in depth).

## Stateless vs stateful
- Rate limiter service is stateless
- Rate limiting state is stored centrally (Redis)

Why:
- Horizontal scalability
- No sticky sessions
- Consistent limits across instances

## Algorithms (chosen + alternatives)
### Token Bucket (chosen)
- Tokens added at a fixed rate; bucket has max capacity
- Each request consumes one token; allows bursts
- Enforces steady long-term rate; O(1) check
- Simple to implement with Redis + Lua

### Sliding Window Counter (popular)
- Keep current and previous window counters
- Effective count = `current + previous × overlap_ratio`
- Smooths boundary spikes; more accurate than fixed window

### Others
- Fixed Window → boundary spike problem
- Sliding Window Log → accurate, but high memory/cleanup cost
- Leaky Bucket → constant drain, no bursts; poor UX under spikes

## Limits and time windows
Combine windows for good UX and predictable backend load:
- Per-second → burst protection
- Per-minute → abuse control (most APIs)

## Identification strategy
Never rely on a single identifier.

Primary: UserId
- IP alone is unreliable (NAT, office Wi‑Fi, mobile networks)
- Devices change, users don’t
- Maps cleanly to business rules (free vs paid)

Best practice: apply multiple limits in parallel
- UserId (primary)
- IP
- API key / device fingerprint

Reject if any applicable limit is exceeded.

## Hard vs soft limits
### Hard limits
- Login, OTP, payments → immediate rejection (HTTP 429)

### Soft limits
- Premium/trusted users; temporary spikes (events/sales)
- Add latency, grace quota, or throttle instead of blocking

## HTTP responses
### Allowed
Return headers:
- X-RateLimit-Limit
- X-RateLimit-Remaining
- X-RateLimit-Reset

### Rejected
- Status: 429 Too Many Requests
- Header: Retry-After
- Optional error body indicating rate limit exceeded

## Storage choice and Redis features
### Redis (preferred)
- In-memory, low latency, atomic ops, centralized state

### Features used
- INCR/INCRBY → atomic counters
- EXPIRE/TTL → automatic reset
- Hashes → tokens + timestamps
- Lua scripts → atomic check/update
- Pipelines → fewer network hops

Tip: Lua scripts are preferred over MULTI for hot paths.

## Concurrency and atomicity
Problem: multiple gateways race on the same counter and violate limits.

Solution:
- Redis is single-threaded per shard; Lua guarantees atomicity
- All gateways operate on the same Redis key

## Performance and reducing Redis load
### Performance
- O(1) per request
- Redis ops per request: ideal 1, acceptable 2; more hurts latency/scale

### Reducing Redis load
- Redis sharding (cluster) and proper key hashing
- Local short-circuiting for coarse cases
- Edge/CDN coarse limits to shed load
- Single-call Lua scripts
- TTL tuning to avoid hot keys

## Edge/CDN rate limiting and layered setup
### Benefits
- Lowest latency; sheds bad traffic before origin
- DDoS protection; reduced origin traffic

### Trade-offs
- Limited user context; less flexible rules
- Edge state is eventually consistent (not perfect globally)

### Layered production setup
1) CDN/Edge → IP/device-based, coarse limits, DDoS/bot protection
2) API Gateway → UserId/API key limits, regional Redis
3) Service-level limiter → critical endpoints only, strong consistency, fail-closed

Edge optimizes latency & cost; backend optimizes correctness.

## When CDN isn’t enough
- Login/OTP, payments, inventory/bidding, compliance-heavy APIs
- Require server-side limits with strong consistency; no fail-open

## Monitoring and alerts
### Metrics
- Allowed vs throttled requests; 429 rate
- Redis latency (p95/p99), errors/timeouts
- Fail-open/closed counts

### Alerts
- Spike in 429s; Redis latency threshold breach
- Redis unavailable; unexpected traffic drop

### Dashboards
- Traffic vs throttled; top blocked users/IPs
- Redis health alongside rate limiter metrics

## Architecture
```
Client
  ↓
API Gateway
  ↓
Rate Limiter Service
  ↓
Redis Cluster
```

## Data model
Key format:
```
rate:{userId}:{endpoint}
```
Stored values:
```
tokens_remaining
last_refill_timestamp
```

## Request flow
1) Request reaches API Gateway
2) Gateway calls Rate Limiter
3) Redis executes atomic logic:
   - Refill tokens
   - If token exists → allow
   - Else → reject
4) Response returned:
   - 200 OK or 429 Too Many Requests
   - Rate-limit headers

Concurrency is handled using atomic operations to avoid race conditions.

## Complexity and scaling
- Time: O(1) per request; Space: proportional to active users
- Redis Cluster with sharding; per-endpoint limits to avoid hot keys
- Deploy close to gateway; multi-region: enforce per-region; small inconsistencies acceptable

## Failure handling
If Redis is unavailable:
- Critical APIs (auth, payments): fail-closed
- Non-critical APIs (public reads): fail-open

Never let the rate limiter become the reason production is down.

## Trade-offs
- Favor latency and availability over perfect accuracy
- Best-effort consistency is sufficient
- Keep the system simple and fast

---

## Appendix: Redis Lua token bucket (minimal example)
Pseudocode structure for an atomic token bucket in Redis using Lua:

```
-- KEYS[1] = key (e.g., rate:{userId}:{endpoint})
-- ARGV = capacity, refill_per_ms, now_ms

local capacity = tonumber(ARGV[1])
local rate = tonumber(ARGV[2])       -- tokens per millisecond
local now = tonumber(ARGV[3])

local last = tonumber(redis.call('HGET', KEYS[1], 'last_refill') or now)
local tokens = tonumber(redis.call('HGET', KEYS[1], 'tokens') or capacity)

local elapsed = math.max(0, now - last)
tokens = math.min(capacity, tokens + (elapsed * rate))

local allowed = 0
if tokens >= 1 then
  tokens = tokens - 1
  allowed = 1
end

redis.call('HSET', KEYS[1], 'tokens', tokens)
redis.call('HSET', KEYS[1], 'last_refill', now)
redis.call('PEXPIRE', KEYS[1], 60000)  -- auto cleanup

return allowed  -- 1 = allow, 0 = reject
```

Note: Tune `capacity` and `rate` to match per-second/per-minute windows. Use `PEXPIRE` to avoid hot keys lingering.
