# Lesson 04 — Headers & payloads: the anatomy of a message

> **Why this lesson exists:** you asked *"what is payload, what is header?"* — and the reason it's worth a full lesson is that **headers are where the operational half of your API contract lives.** Auth, caching, content negotiation, compression, correlation, rate-limit feedback, concurrency control: none of it is in the body. Engineers who only think about the body ship APIs where every client has to invent its own conventions.

**Time:** ~80 minutes · **Prereq:** Lesson 03

---

## 1. The idea in one sentence

> **An HTTP message is a control plane (start line + headers) wrapped around an opaque data plane (the body) — headers describe and govern the body, and the body is meaningless without them.**

---

## 2. The anatomy, precisely

Every HTTP message has exactly the same four-part structure:

```http
POST /v1/payments HTTP/1.1                    ← 1. start line
Host: api.ledger.dev                          ┐
Content-Type: application/json                │ 2. headers
Content-Length: 62                            │
Authorization: Bearer eyJhbGci...             ┘
                                              ← 3. empty line (CRLF) — the separator
{"amount":4999,"currency":"usd","customer":"cus_9s2k"}   ← 4. body
```

The blank line is not cosmetic. **It is the delimiter that tells the parser "headers are done, everything after this is bytes I should not interpret."** That single rule is why HTTP header injection is a vulnerability class: if you let user input contain a `\r\n`, they can forge headers or split your response into two ([§9](#9-security-implications-you-must-know)).

Response side, same structure:

```http
HTTP/1.1 201 Created
Content-Type: application/json; charset=utf-8
Content-Length: 184
Location: /v1/payments/pi_3Nx8
X-Request-Id: 01HQ8ZK3

{"id":"pi_3Nx8","amount":4999,...}
```

### Header syntax rules that matter

| Rule | Detail |
|---|---|
| **Names are case-insensitive** | `Content-Type` ≡ `content-type`. HTTP/2 and HTTP/3 **require lowercase** on the wire |
| **Values are (mostly) ASCII strings** | Non-ASCII must be encoded (RFC 8187: `filename*=UTF-8''caf%C3%A9.pdf`) |
| **Some can repeat** | `Set-Cookie` legitimately appears many times. Others are comma-joined: `Accept: a, b` |
| **Order is not significant** | Except for repeated `Set-Cookie` |
| **Size is limited by the server, not the spec** | Typically 8KB total (nginx `large_client_header_buffers`, Node's `--max-http-header-size`). Exceed it → **`431 Request Header Fields Too Large`**. This is how "too many cookies" breaks a site |
| **`X-` prefix is deprecated** | RFC 6648 (2012) retired it. `X-Request-Id` survives by sheer convention; new headers shouldn't add the prefix |

---

## 3. What "payload" actually means

You asked specifically, so let's be precise, because the word is used three different ways and the sloppiness causes real confusion.

| Term | Precise meaning |
|---|---|
| **Body** | The bytes after the blank line, exactly as transmitted (possibly gzipped, possibly chunk-framed) |
| **Payload** *(HTTP/1.1, RFC 7230)* | The body **plus** the headers that describe it (`Content-Type`, `Content-Length`, `Content-Encoding`) — "payload" = the shipment and its label |
| **Content / representation** *(RFC 9110, current)* | The **decoded, semantic data** — the JSON object itself, after decompression |

The modern spec deliberately renamed "payload" to **"content"** to fix exactly this ambiguity. So `Content-Length` is the length *after* encoding, while the "content" is what you get *after* decoding. That's why `Content-Length: 812` can accompany a 4KB JSON object — 812 is the gzipped size.

**In everyday engineering speech, "payload" just means "the body you send/receive"** — that's fine, and it's what everyone means in an interview. But if someone asks *"is `Content-Type` part of the payload?"*, the precise answer is yes in HTTP/1.1 terms, and knowing that distinction exists is a nice detail.

### Which methods carry a payload?

| Method | Request body | Notes |
|---|---|---|
| `GET` / `HEAD` | **No** (in practice) | Spec-legal but proxies drop it, servers ignore it, and caches key on URI only. Never design for it |
| `POST` / `PUT` / `PATCH` | **Yes** | The point |
| `DELETE` | Discouraged | Some APIs use it for bulk delete; many intermediaries drop it. Prefer `POST /resource/bulk-delete` |
| `OPTIONS` | No | |

### How the receiver knows where the body ends (framing)

This is genuinely important and almost never taught:

1. **`Content-Length: 62`** — read exactly 62 bytes. Requires knowing the size up front, so the whole body must be buffered/computed first.
2. **`Transfer-Encoding: chunked`** — the body arrives as length-prefixed chunks terminated by a zero-length chunk. Size unknown in advance. **This is how you stream** a giant CSV export, a live log tail, or an LLM's tokens without holding it all in memory.
   ```
   1a\r\n{"event":"payment.paid"}\r\n
   0\r\n\r\n
   ```
3. **Connection close** — HTTP/1.0 style, "body ends when the socket closes." Unusable with keep-alive.
4. **HTTP/2 / HTTP/3** — frames carry their own lengths; `Transfer-Encoding` is *forbidden* and `Content-Length`, if present, is only a consistency check.

> **Why you care:** sending both `Content-Length` and `Transfer-Encoding`, or letting a proxy and an origin disagree about which to honour, is the **HTTP request smuggling** vulnerability class — an attacker prepends bytes that the backend reads as a *separate request*, letting them poison another user's response. It's a top-tier real-world attack. Practical defence: keep your proxy and app on the same HTTP version and don't hand-roll HTTP parsing.

---

## 4. Content negotiation — the mechanism REST is built on

The client says what it *wants*; the server says what it *sent*. Same resource, many representations.

```http
GET /v1/payments/pi_1
Accept: application/json                 ← "give me JSON"
Accept-Encoding: gzip, br                ← "you may compress"
Accept-Language: en-GB, en;q=0.8         ← "British English preferred"
```
```http
200 OK
Content-Type: application/json; charset=utf-8   ← "here's what it is"
Content-Encoding: gzip                          ← "and it's gzipped"
Content-Language: en-GB
Vary: Accept-Encoding, Accept-Language          ← "cache separately per these"
```

Four things to internalise:

**1. `Accept` vs `Content-Type` are opposite directions.** `Accept` = what I can consume (a request-side wish list, with `q=` quality weights). `Content-Type` = what this message's body *is*. A request can carry both: *"here's JSON (`Content-Type`), send me back XML (`Accept`)."* Mixing these up is a very common junior error.

**2. `Content-Type` includes parameters.** `application/json; charset=utf-8`. For JSON, charset is technically redundant (JSON is UTF-8 by definition per RFC 8259) but harmless. For `multipart/form-data` the parameter is essential: `boundary=----X`.

**3. `Vary` is mandatory when the response depends on a request header.** If you return different bodies for different `Accept` or `Authorization` or `Origin` values and don't set `Vary`, a shared cache will serve one user's response to another. **This is a real data-leak mechanism, not a theoretical one** — it's how CDNs have served logged-in users' pages to strangers.

**4. Content negotiation vs URL-based versioning** is a genuine design debate: `Accept: application/vnd.ledger.v2+json` (correct-by-the-book) vs `/v2/payments` (obvious, cacheable, curl-able). [Lesson 11](../02-rest-design/11-versioning-and-evolution.md) settles it — spoiler: almost everyone chooses the URL, and they're right for pragmatic reasons.

---

## 5. The headers you must know cold

Grouped by job. Learn the group first; the individual headers then feel obvious.

### A. Framing & description of the body

| Header | Direction | What it does |
|---|---|---|
| `Content-Type` | both | Media type + parameters. **Servers must validate it** — else a client can post XML to your JSON parser |
| `Content-Length` | both | Byte length after encoding |
| `Transfer-Encoding: chunked` | both | Streaming framing (HTTP/1.1 only) |
| `Content-Encoding` | both | Compression actually applied (`gzip`, `br`, `zstd`) |
| `Content-Disposition` | response | `attachment; filename="report.csv"` → triggers a download |
| `Content-Language` / `Content-Location` | response | Which language / canonical URI of this representation |

### B. Authentication & identity

| Header | Notes |
|---|---|
| `Authorization: Bearer <token>` | The standard. **`Bearer` literally means "whoever holds it, wins"** — so it must never be logged, never appear in a URL, and must travel over TLS only |
| `Authorization: Basic base64(user:pass)` | **Base64 is encoding, not encryption.** Trivially reversible. Only acceptable over TLS, and even then prefer tokens |
| `WWW-Authenticate` | Response header **required** with `401`: `Bearer realm="api", error="invalid_token", error_description="expired"` — this is how a client knows whether to refresh or re-login |
| `Proxy-Authorization` | For the proxy hop, not the origin |
| `Cookie` / `Set-Cookie` | Browser-managed credential store — see §7 |
| `X-API-Key` | Non-standard but ubiquitous. Fine; just be consistent ([Lesson 12](../03-security/12-authentication-landscape.md)) |

### C. Caching & concurrency (the highest-value group)

| Header | Notes |
|---|---|
| `Cache-Control` | The master switch, both directions. `max-age`, `no-cache`, `no-store`, `private`, `public`, `must-revalidate`, `stale-while-revalidate` |
| `ETag: "a3f9c1"` | An opaque version identifier for this representation |
| `If-None-Match` | Request: "only send it if the ETag differs" → **`304 Not Modified`**, zero body |
| `If-Match` | Request: "only *write* if the ETag still matches" → **`412`** on mismatch. **This is optimistic concurrency control** and it's how you prevent lost updates |
| `Last-Modified` / `If-Modified-Since` | Weaker, second-resolution version of the same idea |
| `Age`, `Expires` | Cache age; legacy absolute expiry |
| `Vary` | Which request headers change the response |

`ETag` earns two lessons of its own ([17](../04-production/17-caching-and-performance.md) and [09](../02-rest-design/09-writes-patch-and-bulk.md)) because it solves *two different problems with one header*: bandwidth (`If-None-Match`) and correctness (`If-Match`).

### D. Rate limiting & operations

| Header | Notes |
|---|---|
| `Retry-After: 30` | Seconds, or an HTTP date. **Required** on `429` and `503`. Without it, clients guess — and they guess badly |
| `RateLimit-Limit` / `RateLimit-Remaining` / `RateLimit-Reset` | The IETF draft standard. GitHub/Stripe use `X-RateLimit-*`. Pick one and expose it — a client that can see its budget doesn't need to be throttled |
| `X-Request-Id` / `Traceparent` | Correlation. `traceparent` is the W3C standard for distributed tracing; propagate it ([Lesson 20](../04-production/20-observability.md)) |
| `Server`, `X-Powered-By` | **Remove these.** Free reconnaissance for attackers |
| `Deprecation`, `Sunset`, `Link` | How you announce an endpoint's retirement in-band ([Lesson 11](../02-rest-design/11-versioning-and-evolution.md)) |

### E. Client and network context

| Header | Notes |
|---|---|
| `Host` | Mandatory in HTTP/1.1; enables virtual hosting. **Never trust it for URL generation without an allowlist** — "Host header injection" poisons password-reset emails |
| `User-Agent` | Advisory. Don't branch business logic on it |
| `Origin` | Set by the browser on cross-origin requests. **Cannot be forged by JS** — which makes it usable for CSRF defence |
| `Referer` | The [sic] misspelling is permanent. Leaks URLs; control with `Referrer-Policy` |
| `X-Forwarded-For` / `X-Forwarded-Proto` / `Forwarded` | Original client IP/scheme through proxies. **Only trustworthy if your own proxy sets it** — clients can forge it and bypass IP rate limits. Configure `trust proxy` to a specific hop count, never `true` blindly |

### F. Security response headers (set these once, globally)

| Header | What it prevents |
|---|---|
| `Strict-Transport-Security: max-age=31536000; includeSubDomains` | Downgrade to HTTP / SSL-stripping |
| `X-Content-Type-Options: nosniff` | Browser MIME-sniffing your JSON as HTML and executing it |
| `Content-Security-Policy` | XSS payload execution (matters most for HTML, but set `default-src 'none'` on API responses) |
| `X-Frame-Options: DENY` / CSP `frame-ancestors` | Clickjacking |
| `Referrer-Policy: no-referrer` | URL leakage to third parties |
| `Cross-Origin-Resource-Policy: same-origin` | Side-channel resource inclusion |

> On an API, `helmet()` in Express (or Spring Security's defaults) gives you most of these in one line. Do it — it's free, and its absence is the first thing a security review flags.

---

## 6. The body: how data is actually encoded

Four request-body encodings exist in the wild. Knowing all four — and when each is right — is a common interview gap.

### 1. `application/json` — the default for APIs
```http
Content-Type: application/json
{"amount":4999,"currency":"usd"}
```
Nested structures, typed values, arrays. This is your default. Details and limitations in [Lesson 05](05-data-formats.md).

### 2. `application/x-www-form-urlencoded` — HTML form default
```http
Content-Type: application/x-www-form-urlencoded
amount=4999&currency=usd&metadata%5Border%5D=A17
```
Flat key–value, percent-encoded, `+` for space. **No types — everything is a string.** No native nesting (frameworks invent `a[b]=c` conventions, and they disagree with each other, which is a real source of bugs).

Why it still matters: it's what a plain HTML `<form>` sends, it's what **OAuth 2 token endpoints require** (a spec mandate that surprises people), and Stripe's API accepts it as a first-class alternative to JSON.

### 3. `multipart/form-data` — files + fields
```http
Content-Type: multipart/form-data; boundary=----X

----X
Content-Disposition: form-data; name="merchant_id"

mrc_9s2k
----X
Content-Disposition: form-data; name="document"; filename="invoice.pdf"
Content-Type: application/pdf

%PDF-1.4 ...binary...
----X--
```
The **only** standard way to send binary files alongside metadata in one request. Each part gets its own headers, separated by the `boundary` string declared in `Content-Type`.

Production rules for uploads — all four are common failures:
- **Never trust `filename`.** Path traversal (`../../etc/passwd`) and null bytes. Generate your own name; store the original only as a display label.
- **Never trust the part's `Content-Type`.** Sniff magic bytes. A `.jpg` claiming `image/jpeg` can be a polyglot HTML file that executes when served — hence `nosniff` and serving user content from a separate origin.
- **Cap size and part count** *before* buffering (`multer`'s `limits`), or a single request fills your disk or RAM.
- **Stream to storage; don't buffer in memory.** In Node, a buffered 500MB upload is an instant OOM.
- **Prefer pre-signed URLs at scale.** Have the client `PUT` straight to S3 with a short-lived signed URL; your API only issues the URL and records the result. This removes upload bandwidth, memory pressure and timeouts from your app entirely — and *"I'd use pre-signed URLs"* is the expected senior answer to "design a file upload API."

### 4. Binary / everything else
`application/octet-stream`, `application/pdf`, `image/png`, `application/x-protobuf`, `text/csv`. Raw bytes with a correct `Content-Type`. Use for single-file uploads or downloads where no metadata accompanies the bytes.

### Base64-in-JSON — the thing everyone does and mostly shouldn't
```json
{ "document": "JVBERi0xLjQK..." }
```
Costs **+33% size**, plus CPU on both ends, plus it must be fully buffered in memory (no streaming), plus it lands in your logs. Acceptable for genuinely small blobs (a 4KB signature image, a webhook's tiny attachment). For anything real, use `multipart` or a pre-signed URL.

---

## 7. Cookies vs `Authorization` — the decision, made properly

This is the single most argued-about API question in front-end work, so hold the reasoning, not a preference.

```http
Set-Cookie: session=abc123; HttpOnly; Secure; SameSite=Lax; Path=/; Max-Age=604800
```

| Attribute | Effect | Why it matters |
|---|---|---|
| `HttpOnly` | JS **cannot** read it | Removes token theft via XSS — the killer feature |
| `Secure` | HTTPS only | Prevents plaintext leak |
| `SameSite=Strict` | Never sent cross-site | Strongest CSRF defence; breaks inbound links to logged-in pages |
| `SameSite=Lax` *(browser default)* | Sent on top-level GET navigations only | The sane default |
| `SameSite=None` | Sent cross-site; **requires `Secure`** | Needed when your SPA and API are on different sites |
| `Domain` | Share across subdomains | Widening this widens your blast radius |
| `Path`, `Max-Age`/`Expires` | Scope and lifetime | Omit `Max-Age` → session cookie, dies with the browser |

**The trade-off table you should be able to draw:**

| | Cookie (`HttpOnly`) | `Authorization: Bearer` (in JS memory) | Bearer in `localStorage` |
|---|---|---|---|
| Vulnerable to **XSS** token theft | **No** | Partially (memory only, gone on reload) | **Yes — fully readable** |
| Vulnerable to **CSRF** | **Yes** — needs `SameSite` + CSRF token | No (attacker's page can't set your header) | No |
| Works for **native/mobile/server** clients | Awkward | **Yes** | Yes |
| Cross-origin setup | Fiddly (`SameSite=None`, `Allow-Credentials`, exact origin) | Simple | Simple |
| Survives page reload | Yes | No (needs a refresh flow) | Yes |

**What to actually do:**
- **Browser-only product, same site or subdomain:** `HttpOnly`, `Secure`, `SameSite=Lax` cookies. XSS is the more likely attack, and cookies are the only option that defends against it.
- **Public API for third parties, mobile apps, server-to-server:** `Authorization: Bearer`.
- **SPA on a different origin from the API:** either a `SameSite=None; Secure` cookie with strict CORS, or a short-lived access token in memory with a refresh token in an `HttpOnly` cookie. **That hybrid is the current best practice** and a great interview answer ([Lesson 13](../03-security/13-sessions-and-jwt.md)).
- **Never `localStorage`** for a long-lived token if you can avoid it. One XSS anywhere on your origin = total account compromise, with no expiry.

---

## 8. Custom headers vs body fields — when to use which

A design question that comes up in reviews constantly.

**Put it in a header when it is metadata about the *transmission*, applies uniformly to every endpoint, or must be readable without parsing the body:**
`Authorization`, `Idempotency-Key`, `X-Request-Id`, `traceparent`, `Accept`, `If-Match`, `Stripe-Version`, `X-Tenant-Id`.

Why headers win for these: a load balancer, WAF or gateway can route/limit/authorise on them **without buffering and parsing the body** — which is both faster and architecturally necessary at the edge.

**Put it in the body when it's part of the domain data:** amounts, IDs, statuses, filters for a `POST /search`.

**Rules for custom headers:**
- Don't add the `X-` prefix on new ones (RFC 6648), but don't rename existing conventions like `X-Request-Id` either.
- Namespace them: `Ledger-Api-Version`, not `Version`.
- **Cross-origin responses:** custom headers are invisible to browser JS unless you add `Access-Control-Expose-Headers`. This is the #1 reason "my rate-limit headers are missing in the browser."
- Keep them small. Under HTTP/1.1 they're re-sent uncompressed on every request; a fat 4KB header block on 200 requests/page is real bandwidth.
- **Never put a secret in a header you also log.** Redact `Authorization` and `Cookie` in your logging middleware, deliberately, with a test.

---

## 9. Security implications you must know

| Attack | Mechanism | Defence |
|---|---|---|
| **Header injection / response splitting** | Unvalidated `\r\n` in a value you echo (e.g. into `Location`) forges headers or a second response | Never build headers from raw user input; modern frameworks reject CRLF — don't hand-roll |
| **Request smuggling** | Proxy and origin disagree on `Content-Length` vs `Transfer-Encoding` | Consistent HTTP versions end-to-end; don't write your own parser |
| **Host header injection** | You use `Host` to build password-reset links → link points to the attacker | Allowlist expected hosts |
| **`X-Forwarded-For` spoofing** | Client sends a fake IP; your IP rate limit and audit log both lie | Trust only the hop your proxy adds (`trust proxy: 1`) |
| **Cache poisoning via missing `Vary`** | Shared cache serves user A's authorised response to user B | `Vary` on every request header you branch on; `Cache-Control: private` for per-user data |
| **CORS misconfiguration** | Reflecting arbitrary `Origin` with `Allow-Credentials: true` | Explicit allowlist. Never reflect blindly, never `*` with credentials |
| **Information disclosure** | `Server: nginx/1.18.0`, `X-Powered-By: Express`, stack traces in 500 bodies | Strip them; generic error bodies |
| **Token leakage** | Token in a URL → access logs, `Referer`, browser history, CDN logs | Tokens go in headers, only |
| **BREACH** | Compressing a response containing a secret + attacker-influenced text | Don't compress responses with CSRF tokens; keep secrets out of compressed bodies |

---

## 10. Production rules

| Rule | Why |
|---|---|
| **Always set `Content-Type` on responses with bodies** | Without it, browsers sniff, and sniffing is a vulnerability |
| **Validate incoming `Content-Type`; `415` if unsupported** | Otherwise clients silently send form-encoded data to a JSON endpoint and get confusing 400s |
| **Bound body size explicitly** | `express.json({ limit: "100kb" })`. Unbounded = trivial DoS |
| **Return `X-Request-Id` on every response, including errors** | The only way to correlate a customer report with your logs |
| **Expose rate-limit headers** | A client that can self-throttle doesn't need to be blocked |
| **`Vary` on anything you negotiate** | Prevents cross-user cache leaks |
| **Redact `Authorization`, `Cookie`, `Set-Cookie` and `Idempotency-Key` in logs** | With a unit test asserting it, because this regresses silently |
| **Strip `Server` / `X-Powered-By`** | Free recon for attackers |
| **Never accept auth via query string** | Even "just for websockets/downloads" — use a short-lived single-use token instead |
| **Compress text responses; skip already-compressed ones** | 60–90% saving on JSON; pure CPU waste on JPEG/ZIP |
| **Prefer pre-signed URLs over proxying large uploads** | Removes bandwidth, memory and timeout risk from your app |

---

## 11. Interview traps

**Q1. "What's the difference between a header and a payload?"**
> *"Headers are metadata about the message — how to interpret the body, who's calling, how it may be cached, how to correlate it. The payload is the body plus the headers that describe it; the current spec calls the decoded data 'content'. The practical point is that the operational contract — auth, caching, concurrency, idempotency, tracing — lives in headers, not in the body."*

**Q2. "Where do you put an auth token: header, cookie, or query string?"**
Header or cookie; **never** a query string, because URLs are logged at every hop and leak via `Referer` and history. Then give the decision rule from §7 — browser-only → `HttpOnly` cookie; third-party/mobile → Bearer; cross-origin SPA → access token in memory + refresh token in an `HttpOnly` cookie. Naming the *threat* each choice addresses (XSS vs CSRF) is what makes the answer senior.

**Q3. "Is Basic auth encrypted?"**
No. Base64 is encoding — reversible in one line. It's only as safe as the TLS underneath.

**Q4. "`Accept` vs `Content-Type`?"**
`Accept` = what I want back. `Content-Type` = what this body is. Both can appear in one request, in opposite directions.

**Q5. "What does `Vary` do and why does it matter?"**
Tells caches which request headers change the response. Omit it while varying on `Authorization` or `Origin` and a shared cache will serve one user's data to another. This is a data-leak bug, not a performance bug.

**Q6. "How do you upload a 2GB file to an API?"**
Not through your API. Pre-signed URL directly to object storage (multipart upload for resumability), then a callback/webhook to record completion. Reasons: no request timeouts, no memory pressure, no bandwidth bill on your app tier, and resumability. If you must proxy it: `multipart/form-data`, streamed to storage, with size limits — never buffered.

**Q7. "Why not put everything in the body, including auth and idempotency keys?"**
Because intermediaries (gateway, LB, WAF, CDN) must act on that data **without parsing the body**, which they often can't do at all (streaming, compression, size). Also `GET`s have no body, so any header-only mechanism must work uniformly.

**Q8. "Client sends `Content-Type: text/plain` with a JSON body. What do you do?"**
`415 Unsupported Media Type`. Do **not** be helpful and parse it anyway — permissive parsing is how content-type-confusion and CSRF bypasses happen (recall from Lesson 02 that `text/plain` avoids a CORS preflight, so accepting it can open a cross-site write path). Strict `Content-Type` checking is genuinely a security control.

**Q9. "How does the server know where the body ends?"**
`Content-Length`, or `Transfer-Encoding: chunked`, or (HTTP/2+) frame lengths. Bonus: mention that ambiguity between the first two is the request-smuggling class.

**Q10. "What's the maximum size of a header, and what happens if you exceed it?"**
No spec limit; servers set one (~8KB typical). Exceeding it gives `431`. Real cause: cookie bloat, or a JWT with too many claims stuffed into a header. Which is a decent argument for keeping JWTs small ([Lesson 13](../03-security/13-sessions-and-jwt.md)).

---

## 12. Build & break

### Build — headers middleware you'd actually ship
```ts
import type { Request, Response, NextFunction } from "express";
import { randomUUID } from "node:crypto";

/** Correlation: first middleware in the chain, so every log line has it. */
export function requestId(req: Request, res: Response, next: NextFunction) {
  // Accept an inbound id (from the gateway) but never trust its format.
  const inbound = req.header("x-request-id");
  const id = inbound && /^[\w-]{1,64}$/.test(inbound) ? inbound : randomUUID();
  (req as any).id = id;
  res.setHeader("X-Request-Id", id);                    // on EVERY response, errors included
  next();
}

/** Strict content-type checking: a security control, not pedantry. */
export function requireJson(req: Request, res: Response, next: NextFunction) {
  if (!["POST", "PUT", "PATCH"].includes(req.method)) return next();
  if (req.header("content-length") === "0") return next();

  const ct = req.header("content-type")?.split(";")[0].trim().toLowerCase();
  if (ct !== "application/json") {
    return res.status(415).json({
      type: "https://ledger.dev/errors/unsupported_media_type",
      title: "Unsupported Media Type",
      status: 415,
      detail: `Expected application/json, received ${ct ?? "none"}`,
    });
  }
  next();
}

/** Never log a credential. Assert this with a test. */
const REDACT = new Set(["authorization", "cookie", "set-cookie", "x-api-key", "idempotency-key"]);
export function safeHeaders(h: Record<string, unknown>) {
  return Object.fromEntries(
    Object.entries(h).map(([k, v]) => [k, REDACT.has(k.toLowerCase()) ? "[REDACTED]" : v]),
  );
}

/** Make custom response headers visible to browser JS. */
export const corsOptions = {
  origin: ["https://app.ledger.dev"],                   // allowlist, never reflect
  credentials: true,
  exposedHeaders: ["X-Request-Id", "RateLimit-Remaining", "RateLimit-Reset", "ETag"],
  maxAge: 86_400,                                      // cache the preflight
};
```

Then, globally: `app.use(helmet())`, `app.disable("x-powered-by")`, `app.set("trust proxy", 1)`.

### Break — six experiments
```bash
# 1. See every header both ways
curl -v https://api.github.com/users/torvalds 2>&1 | grep -E "^[<>]"

# 2. Negotiate a different representation of the same resource
curl -s -H "Accept: application/vnd.github.v3.raw"  https://api.github.com/repos/facebook/react/readme | head -3
curl -s -H "Accept: application/vnd.github.v3.html" https://api.github.com/repos/facebook/react/readme | head -3

# 3. Wrong Content-Type → does the API 415 or silently accept?
curl -i -X POST https://httpbin.org/post -H "Content-Type: text/plain" -d '{"a":1}'

# 4. Watch multipart framing on the wire (note the boundary)
curl -v -F "merchant_id=mrc_1" -F "document=@README.md" https://httpbin.org/post 2>&1 | head -30

# 5. Compression: compare bytes
curl -s -o NUL -w "identity %{size_download}\n" -H "Accept-Encoding: identity" https://api.github.com/repos/facebook/react
curl -s -o NUL -w "br/gzip  %{size_download}\n" -H "Accept-Encoding: br,gzip"  https://api.github.com/repos/facebook/react

# 6. Blow past the header limit → 431
curl -s -o NUL -w "%{http_code}\n" -H "X-Big: $(head -c 20000 /dev/urandom | base64 -w0)" https://httpbin.org/get
```

### Break your own server — four failures to feel
1. **Remove `Content-Type` from a JSON response.** Watch a browser render it as text — or, if the body starts with `<`, sniff it as HTML. Now add `nosniff` and see the difference.
2. **Log the whole request headers object.** Find your own Bearer token in the log file. Delete the log, add `safeHeaders`, and write the test that keeps it out.
3. **Omit `Access-Control-Expose-Headers`.** Read `X-Request-Id` from browser JS → `null`. Same call in curl → present. That's the confusion, solved permanently.
4. **Remove the `express.json` limit** and POST a 200MB body. Watch RSS climb. Put the limit back.

### Explain out loud (90 seconds)
1. The four parts of an HTTP message and what the blank line is for.
2. Payload vs body vs content.
3. Two ways framing works, and the vulnerability the ambiguity enables.
4. Cookie vs Bearer, and the attack each one is defending against.

---

## What's next

You know how a body is framed, described and negotiated. Next: *what to put in it* — the real argument between JSON, XML, Protobuf, MessagePack and CSV, with numbers, plus JSON's five genuine limitations (including the one that silently corrupts money and IDs, which you'll hit in Ledger).

Next → **[Lesson 05: JSON vs XML vs Protobuf vs the rest](05-data-formats.md)**
