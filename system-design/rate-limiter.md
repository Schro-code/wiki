---
domain: system-design
tags: [system-design]
sources:
  - https://github.com/liquidslr/system-design-notes/blob/main/04.%20Rate%20Limiter/Readme.md
updated: 2026-09-14
---

# Rate Limiter

## What it is

A rate limiter controls how many requests a client (user, IP, service) can make in a given time window. Requests that exceed the limit are rejected with HTTP 429 or queued for later.

## What problem it solves

Without rate limiting, a single client can exhaust server resources — intentionally (DoS attack) or accidentally (buggy client in a loop). Rate limiting protects availability, reduces unnecessary infrastructure cost, and enforces fair usage across clients.

## Notes

### Placement options

| Option | Notes |
|---|---|
| Client-side | Unreliable — clients can bypass it |
| Server-side (in-app) | Full control; rate limiter lives inside the application code |
| Middleware / API Gateway | Preferred for microservices; a standalone service that sits in front of all apps |

**Standalone service vs. application-level code:**  
Application-level means the limiter runs inside each service. Standalone means a separate process (e.g., API Gateway, Kong, AWS API Gateway) intercepts all traffic before it reaches any service. Standalone is preferred at scale because it centralizes the logic and shares state across all instances.

### Throttle rules

A throttle rule defines *who* gets *how many* requests in *what window*. Rules are stored in a database and loaded into cache. Multiple rules can coexist:

```
free tier users   → 100 req/min
premium users     → 1000 req/min
/login endpoint   → 5 req/min per IP
any single IP     → 10,000 req/day
```

Each rule targets a different dimension (user tier, endpoint, IP). Rules are matched per request; the most specific one applies.

---

### Algorithms

#### 1. Token Bucket

A bucket holds tokens up to a max capacity. A refiller adds tokens at a fixed rate. Each request consumes one token. If the bucket is empty, the request is dropped.

```
Refiller: adds N tokens/sec
Bucket:   [ T T T T T ]  (max = bucket_size)
Request → takes 1 token → allowed
Request → bucket empty  → dropped
```

- Allows **bursts** up to bucket size (unused tokens accumulate).
- Parameters: `bucket_size` (burst capacity), `refill_rate`.
- Low memory: one counter per user in Redis.
- Used by most public APIs (Twitter, Stripe).

#### 2. Leaking Bucket

Requests enter a FIFO queue. The system drains the queue at a constant rate — like water leaking from a hole. If the queue is full, new requests are dropped.

```
Requests → [ queue ] → processed at fixed rate → servers
                ↑
           overflow = dropped
```

- **No bursts reach the server** — output is always constant.
- Downside: a burst fills the queue with old requests; newer ones are dropped even after the burst ends.
- Used by: `uber-go/ratelimit`.

**Key difference from Token Bucket:** Token Bucket lets bursts through immediately. Leaking Bucket smooths everything into a steady stream.

#### 3. Fixed Window Counter

Divide time into fixed windows (e.g., each minute). Each window has a counter. Increment on each request; drop if the counter exceeds the limit. Counter resets at window boundary.

```
00:00–01:00 → counter: 100/100  (limit hit)
01:00–02:00 → counter: 0        (reset)
```

- Simple and cheap: one counter per user per window in Redis.
- **Edge spike problem:** a burst at the end of window 1 and another at the start of window 2 can let through 2× the intended limit in a short span.

#### 4. Sliding Window Log

Store the timestamp of every request. On each new request:
1. Delete timestamps older than `now - window`.
2. Count remaining timestamps.
3. If count < limit → allow and record timestamp. Else → drop.

```
now=1:30, window=60s
Log: [0:31, 0:45, 1:02, 1:15, 1:28] → count=5 → allow
```

- **Perfectly accurate** — no edge spike.
- **Expensive:** stores one timestamp per request. High-traffic users consume a lot of memory.

#### 5. Sliding Window Counter

Hybrid of Fixed Window and Sliding Log. Uses two counters (current window and previous window) and weights the previous one by how far into the current window you are.

```
estimate = prev_count × (1 - elapsed%) + curr_count

Example:
  window=60s, limit=100
  prev window: 80 requests
  current window: 20s in (33%), 30 requests so far
  estimate = 80 × 0.67 + 30 = 83.6 → allow (< 100)
```

- Low memory: only 2 counters per user.
- Approximate (assumes even distribution in previous window) — accurate enough at scale.
- Best balance of memory, accuracy, and burst handling.

#### Algorithm comparison

| Algorithm | Accurate | Memory | Allows burst | Constant output |
|---|---|---|---|---|
| Token Bucket | Good | Low | Yes | No |
| Leaking Bucket | Good | Low | No (smoothed) | Yes |
| Fixed Window Counter | Poor (edge spike) | Very low | Yes | No |
| Sliding Window Log | Exact | High | No | No |
| Sliding Window Counter | ~Exact | Low | Partial | No |

---

### High-level architecture

```
[Client] → [Rate Limiter Middleware] → [API Servers]
                    ↕                       
                 [Redis]              (counters per user/IP)
                    ↑
               [Cache]               (cached rules, fast read)
                    ↑
              [Workers] ←── polls ──→ [Rules DB]
```

**Request flow (allowed):**
1. Client sends request to middleware.
2. Middleware reads rules from **Cache** (fast, in-memory).
3. Middleware runs `INCR counter:user_X` in **Redis** → gets new count.
4. Count ≤ limit → forward to API Server → response to client.

**Request flow (throttled):**
1. Count > limit → return **429 Too Many Requests** to client.
2. Throttled request goes to:
   - **Option 1 — Drop:** discarded immediately.
   - **Option 2 — Queue:** sent to a message queue for deferred processing (useful for non-time-sensitive work like email sends).

**Why Redis for counters?**  
In-memory → microsecond reads/writes. Atomic `INCR` prevents race conditions across multiple middleware instances. TTL on each key auto-resets the counter when the window expires.

**Why Cache for rules?**  
Rules change rarely (weekly, not per-request). Reading the rules DB on every request would add latency and overload it. Workers poll the DB periodically and write fresh rules to Cache. Middleware reads only from Cache — a tolerable ~60s staleness.

---

### Distributed environments

**Race condition:** two middleware instances read the same counter simultaneously and both see 99 (below limit 100) and both allow the request — counter ends at 101.

**Solutions:**
- **Lua scripts in Redis:** execute the read-check-increment as a single atomic operation.
- **Redis sorted sets:** store timestamps for sliding window log atomically.
- **Locks:** pessimistic, adds latency — use only if necessary.
- **Centralized Redis cluster:** all middleware instances share one Redis, so counts are consistent.

**Performance at scale:**
- Multi-region: deploy Redis close to each data center to reduce cross-region latency.
- Eventual consistency: allow slight over-counting across regions in exchange for lower latency. Acceptable for most use cases.

### Headers to inform clients

When throttled, return these headers so clients know when to retry:

```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1725321600   ← Unix timestamp when the window resets
Retry-After: 47                 ← seconds until retry is safe
```

## Relations

- [[redis]] — used as the counter store and optionally the rules cache
- [[distributed-transactions]] — race conditions in distributed rate limiters share the same root problem as distributed writes
- [[rest-api]] — 429 Too Many Requests is the standard HTTP response for a throttled request

## Tags

#system-design
