# Lesson 02 — The journey of one request: DNS → TCP → TLS → HTTP

> **Why this lesson exists:** you asked *"how does the browser handle the call, how does the server handle the call?"* — and it's the right question, because **every latency question, every timeout bug, every mysterious CORS error, and half of all API design decisions are consequences of this pipeline.** Engineers who treat the network as a black box guess. Engineers who know these twelve stages diagnose. This lesson makes you the second kind.

**Time:** ~90 minutes · **Prereq:** Lesson 01

---

## 1. The idea in one sentence

> **One API call is not one action — it is a dozen sequential stages, each with its own failure mode and its own cost, and "the API is slow" is meaningless until you know which stage.**

---

## 2. The whole map first

You call this:

```js
const res = await fetch("https://api.ledger.dev/v1/payments?limit=20", {
  headers: { Authorization: "Bearer eyJhbGci..." }
});
```

Here is everything that happens, in order. Memorise this list — it's the spine of the lesson and a genuinely great whiteboard answer.

```
CLIENT SIDE
 1. URL parse & normalise
 2. Cache check              → may return instantly, no network at all
 3. Security gates           → mixed content, HSTS, CORS preflight decision
 4. Service worker           → may intercept and answer offline
 5. Connection reuse check   → if a warm connection exists, skip 6–8 entirely
 6. DNS resolution           → hostname → IP        (~0–100 ms)
 7. TCP handshake            → SYN / SYN-ACK / ACK  (1 round trip)
 8. TLS handshake            → certificate + keys   (1–2 round trips)
 9. Request serialisation    → method, path, headers, body onto the wire

SERVER SIDE
10. Kernel accept queue → load balancer → reverse proxy → app process
11. App pipeline:  parse → route → authenticate → authorise → validate
                   → handler → DB / cache / other services → serialise
12. Response travels back, TCP-windowed and possibly compressed

CLIENT AGAIN
13. Parse status + headers, apply CORS check, store cookies, cache the response
14. Decompress, decode, `JSON.parse`, resolve the promise
```

**The single most important thing on that list:** stages 6, 7 and 8 happen **once per connection**, not once per request. On a warm HTTP/2 connection your request skips straight from stage 5 to stage 9. That one fact explains why your first API call takes 400ms and the next takes 40ms — and why "the API got faster after the first call" is not a caching mystery.

---

## 3. Client side, stage by stage

### Stage 1 — URL parse & normalise

`https://api.ledger.dev/v1/payments?limit=20` decomposes into:

| Part | Value | Note |
|---|---|---|
| scheme | `https` | decides port 443 and that TLS is mandatory |
| host | `api.ledger.dev` | what DNS resolves; **case-insensitive** |
| port | 443 (implied) | |
| path | `/v1/payments` | **case-sensitive** — `/Payments` is a different resource |
| query | `limit=20` | |
| fragment | *(none)* | `#foo` is **never sent to the server**. Client-only |

Three facts that show up in real bugs:

- **Host is case-insensitive; path is case-sensitive.** So `/v1/Payments` legitimately 404s. Pick lowercase and enforce it ([Lesson 07](../02-rest-design/07-resource-modelling-and-urls.md)).
- **The fragment never leaves the browser.** This is why OAuth's dead implicit flow put tokens in the fragment — the token never reached a server log. ([Lesson 14](../03-security/14-oauth2-and-oidc.md).)
- **Query strings are logged everywhere** — browser history, proxies, load balancers, CDN logs, your own access logs. Therefore: **never put a token, password, or PII in a query string.** This is a hard rule, and interviewers ask it.

### Stage 2 — Cache check (the request that never happens)

Before any network activity the browser consults its HTTP cache. If a fresh entry exists, you get a response **with zero network cost** (DevTools: `(memory cache)` / `(disk cache)`, `Size: 0 B`).

This is the fastest possible API call, and it is entirely controlled by *your server's response headers*. An API that sets no `Cache-Control` has silently opted out of the single largest performance win available to it. Full mechanics in [Lesson 17](../04-production/17-caching-and-performance.md).

> **Note:** `fetch()` respects the HTTP cache by default (`cache: "default"`). `cache: "no-store"` bypasses it. Many API clients set `no-store` reflexively and then wonder why their app makes 200 requests per page load.

### Stage 3 — Security gates, and where CORS actually happens

Three checks, in order:

1. **HSTS** — if the host previously sent `Strict-Transport-Security`, the browser rewrites `http://` → `https://` *before any request*. This is a client-side memory, not a redirect.
2. **Mixed content** — an `https://` page calling `http://` is blocked outright.
3. **CORS decision** — this is the one that costs you a day of your life at least once.

#### CORS, mechanically (the version that finally makes sense)

The rule browsers enforce is the **same-origin policy**: JavaScript on origin A may not *read* responses from origin B. An origin is the triple `(scheme, host, port)` — so `https://app.ledger.dev` and `https://api.ledger.dev` are **different origins**, and so are ports 3000 and 8080 on localhost.

Two things people get wrong, both worth stating precisely:

- **CORS is enforced by the browser, not by your server.** `curl` and Postman have no origin and no same-origin policy — which is why *"it works in Postman but not the browser"* is the single most common CORS symptom. CORS is not a security feature protecting your server; it protects the *user* from a malicious page reading their authenticated data from another site.
- **The request is often actually sent.** For a "simple" request the browser sends it, the server processes it (side effects included), and *then* the browser refuses to hand the response to your JS. Your `POST` may have succeeded while your code saw a network error.

**Simple vs preflighted.** The browser skips the preflight only if the request is "simple":
- method is `GET`, `HEAD`, or `POST`, **and**
- headers are limited to a CORS-safelisted set (`Accept`, `Accept-Language`, `Content-Language`, `Content-Type`), **and**
- `Content-Type` is one of `application/x-www-form-urlencoded`, `multipart/form-data`, `text/plain`.

Anything else — and note that this means **`Content-Type: application/json` or an `Authorization` header is enough** — triggers a **preflight**: a separate `OPTIONS` request asking permission first.

```http
OPTIONS /v1/payments HTTP/1.1
Host: api.ledger.dev
Origin: https://app.ledger.dev
Access-Control-Request-Method: POST
Access-Control-Request-Headers: authorization,content-type
```
```http
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://app.ledger.dev
Access-Control-Allow-Methods: GET,POST,PATCH,DELETE
Access-Control-Allow-Headers: authorization,content-type
Access-Control-Allow-Credentials: true
Access-Control-Max-Age: 86400
Vary: Origin
```

So **almost every real API call from a browser costs two round trips unless you handle it**, which is what `Access-Control-Max-Age` is for — it caches the preflight (capped at 2 hours in Chrome, 10 minutes in Firefox; browsers ignore larger values).

The rules that cause the actual bugs:

| Rule | Consequence |
|---|---|
| With `Allow-Credentials: true`, `Allow-Origin` **may not be `*`** | You must echo the specific origin. Reflecting *any* origin without an allowlist is a vulnerability |
| Custom **response** headers are invisible to JS unless listed in `Access-Control-Expose-Headers` | Your `X-Request-Id` or `X-RateLimit-Remaining` is "missing" in the browser but present in curl |
| `Vary: Origin` is required when you echo the origin | Otherwise a CDN caches merchant A's `Allow-Origin` and serves it to merchant B, breaking everyone |
| A **redirect** during a preflight fails | The `OPTIONS` must be answered directly, so put CORS *before* auth/redirect middleware |
| Preflights must **not require auth** | An `OPTIONS` carries no `Authorization` header. If your auth middleware 401s it, every cross-origin call dies |

> **The debugging rule:** if the browser fails but `curl` works, it is CORS or a cookie attribute — not your API logic. Read the *exact* console message; it names the missing header. Never "fix" CORS by disabling browser security or slapping `*` everywhere; that's how you fail a security review.

> **Spring equivalent:** `@CrossOrigin` / `WebMvcConfigurer#addCorsMappings`, and with Spring Security you must also add `.cors()` — the filter chain order bug you already know from `spring boot/05-web/17`.

### Stage 4 — Service worker
If registered, a service worker's `fetch` handler intercepts *before the network*. It can serve from Cache Storage, rewrite, or fabricate a response. Mostly a PWA/offline concern, but it's the reason a "network" response can arrive in 2ms while offline.

### Stage 5 — Connection reuse (the biggest free win)

The browser (and Node, and every HTTP client) keeps a pool of open connections per origin. If a usable one exists, **stages 6–8 are skipped entirely.**

| Protocol | Concurrency per origin | Note |
|---|---|---|
| HTTP/1.1 | ~6 connections, 1 in-flight request each | Requests 7+ **queue**. This is real: 20 parallel API calls serialise into 4 waves |
| HTTP/2 | 1 connection, ~100 concurrent streams | Multiplexed. This is why HTTP/2 changed API design — chatty is cheaper |
| HTTP/3 | 1 QUIC connection over UDP | No TCP head-of-line blocking; a lost packet stalls one stream, not all |

Practical consequences you should be able to state:

- **In Node, always reuse an agent.** `new Agent({ keepAlive: true })` (or `undici`, which pools by default) turns a 3-round-trip call into 1. Creating a fresh connection per request is one of the most common Node performance bugs — it can triple your outbound latency and exhaust ephemeral ports at volume.
- **Domain sharding is now an anti-pattern.** It was a genuine HTTP/1.1 trick (more origins = more connections). Under HTTP/2 it *hurts*, because you lose multiplexing and pay extra handshakes.

### Stage 6 — DNS: hostname → IP

`api.ledger.dev` is meaningless to IP. Resolution walks a cache hierarchy, stopping at the first hit:

```
browser cache (~60s) → OS cache → router → ISP/public resolver (8.8.8.8, 1.1.1.1)
   → root (.) → TLD (.dev) → authoritative NS → A/AAAA record
```

- **Cold, uncached:** 20–120ms, occasionally much worse.
- **Cached:** ~0ms.
- **TTL** on the record controls how long caches hold it. Low TTL (60s) = fast failover, more lookups. High TTL (1h) = fewer lookups, but a failover nobody honours for an hour. **This is why DNS-based failover is unreliable and why load balancers exist.**

Failure modes worth recognising instantly:
- `ENOTFOUND` / `NXDOMAIN` → the name doesn't exist (typo, or the record wasn't created).
- `EAI_AGAIN` → resolver timeout, i.e. a *DNS* problem, not your API.
- **Node's DNS resolution is not on the event loop** — it uses the libuv threadpool (default 4 threads). Slow DNS can therefore block file I/O and crypto in a Node service. A genuinely senior-sounding detail.

> **Speed-up you'll see in prod:** `<link rel="dns-prefetch">` / `preconnect` to your API origin, so DNS+TCP+TLS complete *while the page is still parsing*. On a real dashboard this removes 100–300ms from the first API call.

### Stage 7 — TCP: the three-way handshake

```
client → SYN         "let's talk, my sequence number is X"
server → SYN-ACK     "acknowledged, mine is Y"
client → ACK         "acknowledged"      ← data may now flow
```

**Cost: exactly one round trip (1 RTT).** RTT is dominated by physical distance:

| Path | RTT |
|---|---|
| Same datacenter | 0.2–1 ms |
| Same city | 5–15 ms |
| Mumbai ↔ Frankfurt | ~110 ms |
| Mumbai ↔ us-east-1 | ~200 ms |
| Satellite / bad mobile | 300–700 ms |

You cannot beat the speed of light. **This is the entire reason CDNs and regional deployments exist**, and the reason "reduce round trips" outranks "make the server faster" for most API performance work. A 200ms RTT means a cold HTTPS call costs ≥600ms *before your server does anything*.

Also from TCP, two things that appear in interviews:

- **Slow start.** TCP begins with a small congestion window (~10 packets ≈ 14KB) and doubles it per RTT. So the first ~14KB of a response arrives in one RTT, and a 100KB JSON payload takes several. **Payload size costs round trips, not just bandwidth** — which is a much better argument for pagination than "it's cleaner".
- **`ECONNRESET` vs `ETIMEDOUT` vs `ECONNREFUSED`.** Refused = nothing listening on that port (process down, wrong port). Reset = the peer actively killed an established connection (crash, LB idle timeout, or a proxy dropping you). Timed out = packets vanished (firewall blackhole, wrong security group). Knowing these three apart makes you look like you've been on call.

### Stage 8 — TLS: the handshake that protects it

```
TLS 1.2:  ClientHello → ServerHello+Certificate → KeyExchange → Finished   ≈ 2 RTT
TLS 1.3:  ClientHello → ServerHello+Finished                               ≈ 1 RTT
TLS 1.3 with resumption / 0-RTT:                                           ≈ 0 RTT
```

What it establishes:
1. **Authentication** — the server proves it owns the domain via a certificate chained to a trusted CA. (This is what a browser's padlock means: *identity*, not *safety*.)
2. **Key agreement** — both sides derive a shared symmetric key (ECDHE, giving forward secrecy).
3. **Integrity** — nobody can tamper undetected.

Facts that matter operationally:

- **TLS 1.3 halves handshake cost.** If your API still negotiates 1.2, you're paying an extra RTT on every cold connection — 200ms extra for a mobile user in another region.
- **ALPN** — during the handshake, client and server negotiate `h2` vs `http/1.1`. So **HTTP/2 is chosen inside the TLS handshake**, which is why HTTP/2 in browsers is effectively HTTPS-only.
- **SNI** — the client sends the hostname *in the clear* so one IP can host many certs. Consequence: **the hostname you visit is visible to the network even over HTTPS** (unencrypted-SNI is why "HTTPS hides everything" is false).
- **Common failures:** expired certificate (the #1 self-inflicted outage in the industry — set expiry alerts), incomplete chain (works in browsers which cache intermediates, fails in `curl`/Java — a classic "works on my machine"), hostname mismatch, and clock skew on the client.
- **mTLS** adds a client certificate, so both sides authenticate. This is the standard for partner/bank APIs ([Lesson 12](../03-security/12-authentication-landscape.md)).

### Stage 9 — Serialising the request

What actually goes on the wire (HTTP/1.1, human-readable):

```http
GET /v1/payments?limit=20 HTTP/1.1
Host: api.ledger.dev
Authorization: Bearer eyJhbGciOiJSUzI1NiIs...
Accept: application/json
Accept-Encoding: gzip, br
User-Agent: Mozilla/5.0 ...
Connection: keep-alive
```

Note `Host` — mandatory in HTTP/1.1 and the mechanism by which one IP serves many domains (virtual hosting). Under HTTP/2 this becomes the `:authority` pseudo-header and everything is **binary** and **HPACK-compressed**, so those repeated 800-byte header blocks cost almost nothing after the first request. Headers get their full treatment in [Lesson 04](04-headers-and-payloads.md).

---

## 4. Server side: what happens on arrival

This is the half most tutorials skip entirely, and it's where you should slow down — it's the answer to *"how does the server handle the call?"*.

### Stage 10 — Before your code runs

```
NIC → kernel (SYN queue → accept queue) → load balancer → reverse proxy / API gateway → app process
```

- **Kernel accept queue** (`somaxconn`, `backlog`). Completed connections wait here until your app calls `accept()`. **If your app is too busy to accept, the queue fills and the kernel drops SYNs** — the client sees a *connection timeout* while your app logs nothing at all and your CPU looks fine. Recognising this symptom is a genuinely senior diagnostic.
- **Load balancer** (ALB/NGINX/Envoy) — picks a backend, and **has its own timeouts**. The classic production bug: LB idle timeout 60s, app keep-alive 75s → the app holds a connection the LB already discarded → intermittent `502`s that no single service's logs explain. **Rule: the upstream idle timeout must be shorter than the downstream one.**
- **API gateway** — TLS termination, routing, authn, rate limiting, WAF. Note that after TLS terminates at the edge, your app sees plain HTTP, and the real client IP survives only in `X-Forwarded-For`. Trusting that header blindly is how IP-based rate limits get bypassed ([Lesson 19](../04-production/19-rate-limiting.md)).

### Stage 11 — Inside your app: the middleware pipeline

Every framework is the same idea — a chain of functions around your handler. Order is not cosmetic; it's a security and correctness decision.

```ts
// Express — the order I would defend in a review, and why
app.use(requestId());        // 1. first, so EVERY log line and error carries the ID
app.use(logger());           // 2. log even requests that get rejected later
app.use(helmet());           // 3. security headers, cheap
app.use(cors(corsOptions));  // 4. BEFORE auth — preflights carry no credentials
app.use(rateLimit());        // 5. BEFORE auth — cheap rejection before expensive crypto
app.use(express.json({ limit: "100kb" }));  // 6. bounded! unbounded body = DoS
app.use(authenticate);       // 7. who are you  → 401
app.use(authorize);          // 8. may you      → 403
app.use(validate(schema));   // 9. is the input sane → 400/422
app.use("/v1", routes);      // 10. your handler
app.use(errorHandler);       // 11. LAST — the only place that formats errors
```

Four things in that list are exam material:

1. **`requestId` first.** Without it you cannot correlate a customer's failed request with your logs, and support becomes guesswork. Return it as `X-Request-Id` on **every** response, including errors.
2. **CORS before auth.** Preflights are unauthenticated; auth-first returns 401 to an `OPTIONS` and every browser call dies.
3. **Rate limit before auth.** Verifying a JWT signature or bcrypt-comparing a password is expensive; an attacker sending 10,000 bad tokens shouldn't get 10,000 signature verifications.
4. **Bounded body parsing.** `express.json()` with no `limit` will happily buffer a 2GB body into memory. That's a one-line denial of service.

Then, inside the handler:

```
route match → deserialise body → validate → authorise *this specific object* (BOLA!)
   → business logic → DB queries / cache / downstream calls → map to DTO → serialise → send
```

The **authorise-this-object** step is the one people skip, and it's the #1 real-world API vulnerability: you checked the caller is *a* logged-in merchant, not that they own payment `pi_123` ([Lesson 15](../03-security/15-authorization-and-multitenancy.md)).

### How the server handles *concurrency* — the part that decides your throughput

This is where interviewers separate people who deploy servers from people who understand them.

| Model | Who | How concurrency works | Fails when |
|---|---|---|---|
| **Thread per request** | Spring MVC (Tomcat), Rails, PHP | Pool of ~200 threads; a blocking DB call parks a thread | All threads blocked on a slow dependency → total stall, even though CPU is 3% |
| **Event loop** | Node.js, nginx | 1 thread, non-blocking I/O, callbacks on a queue | **Any CPU-heavy or synchronous work blocks *everything*** |
| **Async/await runtime** | Go, Rust/Tokio, Spring WebFlux | Lightweight tasks multiplexed onto few OS threads | Complexity; a single blocking call poisons the pool |
| **Virtual threads** | Java 21+ | Thread-per-request *syntax*, event-loop *economics* | Pinning on `synchronized` blocks/native calls |

**For Node specifically — the two facts you must be able to say:**

1. **`JSON.parse` of a 10MB payload blocks the event loop.** Every other in-flight request waits. This is why payload size limits and pagination are *availability* features in Node, not tidiness. Same for `bcrypt` sync, big `JSON.stringify`, regex backtracking, and synchronous `fs` calls.
2. **The libuv threadpool is 4 threads by default** (`UV_THREADPOOL_SIZE`) and serves file I/O, DNS, `crypto.pbkdf2`, and zlib. Saturate it and unrelated things get slow for no visible reason.

> **Spring equivalent:** you already know this from `spring boot/05-web/18-mvc-vs-webflux-virtual-threads.md`. Same trade-off space, different defaults. Being able to compare the two ecosystems out loud is a strong senior signal — most candidates only know one.

### Stage 12 — The response back

```http
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Content-Length: 3184
Content-Encoding: gzip
Cache-Control: private, max-age=0, must-revalidate
ETag: "a3f9c1"
X-Request-Id: 01HQ8...
Vary: Accept-Encoding, Origin
```

Two mechanics worth knowing:

- **Framing:** the client must know where the body ends — either `Content-Length` or `Transfer-Encoding: chunked` (streaming, length unknown up front). Chunked is how you stream a huge export or an LLM's tokens without buffering it all in memory.
- **Compression:** `gzip` or `br` (Brotli). Text/JSON compresses 60–90%. Two important caveats: don't compress already-compressed bytes (JPEG, ZIP — you burn CPU for nothing), and never compress a response that mixes secrets with attacker-controlled input over a shared channel (the BREACH class of attack) — in practice, don't compress responses containing CSRF tokens.

### Stages 13–14 — The client finishes the job

Parse status/headers → **apply the CORS check** (only now does the browser decide whether your JS may see it) → store `Set-Cookie` → write to HTTP cache per `Cache-Control`/`ETag` → decompress → `JSON.parse` → resolve the promise.

One trap worth internalising: **`fetch()` does not reject on 4xx/5xx.** It rejects only on *network* failure. So this code is wrong, and it's in a lot of real codebases:

```ts
// ❌ 500 sails straight through
const data = await fetch(url).then(r => r.json());

// ✅
const res = await fetch(url);
if (!res.ok) throw new ApiError(res.status, await res.json().catch(() => null));
const data = await res.json();
```

(`axios` throws by default; `fetch` does not. That difference is a real interview question.)

---

## 5. Now measure it yourself

Reading this is worth 20% of the lesson. This is the other 80%.

```bash
curl -w "\nDNS:        %{time_namelookup}s\nTCP:        %{time_connect}s\nTLS:        %{time_appconnect}s\nTTFB:       %{time_starttransfer}s\nTOTAL:      %{time_total}s\nHTTP:       %{http_version}\n" -o NUL -s https://api.github.com/users/torvalds
```

These are **cumulative** timestamps, so the stage costs are the differences:

| Stage cost | Formula | What a bad number means |
|---|---|---|
| DNS | `time_namelookup` | Resolver problem, cold cache, or too-low TTL |
| TCP | `connect − namelookup` | Physical distance / RTT. Fix with a closer region or CDN |
| TLS | `appconnect − connect` | Stuck on TLS 1.2, or a huge cert chain |
| **Server think time** | `starttransfer − appconnect` | **This is the only part your application code controls** |
| Transfer | `total − starttransfer` | Payload too big, or no compression |

Run it twice in a row and notice the second is faster — that's DNS caching plus TLS session resumption. Now you have felt stage 5.

**Do these four experiments** (15 minutes, and they will stick for years):

```bash
# 1. Watch the protocol negotiation and full header exchange
curl -v https://api.github.com/users/torvalds 2>&1 | head -40

# 2. Force HTTP/1.1 and compare TTFB with the default (h2)
curl --http1.1 -w "\n%{time_total} %{http_version}\n" -o NUL -s https://api.github.com/
curl -w "\n%{time_total} %{http_version}\n" -o NUL -s https://api.github.com/

# 3. See a CORS preflight with your own eyes
curl -X OPTIONS https://api.github.com/user -H "Origin: https://evil.com" \
     -H "Access-Control-Request-Method: DELETE" -i

# 4. Prove compression is real (compare the two Content-Length values)
curl -s -o NUL -w "identity: %{size_download} bytes\n" -H "Accept-Encoding: identity" https://api.github.com/repos/facebook/react
curl -s -o NUL -w "gzip:     %{size_download} bytes\n" -H "Accept-Encoding: gzip"     https://api.github.com/repos/facebook/react
```

> **On Windows PowerShell** `curl` is an alias for `Invoke-WebRequest`. Use `curl.exe` explicitly, or run these in Git Bash.

---

## 6. Production rules that fall out of this pipeline

| Rule | Why (name the stage) |
|---|---|
| **Always set an explicit client timeout** | Stage 7/11. Default in `fetch` is *none* — a hung request holds a socket and a Node handle forever. Node's `undici` defaults help; `axios` defaults to no timeout |
| **Timeouts must decrease downstream** | Gateway 30s > service 10s > DB 3s. If the DB timeout exceeds the gateway's, you do work nobody is waiting for — and hold a connection for it |
| **Reuse connections** (`keepAlive`) | Stage 5–8. Removes 2–3 RTT per call. The cheapest possible win |
| **`preconnect` your API origin from the browser** | Stages 6–8 overlap with page parse, removing them from the critical path |
| **Cap request body size** | Stage 11. Unbounded parsing is a DoS, and in Node it blocks the loop |
| **Paginate; keep responses under ~14KB where practical** | Stage 7 slow start: small responses land in one RTT |
| **Compress text responses** | Stage 12. 60–90% fewer bytes ≈ fewer RTTs |
| **Return `X-Request-Id` on every response, errors included** | Stage 11/13. It's the only way to answer "my request failed at 14:03" |
| **Never put secrets or PII in the query string** | Stage 1. Query strings are logged at every hop |
| **Put CORS + rate limiting before auth** | Stage 11. Preflights are unauthenticated; crypto is expensive |
| **Never trust `X-Forwarded-For` unless your proxy sets it** | Stage 10. Clients can forge it, bypassing IP rate limits |

---

## 7. Interview traps

**Q1. "You type a URL / call an API. Walk me through what happens."**
The classic. Structure your answer in the four blocks from §2 — client pre-flight, connection setup, server pipeline, response handling — and **name a cost and a failure mode per stage**. A candidate who says "DNS, TCP, TLS, then the server responds" has answered adequately. A candidate who says *"and DNS is cached at four levels, TCP costs one RTT which is distance-bound, TLS 1.3 costs one more, and all three are skipped on a warm connection, which is why the second call is 10× faster"* is done being interviewed.

**Q2. "Your API's p50 is 40ms but p99 is 4s. Where do you look?"**
The whole point of §5. Answer with the *method*, not a guess: split total into DNS / TCP / TLS / server-think / transfer, and check them in order.
- p99 only for *some* clients → geography (RTT) or a cold-connection problem.
- Server-think is the spike → DB (missing index, lock contention, connection pool exhaustion), a slow downstream, or GC/event-loop blocking.
- Spikes are periodic → cron job, cache expiry stampede, or log rotation.
- p99 ≈ exactly your timeout value → you're measuring the *timeout*, not the work.
Then: *"and I'd want the p99 broken down per endpoint and per client, because an aggregate p99 usually hides one pathological caller."*

**Q3. "It works in Postman but fails in the browser."**
CORS (or a cookie attribute — `SameSite`/`Secure`). Postman has no origin, so no same-origin policy. Then explain preflights, and — for extra credit — that a "simple" cross-origin `POST` **does execute on the server** even when the browser hides the response, so a failed-looking request may have already charged the card.

**Q4. "What is a CORS preflight and when does it happen?"**
An `OPTIONS` request asking permission before the real one. Triggered by any non-simple request — which includes `Content-Type: application/json` and any `Authorization` header, so essentially every real API call. Mitigate with `Access-Control-Max-Age`.

**Q5. "How would you make your API faster without touching the handler?"**
Everything outside stage 11: keep-alive, HTTP/2 or 3, TLS 1.3, Brotli, `Cache-Control` + `ETag`, a CDN or regional deployment to cut RTT, smaller payloads, and `preconnect` from the client. Naming these proves you understand that *most* API latency is often not in the application code at all.

**Q6. "What's the difference between `ECONNREFUSED`, `ECONNRESET` and `ETIMEDOUT`?"**
Nothing listening / peer killed an established connection / packets disappeared. Then the useful follow-up: *"intermittent `ECONNRESET` under load usually means an idle-timeout mismatch between the LB and the app, or the app being restarted without connection draining."*

**Q7. "Why does the first request to my API take 500ms and the rest 30ms?"**
Cold DNS + TCP + TLS (3 RTTs), plus possibly a cold serverless container, an empty cache, and a lazily-initialised connection pool. All skipped afterwards. **Do not** say "caching" without naming which cache.

**Q8. "Does HTTPS hide the URL?"**
Path and query are encrypted; **the hostname is not** (SNI, and DNS). So `https://api.ledger.dev/v1/customers/ssn/123` hides the path from the network but the hostname leaks — and the path still lands in your own logs, the CDN's logs, and browser history. Hence: never put secrets in URLs.

**Q9. "Why is HTTP/2 better, and when is it not?"**
Better: one connection, multiplexed streams, HPACK header compression, no 6-connection limit. Not better: over lossy networks TCP head-of-line blocking hits *all* streams on that one connection, where HTTP/1.1's six connections degrade more gracefully — which is precisely the problem HTTP/3 (QUIC over UDP) solves.

---

## 8. Build & break

### Build — a client that respects the pipeline
Create `scratch/client.ts`. Everything here is a direct consequence of a stage above.

```ts
import { Agent, request } from "undici";

// Stage 5: one pooled, keep-alive agent for the whole process.
const agent = new Agent({
  keepAliveTimeout: 30_000,
  connections: 50,              // cap so one dependency can't exhaust sockets
});

export async function callApi<T>(path: string, init: {
  method?: string; body?: unknown; token?: string; timeoutMs?: number;
} = {}): Promise<T> {
  const { method = "GET", body, token, timeoutMs = 5_000 } = init;

  // Stage 7/11: never rely on a default timeout. There often isn't one.
  const ac = new AbortController();
  const timer = setTimeout(() => ac.abort(), timeoutMs);

  const requestId = crypto.randomUUID();                 // Stage 11: correlation
  try {
    const res = await request(`https://api.ledger.dev${path}`, {
      method, dispatcher: agent, signal: ac.signal,
      headers: {
        "accept": "application/json",
        "accept-encoding": "gzip, br",                   // Stage 12
        "x-request-id": requestId,
        ...(token ? { authorization: `Bearer ${token}` } : {}),
        ...(body ? { "content-type": "application/json" } : {}),
      },
      body: body ? JSON.stringify(body) : undefined,
    });

    // Stage 14: status is NOT an exception. Check it yourself.
    if (res.statusCode >= 400) {
      const problem = await res.body.json().catch(() => null);
      throw new ApiError(res.statusCode, problem, requestId);
    }
    return (await res.body.json()) as T;
  } finally {
    clearTimeout(timer);
  }
}

class ApiError extends Error {
  constructor(readonly status: number, readonly problem: unknown, readonly requestId: string) {
    super(`API ${status} (request ${requestId})`);
  }
}
```

Read it once more and label each line with its stage number. That exercise is the lesson.

### Break — five deliberate failures
Do all five. Each one buys you a symptom you'll recognise for the rest of your career.

1. **Blow up the event loop.** In a tiny Express server, add `app.get("/slow", () => { const t = Date.now(); while (Date.now() - t < 5000); res.send("ok") })`. Hit `/slow` in one tab and any other route in another. Watch *everything* freeze. That's stage 11, Node edition.
2. **Cause a real CORS failure.** Serve a static HTML page on `http://localhost:3000` that `fetch`es your API on `:4000` with an `Authorization` header. Read the exact console message. Then fix it with proper CORS config — not `*`.
3. **Remove the timeout.** Point your client at `https://httpstat.us/200?sleep=60000` with no timeout, and watch the request hang. Now add `timeoutMs`.
4. **Kill keep-alive.** Loop 50 requests with `keepAlive: true`, then with a fresh agent per request. Compare total time. The number will surprise you — that's stages 6–8, fifty times over.
5. **Overflow the body limit.** POST a 5MB JSON body to `express.json({ limit: "100kb" })`. Note the status code you get (`413`) and, more importantly, that without that limit you'd have buffered all 5MB.

### Explain out loud (90 seconds)
1. The twelve stages, in order.
2. Which stages a warm connection skips, and why that explains "the second call is faster".
3. Why CORS fails in the browser but not in curl.
4. Where in the pipeline *your* code can actually make a difference.

---

## What's next

You've seen the pipeline. Now we go into HTTP itself — the protocol every design rule in Module 2 is derived from: what each method *promises*, what each status code *means to a client*, and the safe/idempotent/cacheable distinction that decides whether retries are allowed to exist.

Next → **[Lesson 03: HTTP in full — methods, status, semantics](03-http-methods-and-status.md)**
