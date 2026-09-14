# Lesson 12 — The authentication landscape

> **Why this lesson exists:** most engineers know exactly one auth mechanism — whichever their current job uses — and assume it's *the* way. Then an interviewer asks *"how would you authenticate a server-to-server call from a bank?"* and JWT isn't the answer. This lesson gives you the whole map, the decision rule, and the failure mode of each option, so you can pick rather than default.

**Time:** ~80 minutes · **Prereq:** Lessons 04, 11

---

## 1. The idea in one sentence

> **Authentication answers "who is calling?" — and the right mechanism is determined almost entirely by *what kind of client* is calling, not by what's fashionable.**

Three words you must never conflate, because interviewers test exactly this:

| Term | Question | HTTP status when it fails |
|---|---|---|
| **Authentication** (authn) | *Who are you?* | **401** |
| **Authorization** (authz) | *May you do this?* | **403** |
| **Accounting / audit** | *What did you do?* | — (logging) |

A system can have perfect authentication and be completely broken, because **authorization is where APIs actually get breached** ([Lesson 15](15-authorization-and-multitenancy.md)). Keep them separate in your head and in your code.

---

## 2. The seven mechanisms

### 1. API keys

```http
GET /v1/payments
Authorization: Bearer sk_live_51H8xK2mR...
# or the non-standard but common:
X-API-Key: sk_live_51H8xK2mR...
```

A long random secret identifying an *application* (not a user). The workhorse of server-to-server public APIs — Stripe, SendGrid, OpenAI, Twilio.

| Pros | Cons |
|---|---|
| Dead simple to issue, use, rotate, revoke | Long-lived: a leak is durable access |
| Revocation is one DB row | No user identity, only app identity |
| Easy to scope and rate-limit per key | Sent on every request, so it's in every log if you're careless |
| No refresh dance | Not usable from a browser (you'd expose it in JS) |

**How to build them properly** — this is a genuinely good interview topic because most people get it wrong:

```ts
import { randomBytes, createHash, timingSafeEqual } from "node:crypto";

/** Generate: a public prefix for lookup + a secret half never stored in plaintext. */
export function generateApiKey(env: "live" | "test") {
  const id     = randomBytes(8).toString("base64url");    // lookup id, stored plainly
  const secret = randomBytes(32).toString("base64url");   // 256 bits of entropy
  const plaintext = `sk_${env}_${id}_${secret}`;          // shown to the user ONCE
  const hash = createHash("sha256").update(secret).digest("hex");
  return { plaintext, id, hash, last4: secret.slice(-4) };
}

/** Verify: single indexed lookup by id, then constant-time compare. */
export async function verifyApiKey(header: string | undefined) {
  if (!header?.startsWith("Bearer ")) return null;
  const token = header.slice(7);
  const m = /^sk_(live|test)_([\w-]{11})_([\w-]{43})$/.exec(token);
  if (!m) return null;                                    // fail fast on shape

  const [, env, id, secret] = m;
  const row = await db.apiKeys.findByPublicId(id);        // O(1), indexed
  if (!row || row.revokedAt) return null;

  const expected = Buffer.from(row.hash, "hex");
  const actual   = createHash("sha256").update(secret).digest();
  if (expected.length !== actual.length) return null;
  if (!timingSafeEqual(expected, actual)) return null;    // no timing oracle

  await db.apiKeys.touch(row.id);                         // last_used_at, async is fine
  return { merchantId: row.merchantId, scopes: row.scopes, env };
}
```

The five design decisions in that snippet, each of which is an interview answer:

1. **Store a hash, never the key.** A database leak must not hand over live credentials. Since the key has 256 bits of entropy, **SHA-256 is correct here** — you do *not* need bcrypt/argon2. Those exist to slow down brute force against *low-entropy human passwords*; a random 256-bit key cannot be brute-forced, and bcrypt on every API request would wreck your latency. **Knowing why the answer differs from passwords is the real signal.**
2. **Split the key into a public lookup id and a secret.** Without it, verifying means comparing against *every* hashed key in the table — O(n) per request.
3. **Prefix it** (`sk_live_`) so it's identifiable in logs and by secret scanners. GitHub, AWS and Stripe all do this, and GitHub's secret-scanning partner program literally relies on recognisable prefixes to notify you when your key hits a public repo.
4. **Constant-time comparison.** `===` on secrets leaks length and prefix information through timing. Marginal in practice over a network, free to do right.
5. **Show it once.** No "reveal key" button, because that turns your database into a plaintext credential store. Show a `last4` for identification.

**Also non-negotiable:** separate `test`/`live` keys (so a staging bug can't move real money), per-key scopes, `last_used_at` for detecting stale keys, and one-click revocation.

### 2. HTTP Basic

```http
Authorization: Basic bWVyY2hhbnQ6czNjcjN0
```
`base64(user:password)`. **Base64 is encoding, not encryption** — one line to reverse. Only acceptable over TLS.

Legitimate uses today: internal tooling, `/metrics` endpoints, and — notably — as an **API key transport**: Stripe accepts `Authorization: Basic base64(sk_live_...:)` because every HTTP client on earth supports Basic, so `curl -u sk_live_xxx:` just works. That's a real, thoughtful use.

Never use it with a *user's* password on a public API: it means the client stores the password, so you can't do MFA, can't revoke without a password change, and every client is a password-handling liability.

### 3. Session cookies

```http
POST /login → Set-Cookie: sid=abc; HttpOnly; Secure; SameSite=Lax; Max-Age=604800
GET /me     → Cookie: sid=abc
```
The browser's native mechanism, and **still the right default for first-party web apps**. The server keeps a session record (Redis/Postgres); the cookie is an opaque pointer.

| Pros | Cons |
|---|---|
| `HttpOnly` means **XSS can't steal it** — the single biggest security win available | Needs CSRF protection (`SameSite` + token) |
| **Instant revocation:** delete the row | Requires a session store (a dependency, and a lookup per request) |
| Small (an opaque ID, not a payload) | Awkward for mobile/native and cross-origin |
| Browser handles storage, expiry and sending | Not "stateless" in the purist sense |

The comeback story worth knowing: the industry went all-in on JWT around 2015–2018, then largely walked back for first-party web apps once the revocation and XSS problems became lived experience. **"Sessions are underrated" is a defensible, current, senior position** — and one that surprises interviewers who expect JWT enthusiasm.

### 4. Bearer tokens / JWT

```http
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9.eyJzdWIiOiJ1c2VyXzEiLCJleHAiOjE3NzM...
```
A self-contained, signed token: the server verifies a signature instead of looking anything up. Full treatment in [Lesson 13](13-sessions-and-jwt.md). Right for: mobile/native clients, cross-service calls, and anything where you want to avoid a session lookup per request.

### 5. OAuth 2.1 / OIDC

Not an authentication mechanism so much as a **delegation framework**: it lets a user grant a *third party* limited access to their data without sharing their password. If your answer to "how do I let another company's app read my users' payments?" is anything other than OAuth, it's wrong. [Lesson 14](14-oauth2-and-oidc.md).

### 6. mTLS (mutual TLS)

Both sides present certificates during the TLS handshake (Lesson 02, stage 8). The client's identity is cryptographically proven *before any HTTP is exchanged*.

| Pros | Cons |
|---|---|
| Strongest option; no bearer secret to steal or replay | Certificate lifecycle management is genuinely hard |
| Authentication happens at the transport layer | Painful for browsers; near-impossible for casual developers |
| Standard for banking, PSD2/open banking, and healthcare | Needs a PKI, rotation, revocation (CRL/OCSP) |

**This is the answer to "how would a bank authenticate to your API?"** — mTLS, often combined with request signing. Inside a service mesh (Istio/Linkerd), mTLS between every pod is the default and is handled by sidecars, which is why "zero-trust networking" is achievable at all. Knowing it exists and *when* it's chosen is what's being tested.

### 7. HMAC request signing

The client signs the request itself with a shared secret; the server recomputes the signature.

```http
POST /v1/payments
X-Ledger-Timestamp: 1773480600
X-Ledger-Signature: t=1773480600,v1=5257a869e7ecebeda32affa62cdca3fa...
```
```ts
const signed = `${timestamp}.${rawBody}`;                       // bind time AND body
const sig = createHmac("sha256", secret).update(signed).digest("hex");
```

What this buys you over a bearer token:

| Property | Bearer token | HMAC signature |
|---|---|---|
| Stolen credential replayable? | **Yes**, until expiry | Only within the timestamp window, and only that exact request |
| Body tamper-evident? | No | **Yes** — the body is inside the signature |
| Secret sent over the wire? | **Yes, every request** | **No** — only a derived signature |
| Works if TLS is terminated/inspected? | Secret exposed to the proxy | Still safe |

**Used by:** AWS (SigV4 — this is why AWS API calls are so fiddly to hand-craft), and **every webhook system on earth**, including Stripe's and GitHub's. Which is the key insight for [Lesson 23](../05-beyond-rest/23-realtime-and-webhooks.md): **when you send a webhook, you are the client, and HMAC is how the receiver knows it's really you.**

The three implementation rules:
1. **Include a timestamp in the signed payload and reject anything older than ~5 minutes** — otherwise a captured request is replayable forever.
2. **Sign the raw body bytes, not the parsed-and-re-serialised object.** Key reordering or whitespace changes break the signature. In Express this means capturing the raw body *before* `express.json()` parses it — a classic gotcha.
3. **Compare with `timingSafeEqual`**, and accept multiple valid signatures during a secret rotation.

---

## 3. The decision table

This is the deliverable of the lesson. Learn it, and *"which auth would you use?"* becomes a lookup.

| Client type | Mechanism | Why |
|---|---|---|
| **Your own web SPA, same site** | `HttpOnly` session cookie (or cookie-held refresh + in-memory access token) | XSS-proof storage; instant revocation |
| **Your own mobile app** | OAuth 2.1 auth code + PKCE → access + refresh tokens | No client secret can be kept in an app binary |
| **Third-party app acting for a user** | **OAuth 2.1 authorization code + PKCE** | The user must consent; you must be able to revoke per app |
| **Third-party server, its own data** | API key (or OAuth client credentials) | No user involved |
| **Server-to-server, high security (bank, health)** | **mTLS**, often + request signing | Cryptographic identity, no replayable bearer |
| **Internal service-to-service** | mTLS via service mesh, or short-lived service JWTs | Automated rotation, no human secrets |
| **CLI tool / script** | API key, or OAuth device code flow | Device flow when a user identity is needed and there's no browser |
| **Webhook you send** | **HMAC signature over timestamp + raw body** | The receiver must verify origin *and* integrity |
| **IoT / embedded** | mTLS with per-device certs, or pre-shared keys | Devices can't do interactive flows |
| **Public read-only data** | None, but rate-limit by IP | Don't add auth you don't need |

> **The interview move:** when asked "how would you authenticate X?", *first ask what kind of client it is.* Then answer from this table with the reason. Candidates who lead with "JWT" for everything reveal that they have one hammer.

---

## 4. Cross-cutting rules that apply to every mechanism

### Credential handling
| Rule | Why |
|---|---|
| **TLS only. Always. Everywhere** | Every mechanism above is trivially defeated on plaintext HTTP |
| **Never in a URL** — path or query | URLs are logged at every hop, leak via `Referer`, land in history (Lesson 02) |
| **Redact `Authorization`/`Cookie` in logs, with a test** | This regresses silently |
| **Hash stored credentials** — SHA-256 for high-entropy keys, argon2id/bcrypt for passwords | Different threat models, different tools |
| **Constant-time comparison for every secret** | Avoid timing oracles |
| **Show a secret exactly once** | No "reveal" button |
| **Expiry on everything, with rotation support** | A credential with no expiry is a permanent liability |
| **Scan for leaked secrets in CI** (gitleaks, trufflehog) | Because someone will commit one |

### Rate limiting the auth endpoints specifically
The login/token endpoint is the most attacked endpoint you have. It needs its own, much stricter limits (see [Lesson 19](../04-production/19-rate-limiting.md)):
- Per-IP **and** per-account limits (per-IP alone is defeated by botnets; per-account alone enables lockout-as-DoS)
- Exponential backoff after failures
- **Never lock an account permanently on failed attempts** — that's a denial-of-service someone can trigger against your users deliberately
- Watch for **credential stuffing**: one attempt per account across thousands of accounts stays under per-account limits entirely, so you also need a global anomaly signal

### The four failure modes to design against
| Failure | Defence |
|---|---|
| **Credential theft** | Short lifetimes, `HttpOnly`, no localStorage for long-lived tokens, rotation |
| **Replay** | Timestamps + nonces (HMAC), short expiry (JWT), TLS |
| **Brute force** | High entropy, rate limits, backoff, MFA |
| **Confused deputy** | The dangerous one: your service holds a powerful credential and is tricked into using it on someone's behalf. Defence: scope credentials narrowly, and **always re-check authorization against the *end user*, not just your service identity** |

### Timing-safe verification, and the enumeration subtlety
```ts
// ❌ Leaks whether the user exists, via both response and timing
const user = await db.users.findByEmail(email);
if (!user) return fail("no such user");
if (!await bcrypt.compare(password, user.hash)) return fail("wrong password");

// ✅ Same message, and comparable work either way
const user = await db.users.findByEmail(email);
const hash = user?.passwordHash ?? DUMMY_HASH;    // a real bcrypt hash of random data
const ok = await bcrypt.compare(password, hash);   // always runs, so timing matches
if (!ok || !user) return fail("invalid_credentials");   // one generic message
```
The `DUMMY_HASH` trick is worth remembering: without it, a nonexistent user returns in 2ms and a real one in 200ms, which is a perfectly usable enumeration oracle.

---

## 5. Ledger's auth design (the worked answer)

Ledger has four distinct client types, so it needs four mechanisms — and being able to justify *four* rather than forcing one is the point.

| Client | Mechanism | Details |
|---|---|---|
| **Merchant dashboard** (the React Console) | Session: access token in memory + refresh token in `HttpOnly`, `Secure`, `SameSite=Lax` cookie | Best XSS/CSRF balance for a first-party SPA ([Lesson 13](13-sessions-and-jwt.md)) |
| **Merchant server integrations** | API keys, `sk_live_*` / `sk_test_*`, scoped, hashed, prefixed | Simple, revocable, rate-limited per key |
| **Third-party platforms** building on Ledger | OAuth 2.1 auth code + PKCE, scoped access tokens | User consent + per-app revocation ([Lesson 14](14-oauth2-and-oidc.md)) |
| **Outbound webhooks** | HMAC-SHA256 over `timestamp.rawBody`, with a rotation window | Receiver verifies origin and integrity ([Lesson 23](../05-beyond-rest/23-realtime-and-webhooks.md)) |

The unified middleware — note that **it resolves an identity and then stops**; authorization is a separate, later step:

```ts
export type Principal =
  | { kind: "api_key";  merchantId: string; keyId: string; scopes: string[]; env: "live"|"test" }
  | { kind: "user";     merchantId: string; userId: string; role: Role }
  | { kind: "oauth";    merchantId: string; appId: string;  scopes: string[] };

export async function authenticate(req: Request, res: Response, next: NextFunction) {
  const header = req.header("authorization");
  const cookie = req.cookies?.session;

  let principal: Principal | null = null;

  if (header?.startsWith("Bearer sk_")) {
    principal = await fromApiKey(header);
  } else if (header?.startsWith("Bearer ")) {
    principal = await fromAccessToken(header.slice(7));    // JWT or OAuth token
  } else if (cookie) {
    principal = await fromSession(cookie);
  }

  if (!principal) {
    res.set("WWW-Authenticate", 'Bearer realm="ledger", error="invalid_token"');
    throw ApiError.unauthorized("missing_or_invalid_credentials",
      "Provide a valid API key, access token, or session");
  }

  // Environment isolation: a test key must never touch live data.
  req.principal  = principal;
  req.merchantId = principal.merchantId;      // ← tenancy, from the CREDENTIAL, never the body
  req.env = principal.kind === "api_key" ? principal.env : "live";
  next();
}
```

**The most important line in that function is `req.merchantId = principal.merchantId`.** Tenancy is derived from the credential and nowhere else. Every query downstream scopes by it. That single discipline prevents the entire BOLA class ([Lesson 15](15-authorization-and-multitenancy.md)).

---

## 6. Interview traps

**Q1. "Difference between authentication and authorization?"**
Who you are (401) vs what you may do (403). Then the point that matters: *"and authorization is where APIs actually get breached — OWASP's #1 API risk is broken object-level authorization, not broken authentication."*

**Q2. "How would you design API keys?"**
The five decisions from §2.1: hash-not-store, public-id + secret split for O(1) lookup, identifiable prefix, constant-time compare, show-once. Plus scopes, test/live separation, `last_used_at`, revocation.

**Q3. "Should you bcrypt an API key?"**
**No** — and this is a great question because the naïve answer is yes. bcrypt/argon2 exist to slow brute force against low-entropy human passwords. A 256-bit random key can't be brute-forced, so SHA-256 is sufficient, and bcrypt would add tens of milliseconds to *every API request*. Different threat model, different tool.

**Q4. "Is Basic auth secure?"**
Only as secure as its TLS — base64 is reversible. Acceptable as an API-key transport (Stripe does this for `curl` ergonomics); never for a user's password on a public API.

**Q5. "How does a bank authenticate to your API?"**
mTLS, typically with request signing on top, IP allowlisting, and a formal certificate lifecycle. Then explain *why*: no replayable bearer credential, identity proven at the transport layer, and it satisfies regulatory requirements (PSD2 open banking mandates it).

**Q6. "How does a webhook receiver know the request came from you?"**
HMAC signature over `timestamp + raw body`, verified with a shared secret, with a ±5 minute tolerance to prevent replay, using constant-time comparison, and supporting two valid secrets during rotation. Then the gotcha: **you must sign the raw bytes**, so the receiver must capture the body before JSON parsing.

**Q7. "Why not just use JWT for everything?"**
Because revocation. And because a browser has nowhere safe to store one, a bank won't accept a replayable bearer token, and an IoT device can't run a refresh flow. *"JWT is right when you want stateless verification across services and can accept a short revocation window. It's the wrong default for a first-party web session."*

**Q8. "Where do you store the token in a browser?"**
Neither `localStorage` (XSS reads it) nor `sessionStorage`. Either an `HttpOnly` cookie, or an access token in JS memory with a refresh token in an `HttpOnly` cookie. Name the threat each choice addresses ([Lesson 13](13-sessions-and-jwt.md)).

**Q9. "How do you handle API key rotation without downtime?"**
Support **N valid keys per account simultaneously**. Create the new key → deploy it to the client → verify traffic on the new key via `last_used_at` → revoke the old one. Never a hard cutover. Same pattern for HMAC secrets: accept both, verify against each, retire the old after the window.

**Q10. "A key leaked into a public GitHub repo. Walk me through the response."**
1. **Revoke immediately** — don't wait to assess.
2. Issue a replacement and get it to the customer.
3. **Audit everything that key did**, from your logs, which is only possible if you log `key_id` per request.
4. Notify the customer (and, depending on data touched, your compliance/legal path).
5. Prevention: secret scanning in CI, GitHub's push protection, prefixed keys so scanners recognise them, and short-lived credentials where possible.
The step people forget is **3**, and it's the one that requires you to have designed for it in advance.

---

## 7. Build & break

### Build — the API key system, end to end
Implement `scratch/api-keys.ts` with: `generateApiKey`, `verifyApiKey`, `revokeKey`, `listKeys` (returning `last4` and `last_used_at`, never the key). Then a middleware that populates `req.principal`.

Tests you must write — each maps to a real attack:
```ts
test("rejects a malformed key without touching the DB", ...);        // fail fast
test("rejects a revoked key", ...);
test("rejects a key whose secret half is wrong but id is right", ...); // the O(1) lookup path
test("uses constant-time comparison", ...);                            // assert timingSafeEqual is called
test("never returns the plaintext key after creation", ...);
test("a test-mode key cannot read live-mode data", ...);               // env isolation
test("logs redact the Authorization header", ...);
```

### Build — HMAC signing both directions
```ts
import { createHmac, timingSafeEqual } from "node:crypto";

export function sign(rawBody: string, secret: string, ts = Math.floor(Date.now()/1000)) {
  const v1 = createHmac("sha256", secret).update(`${ts}.${rawBody}`).digest("hex");
  return `t=${ts},v1=${v1}`;
}

export function verify(header: string, rawBody: string, secrets: string[], toleranceSec = 300) {
  const parts = Object.fromEntries(header.split(",").map(p => p.split("=") as [string,string]));
  const ts = Number(parts.t);
  if (!Number.isFinite(ts)) return false;
  if (Math.abs(Date.now()/1000 - ts) > toleranceSec) return false;   // replay window

  const provided = Buffer.from(parts.v1 ?? "", "hex");
  // Accept ANY current secret, so rotation has no downtime.
  return secrets.some(s => {
    const expected = createHmac("sha256", s).update(`${ts}.${rawBody}`).digest();
    return expected.length === provided.length && timingSafeEqual(expected, provided);
  });
}
```
Then wire it into Express **correctly** — this is the part that bites everyone:
```ts
// Capture raw body BEFORE parsing, or your signature will never match.
app.use("/webhooks", express.raw({ type: "application/json", limit: "1mb" }));
app.post("/webhooks/ledger", (req, res) => {
  const raw = req.body.toString("utf8");
  if (!verify(req.header("x-ledger-signature") ?? "", raw, [CURRENT, PREVIOUS]))
    return res.status(400).json({ code: "invalid_signature" });
  const event = JSON.parse(raw);
  // ...
});
```

### Break — five experiments
1. **Timing oracle.** Verify a key with `===` and time 10,000 requests with a correct vs incorrect first character. Plot it. Then switch to `timingSafeEqual` and watch the difference vanish.
2. **O(n) key lookup.** Implement verification by hashing the input and comparing against every row. Seed 50,000 keys and measure p99. Now add the public-id split and measure again.
3. **Replay a signed request.** Capture a valid HMAC-signed request and replay it an hour later. It succeeds. Add the timestamp tolerance and watch it fail.
4. **Break the signature by parsing first.** Verify the HMAC against `JSON.stringify(req.body)` instead of the raw bytes. Send a body with keys in a different order or with extra whitespace. Watch a legitimate request fail verification — and understand why every webhook doc screams about raw bodies.
5. **Enumerate users.** Build a login endpoint without the `DUMMY_HASH` trick and time responses for a known vs unknown email. The gap is your oracle.

### Explain out loud (2 minutes)
1. Authn vs authz, and which one actually gets breached.
2. The seven mechanisms, one sentence each.
3. The decision table, from client type to mechanism.
4. Why API keys get SHA-256 and passwords get argon2id.
5. The three rules of HMAC signing.

---

## What's next

You've picked a mechanism. Now the deep dive on the two that matter most for user-facing apps: sessions and JWTs — what a JWT actually contains, how to verify one correctly (including the attacks on naïve verification), the honest revocation trade-off, and how to build access + refresh with rotation and theft detection.

Next → **[Lesson 13: Sessions, JWT & token lifecycle](13-sessions-and-jwt.md)**
