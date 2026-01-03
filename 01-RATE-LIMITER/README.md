# Rate Limiter – System Design

## 1. Why Rate Limiting Exists
Rate limiting protects systems from:
- Resource starvation (CPU, DB connections, thread pools)
- Cascading failures in downstream services
- Abuse, bots, brute-force attacks
- Infrastructure cost explosions
- Unfair usage by a small set of users

Without rate limiting, a single noisy user can degrade or crash the system.

## 2. Where to Place a Rate Limiter
### Primary Choice: API Gateway
- Centralized enforcement
- Stops bad traffic early
- One implementation for all services

### Other Layers
- Load Balancer → coarse IP-based limits
- Application Service → last line of defense

Note: Real systems use multiple layers (defense in depth).

## 3. Stateful vs Stateless
- Rate limiter service is stateless
- Rate limiting logic is stateful
- State is stored in Redis

Why:
- Horizontal scalability
- No sticky sessions
- Consistent limits across instances

## 4. Rate Limiting Algorithms
### Token Bucket
- Tokens added at a fixed rate
- Bucket has maximum capacity
- Each request consumes one token
- Allows bursts
- Best choice for public APIs

### Leaky Bucket
- Requests are queued
- Processed at a constant rate
- Does not allow bursts
- Poor UX during spikes

### Fixed Window
- Counter per fixed time window
- Easy to implement
- Suffers from boundary burst problem

### Sliding Window Log
- Store timestamp of every request
- Most accurate
- High memory and cleanup cost

### Sliding Window Counter (Most Used)
- Store current window count
- Store previous window count
- Use weighted calculation to smooth spikes

Effective count: `current_count + (previous_count × overlap_ratio)`

## 5. Storage Choice
### Redis (Preferred)
- In-memory
- Low latency
- Atomic operations
- Centralized state

### Why Not Others
- Local memory multiplies limits across instances
- Databases are slow and expensive

## 6. Redis Features Used
- INCR / INCRBY → atomic counters
- EXPIRE / TTL → automatic reset
- Hashes → store tokens and timestamps
- Lua scripts → atomic check and update
- Pipelines → reduce network hops

Tip: Lua scripts are preferred over MULTI for hot paths.

## 7. Concurrency and Consistency
Problem:
- Multiple gateways read the same counter
- All think request is allowed
- Limit gets violated

Solution:
- Redis executes commands single-threaded per shard
- Lua scripts guarantee atomicity
- All gateways operate on the same Redis key

## 8. Redis Failure Strategy
### Fail Open
Use for:
- Public read APIs
- Browse, feed, search

Reason:
- User experience is more important than strict enforcement

### Fail Closed
Use for:
- Payments
- Login and OTP
- Admin APIs

Reason:
- Security and money are more important than availability

## 9. Identification Strategy
Never rely on a single identifier.

Why IP Alone Is Dangerous
- Shared NAT or corporate Wi-Fi
- Mobile IP rotation

Best Practice
- Apply multiple limits in parallel:
  - IP
  - User ID
  - API key
- Reject the request if any applicable limit is exceeded.

## 10. Unauthenticated Users
Before login, context is limited.

Use layered defense:
- IP-based limits
- Device fingerprint
- Endpoint-specific limits (login and OTP are stricter)

This approach is best-effort, not perfect.

## 11. Hard vs Soft Limits
### Hard Limits
Used for:
- Login
- OTP
- Payments

Behavior:
- Immediate rejection
- HTTP 429

### Soft Limits
Used for:
- Events
- Sales
- Expected spikes

Behavior:
- Added latency
- Grace quota
- Gradual throttling

## 12. HTTP Responses
### Successful Requests
Return headers:
- X-RateLimit-Limit
- X-RateLimit-Remaining
- X-RateLimit-Reset

### Rejected Requests
- Status: 429 Too Many Requests
- Header: Retry-After
- Optional error body indicating rate limit exceeded

## 13. Performance
### Time Complexity
- O(1) per request

### Redis Operations per Request
- Ideal: 1
- Acceptable: 2
- More than 2 causes latency and scaling issues

## 14. Reducing Redis Load
- Redis sharding
- Local in-memory short-circuiting
- Coarse rate limiting at the edge
- Lua scripts to keep single Redis call
- TTL tuning to avoid hot keys

## 15. Edge / CDN Rate Limiting
### Benefits
- Lower latency
- DDoS protection
- Reduced origin traffic

### Trade-offs
- Limited user context
- Less flexible rules

Note: Backend rate limiting remains the source of truth.

## 16. Monitoring and Alerts
### Metrics
- Allowed vs throttled requests
- 429 rate
- Redis latency (p95 and p99)
- Redis errors and timeouts
- Fail-open and fail-closed usage

### Alerts
- Sudden spike in 429 responses
- Redis latency threshold breach
- Redis unavailable
- Unexpected traffic drop

### Dashboards
- Traffic vs throttled
- Top blocked users and IPs
- Redis health alongside rate limiter metrics

## Interview One-Liners
- Rate limiting is a protective layer, not business logic
- Lua scripts provide atomicity with fewer network hops
- Edge limits are coarse, backend limits are authoritative
- Fail open for UX, fail closed for money and security
- An unobservable rate limiter is a hidden SPOF
---

## Design Reference

### Problem Statement
Design a rate limiter to protect backend services from abuse while maintaining low latency for legitimate users.

### Requirements
#### Functional
- Limit requests per identity (userId / IP / API key)
- Support different limits per endpoint (e.g., login stricter than search)
- Reject excess requests early using HTTP 429 before hitting backend

#### Non-Functional
- Low latency and high throughput (O(1) check, in-memory)
- Horizontally scalable and highly available
- Best-effort consistency (small inaccuracies acceptable)

### Algorithm Choice
Token Bucket Algorithm

Chosen because:
- Allows burst traffic
- Simple state management
- Constant-time checks

Rejected alternatives:
- Fixed Window – burst at boundaries
- Sliding Window – higher memory and compute cost

### High-Level Architecture
```
Client
	↓
API Gateway
	↓
Rate Limiter Service
	↓
Redis Cluster
```
Redis is used because it is in-memory, fast, and supports atomic operations.

### Data Model
Key format:
```
rate:{userId}:{endpoint}
```
Stored values:
```
tokens_remaining
last_refill_timestamp
```

### Request Flow
1. Request reaches API Gateway
2. Gateway calls Rate Limiter
3. Redis executes atomic logic:
	 - Refill tokens
	 - If token exists → allow
	 - Else → reject
4. Response returned:
	 - 200 OK or 429 Too Many Requests
	 - Rate-limit headers

Concurrency is handled using atomic operations to avoid race conditions.

### Complexity
- Time Complexity: O(1) per request
- Space Complexity: proportional to active users

### Scaling Strategy
- Redis Cluster with sharding
- Per-endpoint limits to avoid hot keys
- Rate limiter deployed close to gateway
- Multi-Region: each region enforces limits independently; no global synchronization; minor inconsistencies are acceptable

### Failure Handling
If Redis is unavailable:
- Critical APIs (auth, payments): fail-closed
- Non-critical APIs (public reads): fail-open

The rate limiter must never become the reason production is down.

### Trade-offs
- Latency and availability over perfect accuracy
- Best-effort consistency is sufficient
- Keep the system simple and fast

### Summary
This design provides a fast, scalable, and highly available rate limiter that protects backend systems while tolerating small inaccuracies — the correct tradeoff at scale.