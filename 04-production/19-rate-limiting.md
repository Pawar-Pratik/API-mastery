# Lesson 19 — Rate limiting, quotas & fairness

> **Why this lesson exists:** rate limiting is where "I know the algorithms" meets "I've operated a real API." Interviewers ask for the algorithm *and* the distributed implementation *and* the client contract, because those three together prove you've shipped it. It's also the mechanism that decides whether one badly-written customer script can degrade service for everyone else.

**Time:** ~80 minutes · **Prereq:** Lessons 10, 18

---

## 1. The idea in one sentence

> **Rate limiting is not about blocking abuse — it's about **fairness under contention**: guaranteeing that no single caller can consume capacity that other callers need, and telling every caller precisely what their budget is so they never have to guess.**

The second half is the part most implementations skip, and it's what makes the difference between a rate limiter clients cooperate with and one they fight.

---

## 2. Three different things people call "rate limiting"

Distinguishing these is a quick credibility win, because they live in different places and use different mechanisms.

| Concept | Question | Window | Where it lives |
|---|---|---|---|
| **Rate limit** | How many requests **per second/minute**? | Short, rolling | Gateway or app, Redis-backed |
| **Quota** | How many requests/units **per month**? | Long, billing-aligned | Application + billing system |
| **Concurrency limit** | How many **simultaneous** in flight? | Instantaneous | Semaphore / bulkhead ([Lesson 18](18-reliability-and-idempotency.md)) |

And a fourth that isn't rate limiting at all: **DDoS protection** — volumetric attacks are absorbed at the network/CDN edge (Cloudflare, AWS Shield), because by the time traffic reaches your app it's too late to be cheap. **Rate limiting protects fairness and cost; DDoS protection protects availability.** Saying that sentence prevents a whole confused conversation.

---

## 3. The five algorithms

You must know all five, their failure modes, and which you'd pick.

### 1. Fixed window counter
Count requests per fixed clock window; reset at the boundary.

```
Limit: 100/min
[10:00:00 – 10:00:59]  count → 100 allowed, rest 429
[10:01:00 – 10:01:59]  count resets to 0
```
```
INCR key; EXPIRE key 60 (on first increment)
```

| Pros | Cons |
|---|---|
| Trivial: one counter, one `INCR` | **The boundary burst**: 100 requests at 10:00:59 + 100 at 10:01:00 = **200 in one second** |
| Tiny memory (one integer per key) | Bursty load right after each reset |

The boundary burst means **your effective limit is 2× the configured one** over the worst-case window. Fine for coarse protection; not fine when you sized capacity from the number.

### 2. Sliding window log
Store a timestamp per request; count those within the last N seconds.

```
ZADD key <now> <uuid>; ZREMRANGEBYSCORE key 0 <now-60s>; ZCARD key
```

| Pros | Cons |
|---|---|
| **Perfectly accurate** | **Memory scales with request count** — 10,000 req/min per key is 10,000 entries |
| No boundary effects | Expensive to maintain at scale |

Correct, and usually too expensive. Reach for it only on low-volume, high-value endpoints (login attempts) where accuracy matters and volume doesn't.

### 3. Sliding window counter (the practical compromise)
Weight the previous window's count by how much of it still overlaps the rolling window.

```
now = 10:00:45, window = 60s
current window [10:00:00–]  count = 40
previous window [09:59:00–] count = 90
overlap of previous still in view = 15/60 = 25%
estimate = 40 + 90 × 0.25 = 62.5  → 62
```

| Pros | Cons |
|---|---|
| ~O(1) memory (two counters), no boundary burst | An *approximation* — assumes the previous window's traffic was uniform |
| Good accuracy in practice | Slightly harder to explain to a customer |

**This is what Cloudflare uses**, and it's a good default for high-volume endpoints.

### 4. Token bucket — **my default recommendation**

A bucket holds up to `capacity` tokens and refills at `rate` tokens/second. Each request takes one token; empty bucket → reject.

```
capacity = 100 (burst), refill = 10 tokens/sec (sustained)
→ a client can burst 100 immediately, then settle to 10/s
```

| Pros | Cons |
|---|---|
| **Bursts are allowed on purpose** — which matches real client behaviour | Two parameters to explain |
| Smooth sustained rate | Needs an atomic read-modify-write |
| Naturally expresses "10/s sustained, 100 burst" | |

**Why bursts matter, and why this is the right default:** real clients are bursty. A dashboard loading fires 8 parallel requests; a nightly sync fires 200 then sleeps. A strict 10/s limiter rejects a legitimate page load; a token bucket absorbs it and still caps the sustained rate. **You are limiting sustained cost, not instantaneous concurrency** — and token bucket expresses exactly that. Used by AWS, Stripe and Google.

### 5. Leaky bucket (queue)
Requests enter a queue that drains at a fixed rate; a full queue rejects. Equivalent to token bucket for *rejection* purposes, but it **delays** rather than rejecting.

Only use it when smoothing genuinely helps (feeding a fragile downstream at a fixed rate). For an HTTP API, delaying a request while the client's own timeout ticks down is usually worse than a fast `429` — you burn their patience *and* hold your resources.

### The choice

```
Login / password reset / OTP           → sliding window log (accuracy matters, volume is low)
Normal API endpoints                   → token bucket (bursts + sustained rate)
Very high volume / edge                → sliding window counter (cheap, no boundary burst)
Feeding a fragile downstream           → leaky bucket
Quick internal protection              → fixed window (accept the 2× burst)
```

---

## 4. Distributed implementation

Two instances each allowing 100/min means the client gets 200/min. So the counter must be **shared and atomic** — and "atomic" is the word that matters, because `GET` then `SET` from two instances is a race that lets both through.

### Token bucket in Redis, atomically, with one Lua script

```lua
-- token_bucket.lua — KEYS[1] = bucket key
-- ARGV: 1=capacity  2=refillPerSec  3=nowMs  4=cost  5=ttlSec
local capacity   = tonumber(ARGV[1])
local refill     = tonumber(ARGV[2])
local now        = tonumber(ARGV[3])
local cost       = tonumber(ARGV[4])
local ttl        = tonumber(ARGV[5])

local bucket = redis.call('HMGET', KEYS[1], 'tokens', 'ts')
local tokens = tonumber(bucket[1])
local ts     = tonumber(bucket[2])

if tokens == nil then
  tokens = capacity          -- first sight of this key: full bucket
  ts = now
end

-- Refill for the elapsed time, capped at capacity.
local elapsed = math.max(0, now - ts) / 1000.0
tokens = math.min(capacity, tokens + elapsed * refill)

local allowed = 0
if tokens >= cost then
  tokens = tokens - cost
  allowed = 1
end

redis.call('HMSET', KEYS[1], 'tokens', tokens, 'ts', now)
redis.call('EXPIRE', KEYS[1], ttl)

-- retryAfter: seconds until enough tokens exist for this cost
local retryAfter = 0
if allowed == 0 then
  retryAfter = math.ceil((cost - tokens) / refill)
end

return { allowed, math.floor(tokens), retryAfter }
```

```ts
import { readFileSync } from "node:fs";
const SCRIPT = readFileSync("token_bucket.lua", "utf8");

export type LimitResult = { allowed: boolean; remaining: number; retryAfter: number };

export async function checkLimit(
  key: string,
  { capacity, refillPerSec, cost = 1 }: { capacity: number; refillPerSec: number; cost?: number },
): Promise<LimitResult> {
  const ttl = Math.ceil(capacity / refillPerSec) + 60;
  const [allowed, remaining, retryAfter] = await redis.eval(
    SCRIPT, 1, key, capacity, refillPerSec, Date.now(), cost, ttl,
  ) as [number, number, number];
  return { allowed: allowed === 1, remaining, retryAfter };
}
```

**Why Lua and not `MULTI`/`WATCH`:** the read-refill-write sequence must be atomic, and a Lua script runs as a single atomic operation on the Redis server — no round trips, no optimistic-retry loop. **One network hop, one atomic decision.** This snippet is a very good thing to be able to reproduce in an interview.

### The two failure modes of a distributed limiter

**1. Redis is down.** You must decide **fail-open or fail-closed**, deliberately:
```ts
try {
  const r = await checkLimit(key, cfg);
  if (!r.allowed) return reject(r);
} catch (e) {
  metrics.inc("ratelimit.backend_error");
  logger.error({ err: e }, "rate limiter unavailable");
  // FAIL OPEN: availability over protection. Note this is a deliberate choice.
  // For login endpoints specifically, consider failing closed instead.
}
```
**Fail open** for normal endpoints — a rate limiter outage shouldn't be an API outage. **Fail closed** (or fall back to a strict local limiter) for authentication endpoints, where failing open means unlimited brute force. State this explicitly; interviewers like that you noticed there's a choice.

**2. The latency you add.** A Redis round trip on every request is ~0.3–1ms — acceptable. If it isn't, the standard optimisation is a **local token bucket per instance sized to `globalLimit / instanceCount`**, with periodic reconciliation. Approximate, much faster, and what large-scale systems actually do.

---

## 5. What to key on (and the mistakes)

```ts
function limitKey(req: Request): { key: string; cfg: LimitConfig } {
  const p = req.principal;

  // 1. Authenticated: key on the STABLE identity, never the IP.
  if (p?.kind === "api_key") return { key: `rl:key:${p.keyId}`,  cfg: tierFor(p.plan) };
  if (p?.kind === "user")    return { key: `rl:user:${p.userId}`, cfg: TIERS.dashboard };
  if (p?.kind === "oauth")   return { key: `rl:app:${p.appId}:${p.merchantId}`, cfg: TIERS.platform };

  // 2. Unauthenticated: IP is all you have — and it's weak. Be strict.
  return { key: `rl:ip:${clientIp(req)}`, cfg: TIERS.anonymous };
}
```

| Key on | When | Weakness |
|---|---|---|
| **API key / user / app ID** | Authenticated traffic. **Preferred** | Requires authentication to have run |
| **IP** | Anonymous traffic only | **Shared IPs**: one corporate NAT or mobile carrier is thousands of users. Also trivially rotated with a botnet or cloud IPs |
| **IP + endpoint** | Login, signup, reset | Still shared-IP-limited |
| **Account/email** | Login attempts | **Enables lockout-as-DoS** against a specific user — so combine with IP, and never lock permanently |
| **Tenant** | Multi-tenant fairness | The right level for cost control |

Two mistakes worth calling out:

1. **Rate limiting *after* authentication.** Verifying a JWT signature or bcrypt-comparing a password is expensive; an attacker sending 10,000 bad credentials shouldn't get 10,000 verifications. **Order: a coarse IP limit → authenticate → a precise identity limit.** (This is the middleware ordering from [Lesson 02](../01-foundations/02-journey-of-a-request.md), with the nuance that you need limiters on *both* sides.)
2. **Trusting `X-Forwarded-For` blindly.** `req.ip` with `trust proxy: true` lets a client forge their IP and get unlimited quota. Set `trust proxy` to the exact number of proxy hops, and take the *rightmost* untrusted value.

### Cost-weighted limiting
Not all requests cost the same. Charge accordingly:
```ts
const COST = {
  "GET  /v1/payments/:id":       1,
  "GET  /v1/payments":           5,     // a query with filters and sorting
  "POST /v1/payments":          10,     // writes, plus downstream calls
  "POST /v1/reports":          100,     // expensive aggregate
  "POST /v1/payments/bulk":      1,     // ×N items — meter per ITEM (Lesson 09)
};
```
**Bulk endpoints must be metered per item**, or a client sends 100-item batches and gets 100× their quota. That's the same rate-limit-bypass note from Lesson 09, and it's a favourite follow-up question.

---

## 6. The client contract — the part that makes a limiter usable

```http
HTTP/1.1 200 OK
RateLimit-Limit: 100
RateLimit-Remaining: 87
RateLimit-Reset: 42                 ← seconds until the budget refills
```
```http
HTTP/1.1 429 Too Many Requests
Retry-After: 12                     ← REQUIRED. Seconds (or an HTTP-date)
RateLimit-Limit: 100
RateLimit-Remaining: 0
RateLimit-Reset: 12
Content-Type: application/problem+json

{
  "type": "https://docs.ledger.dev/errors/rate-limit-exceeded",
  "title": "Too Many Requests",
  "status": 429,
  "code": "rate_limit_exceeded",
  "detail": "You have exceeded the rate limit of 100 requests per minute for this API key.",
  "retry_after": 12,
  "limit": 100,
  "scope": "api_key",
  "request_id": "01HQ8ZK3M"
}
```

| Rule | Why |
|---|---|
| **Send `RateLimit-*` on *every* response, not just 429s** | A client that can see its budget self-throttles and never hits your limiter. This is the single highest-value thing you can do |
| **`Retry-After` is mandatory on 429** | Without it clients guess, and they guess aggressively |
| **Say *which* limit was hit (`scope`)** | Per-key? Per-IP? Per-endpoint? The fix differs |
| **Expose the headers via `Access-Control-Expose-Headers`** | Otherwise browser clients can't read them at all ([Lesson 04](../01-foundations/04-headers-and-payloads.md)) |
| **Don't count 429s toward the limit** | Otherwise a retrying client can never escape |
| **Document limits publicly, per tier** | Integrators design around published numbers |
| **Never leak the limiter's internals** | Don't expose exact bucket state; it aids evasion |

> The IETF draft standardises `RateLimit-Limit`/`Remaining`/`Reset`; GitHub and Stripe use `X-RateLimit-*`. Either is fine — **pick one, be consistent, document it.** Emitting both is also acceptable during a migration.

---

## 7. Tiers, quotas and fairness

### Tiers
```ts
export const TIERS = {
  anonymous: { capacity: 10,    refillPerSec: 0.16 },  // ~10/min, small burst
  free:      { capacity: 100,   refillPerSec: 1.6  },  // ~100/min
  startup:   { capacity: 1_000, refillPerSec: 16   },  // ~1,000/min
  business:  { capacity: 5_000, refillPerSec: 83   },
  enterprise:{ capacity: 20_000,refillPerSec: 333  },
  dashboard: { capacity: 300,   refillPerSec: 5    },  // first-party UI: bursty, low sustained
} as const;
```

### Layered limits (defence in depth)
Real systems apply several at once, and the response must say which one fired:
```
per API key      → 1,000/min       (the customer's contract)
per merchant     → 5,000/min       (across all their keys)
per endpoint     → POST /reports: 10/min   (protect an expensive path)
per IP           → 10,000/min      (crude abuse ceiling)
global           → 100,000/min     (protect yourself; sheds load)
```

### Fairness: the noisy-neighbour problem
Per-tenant limits stop one tenant from *exceeding their share*, but they don't stop tenant A from filling a shared queue while tenant B waits. Techniques worth naming:
- **Per-tenant concurrency caps** (bulkheads) in addition to rate limits.
- **Fair queueing** — round-robin across tenants rather than FIFO, so a tenant with 10,000 queued jobs doesn't starve one with 2.
- **Priority classes** — interactive dashboard traffic ahead of batch exports.
- **Shuffle sharding** — assign each tenant a random subset of workers, so a single abusive tenant degrades only a small, overlapping fraction of others. (AWS's technique; a great term to know.)

### Quotas — a different mechanism
Monthly quotas are billing, not protection: they're enforced against a durable counter (not Redis alone), need to survive restarts, must be reconciled with the billing system, and require **warning thresholds** (80%, 100%) so customers aren't surprised. Business decision to make explicitly: at 100%, do you hard-block (`403 quota_exceeded`) or allow overage and bill for it? Both are defensible; silently doing neither is not.

---

## 8. Production rules

| Rule | Why |
|---|---|
| **Token bucket by default; sliding-window log for auth endpoints** | Bursts are legitimate; auth accuracy matters more than cost |
| **Atomic counter updates (Redis + Lua), never read-then-write** | Two instances racing lets both through |
| **Key on identity when authenticated; IP only for anonymous** | IPs are shared and rotatable |
| **A coarse limit before auth, a precise one after** | Don't spend crypto on attackers; don't limit real users by IP |
| **`trust proxy` set to an exact hop count** | Otherwise `X-Forwarded-For` is forgeable |
| **Cost-weight expensive endpoints; meter bulk per item** | Otherwise bulk is a quota bypass |
| **`RateLimit-*` on every response; `Retry-After` on every 429** | Lets clients self-throttle instead of being blocked |
| **Don't count 429s against the limit** | Or clients can never recover |
| **Decide fail-open vs fail-closed explicitly, per endpoint class** | A limiter outage shouldn't be an API outage — except for login |
| **Layer limits (key/tenant/endpoint/IP/global) and report which fired** | Different causes need different client fixes |
| **Per-tenant concurrency caps in addition to rate limits** | Rate limits alone don't prevent queue starvation |
| **Alert on sustained 429 rates per customer** | A customer hitting limits constantly is a churn or support signal, not a win |
| **Never permanently lock an account on failed logins** | It's a DoS someone can trigger against your users |

---

## 9. Interview traps

**Q1. "Implement rate limiting for an API."**
Structure: (1) pick token bucket and say why — bursts are legitimate; (2) key on API key/user, IP only for anonymous; (3) implement atomically in Redis with Lua, and explain *why* Lua; (4) the client contract — `RateLimit-*` on every response, `Retry-After` on 429; (5) layered limits and cost weighting. Covering the client contract unprompted is what marks you out.

**Q2. "Compare fixed window, sliding window and token bucket."**
Fixed: one counter, cheap, **2× boundary burst**. Sliding log: exact, memory scales with traffic. Sliding counter: two counters, approximate, no boundary burst (Cloudflare's choice). Token bucket: burst + sustained in two parameters, O(1) memory (AWS/Stripe's choice).

**Q3. "You have 10 API instances. How do you enforce one global limit?"**
Shared atomic state — Redis with a Lua script. Then the follow-up you should pre-empt: *"if the Redis hop is too expensive, use per-instance local buckets sized to `global/N` with periodic reconciliation, and accept the approximation."*

**Q4. "What if Redis goes down?"**
A deliberate choice, not an accident: **fail open** for normal endpoints (a limiter outage shouldn't be an API outage) with an alert; **fail closed** or fall back to a local limiter for authentication endpoints, where failing open means unlimited brute force.

**Q5. "Why not rate limit by IP?"**
Shared IPs (corporate NAT, mobile carriers, university networks) mean thousands of users share a budget; and IPs are cheap to rotate via botnets or cloud providers. Use IP only for anonymous traffic, and prefer a stable identity otherwise.

**Q6. "How do you protect a login endpoint?"**
Per-IP **and** per-account limits, exponential backoff on failures, CAPTCHA or proof-of-work after N failures, no permanent lockout (that's a DoS vector), constant-time comparison so there's no timing oracle, and — the part people miss — **a global anomaly signal**, because credential stuffing uses one attempt per account across thousands of accounts and stays under every per-account limit.

**Q7. "Client says they're rate limited but they're within their limit."**
Debug the layers: which limit fired? Their key's, their tenant's (shared across their several keys), an endpoint-specific one, a shared-IP limit, or a global shed? **This is exactly why the 429 body should name the `scope`** — without it this conversation takes a day.

**Q8. "How do you rate limit a bulk endpoint?"**
Per item, not per request. Otherwise 100-item batches multiply the quota by 100.

**Q9. "Rate limiting vs DDoS protection?"**
Different layers, different goals. Rate limiting is application-layer fairness and cost control, keyed on identity. DDoS protection is network/edge-layer volumetric absorption (CDN, Anycast, SYN cookies) — by the time a volumetric flood reaches your app, defending it is already too expensive.

**Q10. "How do you keep one tenant from degrading others even within their limit?"**
Rate limits alone don't solve it. Add per-tenant **concurrency** caps, fair queueing (round-robin across tenants, not FIFO), priority classes for interactive vs batch, and **shuffle sharding** to bound the blast radius of a single bad tenant.

---

## 10. Build & break

### Build — the limiter, end to end
1. `token_bucket.lua` and `checkLimit` from §4.
2. Middleware that: resolves the key (§5), applies **layered** limits (key → tenant → endpoint → global), sets `RateLimit-*` on every response, and returns the §6 problem+json with `scope` on rejection.
3. A cost table, with bulk metered per item.
4. Deliberate fail-open, with a metric and a log line — and fail-closed for `/auth/*`.

Tests:
```ts
test("allows a burst up to capacity, then 429s", ...);
test("refills over time at the configured rate", ...);
test("429 includes Retry-After and it is accurate ±1s", ...);
test("429s do not consume tokens", ...);
test("two instances share one budget", ...);                 // two clients, one Redis
test("concurrent requests never exceed capacity", async () => {
  const results = await Promise.all(Array.from({ length: 500 }, () => call()));
  expect(results.filter(r => r.status === 200)).toHaveLength(CAPACITY);   // exactly, not "about"
});
test("bulk of 50 items consumes 50 tokens", ...);
test("fails open when Redis is unreachable, and emits a metric", ...);
test("auth endpoints fail closed when Redis is unreachable", ...);
test("forged X-Forwarded-For does not grant a fresh budget", ...);
```
That concurrency test is the one that catches a non-atomic implementation. Run it 20 times.

### Break — five experiments
1. **The boundary burst.** Implement fixed-window 100/min and fire 100 requests at `:59.9` and 100 at `:00.1`. Confirm 200 got through in ~200ms. Switch to token bucket and repeat.
2. **The race.** Implement token bucket with `GET` + `SET` in application code instead of Lua. Fire 500 concurrent requests against a capacity of 100 and count successes — it'll be well over 100. Switch to the Lua script; it'll be exactly 100.
3. **Forge your IP.** With `trust proxy: true`, send `X-Forwarded-For: 1.2.3.<random>` on each request and observe unlimited quota. Set `trust proxy: 1` and watch it stop.
4. **Bypass via bulk.** Meter per request, then send 100-item batches and confirm you achieved 100× throughput. Meter per item.
5. **Kill Redis mid-load.** Watch your chosen failure mode actually happen. Then flip it and watch the other. **Decide which you want *before* production decides for you.**

### Explain out loud (2 minutes)
1. Rate limit vs quota vs concurrency vs DDoS protection.
2. The five algorithms, with each one's failure mode.
3. Why the Redis update must be a Lua script.
4. The client contract, and why `RateLimit-*` on every response matters most.
5. Fail-open vs fail-closed, and where each belongs.

---

## What's next

Your API is fast, resilient and fair. The last production lesson is the one that determines whether you can *fix* it at 3am: observability — correlation IDs, structured logs, the four metrics that matter, distributed tracing, and a repeatable path from a symptom to a root cause.

Next → **[Lesson 20: Observability & debugging APIs](20-observability.md)**
