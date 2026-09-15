# Lesson 17 — Caching & HTTP performance

> **Why this lesson exists:** caching is the largest performance lever HTTP gives you, and most APIs decline to use it — they ship no `Cache-Control`, no `ETag`, and then optimise the database. Meanwhile the fastest possible response is the one you never send. This lesson also settles the `no-cache` vs `no-store` question, which is asked constantly and answered wrongly almost as often.

**Time:** ~85 minutes · **Prereq:** Lessons 03, 04

---

## 1. The idea in one sentence

> **Every response you send is a claim about how long it stays true — and if you don't state that claim explicitly, every cache between you and the client will guess, usually wrongly and sometimes dangerously.**

---

## 2. The cache hierarchy: where a response can live

```
Browser memory cache          ~0 ms      per-tab, volatile
Browser disk cache            ~1 ms      per-user, survives restart
Service worker (Cache API)    ~1 ms      programmable, offline-capable
────────── the network begins here ──────────
CDN edge (Cloudflare/Fastly)  ~10 ms     SHARED across all users  ← the big win
Reverse proxy (nginx/Varnish) ~1 ms      shared, in your infra
API gateway cache             ~1 ms      shared
────────── your app begins here ──────────
Application cache (Redis)     ~1 ms      shared, you control invalidation
In-process cache (LRU Map)    ~0.01 ms   per-instance, cheapest, hardest to invalidate
Database buffer / query cache ~1 ms      opaque to you
```

The critical distinction, because it determines what you're *allowed* to cache:

| | **Private cache** | **Shared cache** |
|---|---|---|
| Where | Browser, service worker | CDN, proxy, gateway |
| Serves | One user | **Everyone** |
| May hold user-specific data? | Yes | **No — that's a data breach** |
| Directive | `Cache-Control: private` | `Cache-Control: public` |

> **The rule that prevents the worst caching bug:** if a response depends on *who* is asking, it must be `private` (or `no-store`), and any header it varies on must be in `Vary`. Get this wrong and a CDN serves merchant A's payments to merchant B. This has happened to real companies, publicly.

---

## 3. `Cache-Control`, directive by directive

This is the header. Learn each directive precisely.

### Response directives

| Directive | Precise meaning |
|---|---|
| `max-age=600` | Fresh for 600s. After that, **stale** (not deleted — see `stale-while-revalidate`) |
| `s-maxage=600` | Same, but **only for shared caches**. Overrides `max-age` there — so you can tell the CDN 600s and the browser 0s |
| `public` | Any cache may store it, even with an `Authorization` header present |
| `private` | **Only** the user's own browser. CDNs must not store it |
| `no-cache` | **Store it, but revalidate before every use.** *Not* "don't cache" |
| `no-store` | **Never write it to any cache, anywhere.** For genuinely sensitive responses |
| `must-revalidate` | Once stale, you may **not** serve it — revalidate or fail |
| `proxy-revalidate` | `must-revalidate` for shared caches only |
| `immutable` | This URL's content will never change. Don't even revalidate on reload |
| `stale-while-revalidate=60` | Serve stale for up to 60s **while** refreshing in the background |
| `stale-if-error=86400` | If the origin is down, serve stale for up to a day |
| `no-transform` | Proxies must not recompress/resize your content |

### The question everyone gets wrong

> **`no-cache` vs `no-store` vs `must-revalidate`**

- **`no-cache`** — *"Cache it, but ask me every time before using it."* The response **is** stored. Each use triggers a conditional request (`If-None-Match`), which usually returns a **304** — so you still save the bandwidth of the body, just not the round trip. **This is the right default for authenticated API data**: always fresh, but cheap when unchanged.
- **`no-store`** — *"Do not write this to disk or memory, anywhere."* Nothing is stored, so every request is a full round trip with a full body. Use for genuinely sensitive one-time content: a payment page, a one-time token response, PII exports.
- **`must-revalidate`** — only matters **after** the response goes stale. It forbids serving stale content (which caches are otherwise permitted to do in some circumstances). Combine as `max-age=60, must-revalidate` for *"fresh for a minute, then strictly revalidate."*

**The one-liner for an interview:** *"`no-cache` means 'revalidate before use', not 'don't cache' — the response is still stored, which is why it still gets you 304s. `no-store` is the one that means don't cache."*

### The four recipes you'll actually use

```http
# 1. Immutable versioned asset (hashed filename) — cache forever
Cache-Control: public, max-age=31536000, immutable

# 2. Public, slow-changing API data (exchange rates, a product catalogue)
Cache-Control: public, max-age=60, s-maxage=300, stale-while-revalidate=600, stale-if-error=86400
ETag: "a3f9c1"

# 3. Authenticated per-user API data  ← the common case for Ledger
Cache-Control: private, no-cache
ETag: "7"
Vary: Authorization, Accept-Encoding

# 4. Genuinely sensitive / never cache
Cache-Control: no-store
```

Recipe 3 is the one to internalise: **`private, no-cache` + `ETag`** gives you correctness *and* bandwidth savings on authenticated endpoints. Most APIs use `no-store` here out of caution and throw away the 304s for nothing.

---

## 4. Conditional requests: the 304 mechanism

The highest-value, lowest-effort optimisation available to an API.

```http
GET /v1/payments/pi_3Nx8
→ 200 OK
  ETag: "7"
  Cache-Control: private, no-cache
  { ...3 KB of JSON... }

# Later, the client revalidates:
GET /v1/payments/pi_3Nx8
If-None-Match: "7"

→ 304 Not Modified          ← ~150 bytes total. No body at all.
  ETag: "7"
```

You still pay the round trip, but you save **the entire body** — plus serialization CPU, plus (if you're clever) the database read. For a mobile client on a slow network polling a list endpoint, this is the difference between usable and not.

### ETag: strong vs weak, and how to generate one

```http
ETag: "7"          # strong: byte-for-byte identical
ETag: W/"7"        # weak: semantically equivalent
```
`If-None-Match` accepts weak comparison; `If-Match` (concurrency, [Lesson 09](../02-rest-design/09-writes-patch-and-bulk.md)) requires **strong**.

| Source | Good for | Watch out |
|---|---|---|
| **`version` column** | Concurrency + caching, cheaply | Doesn't change if a *joined* field changes |
| **Hash of the response body** | Always accurate | You must build the body to know the ETag — so no DB savings unless you cache the hash |
| **`updated_at`** | Simple | Second-resolution collisions; `Last-Modified` has the same limit |
| **Composite** (`max(updated_at)` + row count for a list) | List endpoints | Needs a cheap aggregate query |

```ts
import { createHash } from "node:crypto";

/** Cheap, correct ETag middleware for JSON responses. */
export function jsonWithEtag(req: Request, res: Response, body: unknown) {
  const payload = JSON.stringify(body);
  const etag = `"${createHash("sha1").update(payload).digest("base64url").slice(0, 22)}"`;

  res.set("ETag", etag);
  res.set("Cache-Control", "private, no-cache");
  res.set("Vary", "Authorization, Accept-Encoding");

  // Note: If-None-Match can be a list, and may be "*"
  const inm = req.header("if-none-match");
  if (inm && (inm === "*" || inm.split(",").some(t => t.trim() === etag))) {
    return res.status(304).end();          // no body — must not send one
  }
  res.type("application/json").send(payload);
}
```

> **Getting the real win:** the snippet above still queries the database and serialises. To skip that too, store the ETag alongside the resource (a `version` column, or a cached hash in Redis), compare **before** doing the expensive work, and return 304 early. That's the difference between saving bandwidth and saving load.

---

## 5. `Vary`: the correctness header

```http
Vary: Authorization, Accept-Encoding, Origin
```

`Vary` tells caches: *"this response is only valid for requests with the same values of these headers."* Omitting it while varying your response is how caches leak data between users.

| You vary on | You must `Vary` | If you don't |
|---|---|---|
| `Authorization` (per-user data) | `Authorization` | A shared cache serves user A's data to user B. **Breach** |
| `Accept-Encoding` | `Accept-Encoding` | A client that can't gzip receives gzipped bytes |
| `Accept` (JSON/XML/CSV) | `Accept` | XML clients get JSON |
| `Origin` (CORS) | `Origin` | Merchant A's `Allow-Origin` cached and served to merchant B — every cross-origin call breaks |
| `Accept-Language` | `Accept-Language` | Wrong language |

**The safer belt-and-braces approach for authenticated endpoints:** `Cache-Control: private` **and** `Vary: Authorization`. `private` alone should keep it out of shared caches; `Vary` protects you if some intermediary is misconfigured. Defence in depth costs you nothing here.

> **Caution:** `Vary: *` means "never reusable" — effectively `no-store` for caching purposes. And a `Vary` on a high-cardinality header (like `User-Agent`) fragments your cache so badly that the hit rate approaches zero. CDNs let you normalise headers before the cache key precisely for this reason.

---

## 6. Application-level caching, and the three failure modes

HTTP caching handles the client side. Inside your service you'll also cache in Redis or memory — and each of these three failures has a name that interviewers use.

### Cache-aside (the standard pattern)
```ts
export async function getMerchantSettings(merchantId: string) {
  const key = `settings:${merchantId}`;
  const hit = await redis.get(key);
  if (hit) return JSON.parse(hit);

  const fresh = await db.settings.find(merchantId);
  await redis.set(key, JSON.stringify(fresh), "EX", 300);
  return fresh;
}
```

### Failure 1 — Thundering herd / cache stampede
A hot key expires. 5,000 concurrent requests all miss, all hit the database, and the database falls over. **The cache expiry became an outage.**

Fixes, in order of preference:
```ts
// A. Single-flight: only one caller computes; the rest await the same promise.
const inflight = new Map<string, Promise<unknown>>();
async function singleFlight<T>(key: string, fn: () => Promise<T>): Promise<T> {
  const existing = inflight.get(key) as Promise<T> | undefined;
  if (existing) return existing;
  const p = fn().finally(() => inflight.delete(key));
  inflight.set(key, p);
  return p;
}
// (per-instance; for cross-instance use a short Redis lock — SET NX PX)

// B. Jittered TTL: don't let 10,000 keys expire in the same second.
const ttl = 300 + Math.floor(Math.random() * 60);

// C. Serve stale while refreshing — the HTTP `stale-while-revalidate` idea, in Redis.
```

### Failure 2 — Cache penetration
Requests for keys that **don't exist** always miss, so every one hits the database. An attacker requesting random IDs turns your cache into a pass-through.

**Fix: cache the negative result too**, with a short TTL:
```ts
const NEGATIVE = "__none__";
const hit = await redis.get(key);
if (hit === NEGATIVE) return null;                 // cached miss
if (hit) return JSON.parse(hit);

const fresh = await db.find(id);
await redis.set(key, fresh ? JSON.stringify(fresh) : NEGATIVE, "EX", fresh ? 300 : 30);
return fresh;
```
(At extreme scale, a Bloom filter in front is the classic answer — worth naming.)

### Failure 3 — Stale data / invalidation
The genuinely hard one. Two strategies:

| Strategy | How | Trade-off |
|---|---|---|
| **TTL (expiry)** | Data is stale for up to the TTL | Simple, self-healing. **Default to this** |
| **Explicit invalidation** | Delete the key on write | Fresh immediately; but you must find *every* key affected, forever |

The pattern that avoids most invalidation pain — **key versioning**:
```ts
// Instead of deleting many keys, bump one version number.
const v = await redis.incr(`ver:merchant:${merchantId}`);       // on ANY write for this merchant
const key = `v${v}:settings:${merchantId}`;                      // old keys become unreachable
```
Old entries are orphaned and expire on their own. **One write invalidates everything derived from that entity** without you enumerating the derivations. This is a genuinely good answer to *"how do you invalidate a cache?"*

**And the rule that saves you:** *never cache anything you can't afford to be stale, and write down the acceptable staleness per endpoint.* "Balance: must be exact, no cache. Settings: 5 minutes is fine. Exchange rates: 1 minute. Country list: 1 day." That table is a design artifact, and producing it is what a senior engineer does before adding Redis.

---

## 7. What to cache in Ledger (the worked answer)

| Endpoint | Strategy | Why |
|---|---|---|
| `GET /v1/balance` | **`no-store`** | Money. Must be exact. A stale balance is a support ticket or a bad decision |
| `GET /v1/payments/{id}` | `private, no-cache` + ETag | Immutable once terminal; ETag makes repeated reads free |
| `GET /v1/payments` (list) | `private, no-cache` + composite ETag | Changes often, but 304s help pollers enormously |
| `GET /v1/me`, `/v1/settings` | `private, max-age=60` + Redis 5 min | Rarely changes; invalidate on write |
| `GET /v1/currencies`, `/v1/countries` | `public, max-age=86400, immutable`-ish | Reference data. Let the CDN serve it entirely |
| `GET /v1/openapi.json` | `public, max-age=300` | Static per deploy |
| `POST` anything | never cached; **invalidates** related keys | Writes invalidate ([Lesson 03](../01-foundations/03-http-methods-and-status.md)) |
| Auth token verification (JWKS) | in-process cache, 10 min, refetch on unknown `kid` | Otherwise every request fetches JWKS |
| Rate-limit counters | Redis, obviously | Shared state by definition |

**The `no-store` on `/balance` is a deliberate design statement**, and worth defending out loud: *"I won't cache a balance, because the cost of a stale number in a financial product exceeds any latency benefit. Everything else gets an ETag."*

---

## 8. The rest of HTTP performance

Caching is the biggest lever. These are the rest, roughly in order of payoff.

| Technique | Payoff | Notes |
|---|---|---|
| **Connection reuse (keep-alive)** | Removes 2–3 RTT per call | The cheapest win in existence (Lesson 02) |
| **HTTP/2 or /3** | Multiplexing; no 6-connection limit | Also makes chatty APIs viable |
| **Brotli/gzip on text** | 60–90% fewer bytes | `br` first, `gzip` fallback; level 4–5 for dynamic content |
| **A CDN or regional deployment** | Cuts RTT, which is distance-bound | You cannot beat the speed of light |
| **Pagination + small payloads** | Fewer RTTs (TCP slow start), less parse CPU | Under ~14KB lands in one round trip |
| **`?expand=` / field selection** | Removes client-side N+1 | Cap the depth |
| **Server-side N+1 elimination** | Often 10–100× on list endpoints | The most common real cause of a slow API |
| **Indexes matching your filters/sorts** | Turns scans into seeks | Part of your API contract (Lesson 08) |
| **`preconnect` / `dns-prefetch` from the client** | Removes DNS+TCP+TLS from the critical path | 100–300ms on a first call |
| **Streaming (`Transfer-Encoding: chunked`)** | Time-to-first-byte on big responses | Exports, LLM tokens |

### The server-side N+1, because it's the most common real bug
```ts
// ❌ 1 + N queries. Looks innocent. Is 201 queries for 200 payments.
const payments = await db.payments.list(merchantId, opts);
for (const p of payments) {
  p.customer = await db.customers.find(p.customerId);
}

// ✅ 2 queries, regardless of N
const payments = await db.payments.list(merchantId, opts);
const ids = [...new Set(payments.map(p => p.customerId).filter(Boolean))];
const customers = await db.customers.findMany(ids, merchantId);     // WHERE id = ANY($1)
const byId = new Map(customers.map(c => [c.id, c]));
for (const p of payments) p.customer = byId.get(p.customerId) ?? null;
```
**Detect it, don't guess:** log the query count per request and alert when it exceeds a threshold (say 20). A request whose query count scales with its result size is an N+1, and that one metric finds them all.

> **Spring equivalent:** you know this one — `JOIN FETCH`, `@EntityGraph`, `@BatchSize`. See `spring boot/06-data/24-performance-n1-and-migrations.md`. The Node version has no lazy loading to save you, so you must batch explicitly (or use a DataLoader, which is exactly what GraphQL servers do — [Lesson 21](../05-beyond-rest/21-graphql.md)).

---

## 9. Production rules

| Rule | Why |
|---|---|
| **Every response gets an explicit `Cache-Control`. No exceptions** | Silence means every cache guesses |
| **Authenticated data: `private` (or `no-store`) + `Vary: Authorization`** | The only thing preventing cross-user cache leaks |
| **`ETag` on every `GET`, and check `If-None-Match` before the expensive work** | 304s are nearly free and save whole bodies |
| **`no-cache` ≠ `no-store`.** Use `no-cache` for authenticated data you want revalidated | You get 304s instead of full bodies for nothing |
| **`Vary` on every header that changes the response** | Cache correctness, and it's a security control |
| **Jitter every TTL** | Prevents synchronised expiry stampedes |
| **Single-flight hot keys** | Stops one expiry becoming an outage |
| **Cache negative results, briefly** | Blocks cache penetration |
| **Prefer TTL over explicit invalidation; use key versioning when you need immediacy** | Enumerating every derived key is a losing game |
| **Write down the acceptable staleness per endpoint before adding a cache** | Otherwise "cached" silently means "sometimes wrong" |
| **Never cache money** | The cost of a stale balance exceeds the latency saved |
| **Log query count per request; alert above a threshold** | Finds N+1s automatically |
| **Measure with `curl -w`, not intuition** | Latency is usually not where you think (Lesson 02) |

---

## 10. Interview traps

**Q1. "`no-cache` vs `no-store` vs `must-revalidate`?"**
The §3 answer. Lead with *"`no-cache` doesn't mean don't cache"* — that framing shows you know the trap.

**Q2. "How does a 304 work and why do you care?"**
Client sends `If-None-Match` with the ETag it holds; if it still matches, the server returns `304` with **no body**. You save the entire payload (and, done right, the DB query and serialization). For mobile clients polling a list, it's transformative.

**Q3. "What does `Vary` do, and what breaks without it?"**
Declares which request headers change the response. Without it, a shared cache serves one user's authenticated response to another. **Security bug, not a performance bug.** The CORS variant (`Vary: Origin`) is the other common one.

**Q4. "Can you cache an authenticated response?"**
Yes — in a **private** cache. `Cache-Control: private, no-cache` + `ETag` + `Vary: Authorization` is the correct recipe: fresh on every use, but cheap. What you must not do is let it into a shared cache.

**Q5. "What's a cache stampede and how do you prevent it?"**
A hot key expires and N concurrent requests all miss and hit the origin. Fixes: single-flight (one computes, the rest await), jittered TTLs, serve-stale-while-revalidating, and pre-warming for known-hot keys.

**Q6. "How do you invalidate a cache?"**
Say the honest thing first: *"invalidation is the hard part, so I default to short TTLs and only add explicit invalidation where staleness is unacceptable."* Then the technique: **key versioning** — bump a version counter on write so all derived keys become unreachable at once, rather than trying to enumerate them.

**Q7. "Your API is slow. Where do you start?"**
The Lesson 02 method: split total latency into DNS / TCP / TLS / server-think / transfer with `curl -w`. Then, inside server-think: query count (N+1?), missing indexes, connection-pool saturation, downstream calls, and event-loop blocking. **Do not** start by adding Redis — cache the thing you've measured, not the thing you suspect.

**Q8. "You added a cache and now customers report stale data. What went wrong?"**
Almost certainly one of: no invalidation on write, missing `Vary` so a shared cache is serving cross-user, `public` where it should be `private`, or a TTL longer than the business tolerance. The fix starts with writing down the acceptable staleness per endpoint — which should have been step one.

**Q9. "Should you cache `GET /balance`?"**
No — and being able to *decline* to cache with a reason is as valuable as knowing how. Money must be exact; the round trip is cheaper than the support ticket.

**Q10. "What's the fastest possible API response?"**
The one you never send: a fresh browser-cache hit, zero bytes over the network. That's the sentence that shows you understand what caching is *for*.

---

## 11. Build & break

### Build — ETag + conditional GET, done properly
Add to Ledger's `GET /v1/payments/:id`:
1. A `version` column on `payments`, incremented on every write (you already have it from Lesson 09).
2. Fetch **only** `id, version` first; if `If-None-Match` matches, return `304` **without** the full read or serialization.
3. Otherwise do the full read, set `ETag`, `Cache-Control: private, no-cache`, `Vary: Authorization, Accept-Encoding`.

Measure: bytes and server time for a 200 vs a 304, using `curl -w "%{size_download} %{time_starttransfer}\n"`. Write both numbers down.

### Build — a safe cache-aside helper
```ts
type CacheOpts = { ttlSec: number; jitterSec?: number; negativeTtlSec?: number };

const inflight = new Map<string, Promise<unknown>>();

export async function cached<T>(key: string, opts: CacheOpts, fn: () => Promise<T | null>): Promise<T | null> {
  const hit = await redis.get(key);
  if (hit === "__none__") return null;                       // cached negative
  if (hit) return JSON.parse(hit) as T;

  const existing = inflight.get(key) as Promise<T | null> | undefined;
  if (existing) return existing;                              // single-flight

  const p = (async () => {
    const fresh = await fn();
    const ttl = fresh
      ? opts.ttlSec + Math.floor(Math.random() * (opts.jitterSec ?? Math.ceil(opts.ttlSec * 0.2)))
      : (opts.negativeTtlSec ?? 30);
    await redis.set(key, fresh ? JSON.stringify(fresh) : "__none__", "EX", ttl);
    return fresh;
  })().finally(() => inflight.delete(key));

  inflight.set(key, p);
  return p;
}

/** Key versioning: one bump invalidates everything derived from this merchant. */
export async function invalidateMerchant(merchantId: string) {
  await redis.incr(`ver:merchant:${merchantId}`);
}
export async function merchantKey(merchantId: string, suffix: string) {
  const v = (await redis.get(`ver:merchant:${merchantId}`)) ?? "0";
  return `v${v}:m:${merchantId}:${suffix}`;
}
```

### Break — five experiments
1. **See the 304 saving.** `curl -sI` to grab the ETag, then re-request with `If-None-Match`. Compare `size_download`. Then delete the ETag logic and watch the full body return every time.
2. **Cause a cross-user cache leak.** Put a caching proxy (nginx, 10 lines) in front of your API. Return `Cache-Control: public, max-age=60` on `/v1/payments` with **no** `Vary`. Request as merchant A, then as merchant B. **B gets A's payments.** Fix with `private` + `Vary` and confirm. This is the single most instructive security experiment in the module.
3. **Trigger a stampede.** Cache a 300ms query with a 5-second TTL. Fire 500 concurrent requests with `autocannon` and watch DB query count spike at each expiry. Add single-flight and jitter; watch it flatten.
4. **Cache penetration.** Request 10,000 random nonexistent IDs and count DB hits: 10,000. Add negative caching; count again.
5. **Find an N+1.** Add per-request query counting, load a 200-row list endpoint that expands a relation, and read the count. Then batch it and re-read. Log the before/after.

### Explain out loud (2 minutes)
1. The cache hierarchy, and private vs shared.
2. `no-cache` vs `no-store`, precisely.
3. The `private, no-cache, ETag, Vary` recipe and why it's right for authenticated data.
4. Stampede, penetration and invalidation — the three failure modes and their fixes.
5. Why you would refuse to cache a balance.

---

## What's next

Fast is good; **correct under failure** is better. Next is the most important production lesson in the track: what happens when the network breaks mid-request. Timeouts, retries, backoff, circuit breakers — and idempotency keys, the mechanism that makes a retried payment safe, which is the single most likely senior API interview question you will face.

Next → **[Lesson 18: Reliability — timeouts, retries, idempotency, circuit breakers](18-reliability-and-idempotency.md)**
