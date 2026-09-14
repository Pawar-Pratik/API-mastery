# Lesson 13 — Sessions, JWT & token lifecycle

> **Why this lesson exists:** JWT is the most-used and most-misused technology in modern auth. Engineers adopt it for "statelessness," discover they can't log anyone out, and bolt on a database lookup — arriving at a slower session system with worse ergonomics. This lesson gives you the mechanism, the four real attacks on naïve verification, the honest revocation trade-off, and a refresh-rotation design with theft detection that you could ship.

**Time:** ~95 minutes · **Prereq:** Lesson 12

---

## 1. The idea in one sentence

> **A session is a *reference* to server-held state; a JWT *is* the state, signed — and every difference between them follows from that one distinction, including the fact that you cannot un-issue something you don't store.**

---

## 2. Anatomy of a JWT

```
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6IjIwMjYtMDMifQ.eyJzdWIiOiJ1c3JfOXMyayIsIm1yYyI6Im1yY18xIiwicm9sZSI6ImFkbWluIiwiaWF0IjoxNzczNDgwNjAwLCJleHAiOjE3NzM0ODE1MDAsImlzcyI6Imh0dHBzOi8vYXV0aC5sZWRnZXIuZGV2IiwiYXVkIjoiaHR0cHM6Ly9hcGkubGVkZ2VyLmRldiIsImp0aSI6IjAxSFE4WksifQ.SflKxwRJSMeKKF2QT4...
└──────────── header ────────────┘ └──────────────────── payload ────────────────────┘ └── signature ──┘
```

Three base64url segments joined by dots.

```json
// header — how to verify
{ "alg": "RS256", "typ": "JWT", "kid": "2026-03" }

// payload — the claims
{
  "sub": "usr_9s2k",                    // subject: who
  "mrc": "mrc_1",                       // custom: which tenant
  "role": "admin",                      // custom: authorization data
  "iat": 1773480600,                    // issued at
  "exp": 1773481500,                    // expires (15 min)
  "iss": "https://auth.ledger.dev",     // issuer
  "aud": "https://api.ledger.dev",      // audience: who it's FOR
  "jti": "01HQ8ZK"                      // unique token id (for denylisting)
}
```

**The single most important fact about a JWT, and the one people get wrong:**

> **A JWT is signed, not encrypted. The payload is base64url — plainly readable by anyone holding the token.**

Paste any JWT into `jwt.io` (or `atob` the middle segment) and read it. So:
- **Never put a secret, PII you wouldn't log, or anything sensitive in a JWT payload.**
- The signature proves *integrity and origin* (nobody tampered, it came from the issuer) — **not confidentiality**.
- If you genuinely need an encrypted token, that's JWE, not JWS. It exists and is rarely used; know the name.

### The registered claims worth knowing

| Claim | Meaning | Verify it? |
|---|---|---|
| `iss` | Issuer | **Yes** — reject tokens from other issuers |
| `sub` | Subject (the principal) | Yes |
| `aud` | Audience — who this token is *for* | **Yes** — prevents a token for service A being replayed at service B |
| `exp` | Expiry | **Yes**, always |
| `nbf` | Not before | Yes if present |
| `iat` | Issued at | Useful for "max age" policies |
| `jti` | Unique ID | Needed for denylisting and replay detection |

### Algorithms

| Alg | Type | Key | Use when |
|---|---|---|---|
| `HS256` | Symmetric (HMAC) | One shared secret signs *and* verifies | Single service that both issues and verifies |
| `RS256` | Asymmetric (RSA) | Private signs, **public** verifies | **Multiple services verify; only the auth service issues** |
| `ES256` | Asymmetric (ECDSA) | Same, smaller/faster | Same as RS256, preferred for size |
| `EdDSA` | Asymmetric (Ed25519) | Same, best performance | Modern choice where supported |
| `none` | **None** | — | **Never. See §3** |

**The design rule:** if more than one service verifies tokens, use an asymmetric algorithm. With `HS256` every verifier needs the *signing* secret, which means every service can also **mint** tokens — so one compromised service can impersonate anyone. With `RS256`/`ES256` verifiers only get the public key. This is a genuinely good interview answer because it's a real architectural consequence, not trivia.

---

## 3. The four attacks on naïve JWT verification

Know all four. They're common interview questions and they're all real CVEs in real libraries.

### Attack 1 — `alg: none`
```json
{ "alg": "none", "typ": "JWT" }
```
The spec allows an "unsecured JWT" with an empty signature. Naïve libraries read `alg` from the **attacker-controlled header** and, seeing `none`, skip verification. Instant full compromise.

**Defence:** never let the token tell you how to verify it. Pin the expected algorithm:
```ts
jwt.verify(token, key, { algorithms: ["RS256"] })   // ← explicit allowlist, always
```

### Attack 2 — Algorithm confusion (RS256 → HS256)
The classic. Your server verifies with `RS256` using a public key. The attacker:
1. Changes the header to `{"alg": "HS256"}`.
2. Signs the token with **your public key as the HMAC secret** — which they have, because it's public.
3. A naïve library sees `HS256`, grabs "the key" (your public key), and verifies with HMAC. **It matches.**

**Defence:** the same explicit `algorithms` allowlist. Also keep signing and verification key material in separate types/variables so a public key can never be passed to an HMAC function.

### Attack 3 — `kid` injection / key confusion
`kid` (key ID) tells you which key to use. If you use it to look something up naïvely:
- `kid: "../../../dev/null"` → path traversal to a known-empty key
- `kid: "' OR 1=1--"` → SQL injection into your key lookup
- `kid: "https://attacker.com/jwks.json"` → you fetch the attacker's key and verify successfully

**Defence:** treat `kid` as an **opaque lookup key against an allowlist you control**. Never use it as a path, a URL, or unparameterised SQL. If you use JWKS, the JWKS URI must be configured server-side, never taken from the token.

### Attack 4 — Not verifying at all
The most common in the wild:
```ts
// ❌ decode ≠ verify. This checks NOTHING.
const claims = jwt.decode(token);
if (claims.role === "admin") { /* ... */ }
```
`decode` just base64-decodes. An attacker edits the payload to `{"role":"admin","sub":"anyone"}` and it sails through. **`jwt.decode` should essentially never appear in server code** — only `verify`.

### The complete verification checklist

```ts
import { createRemoteJWKSet, jwtVerify } from "jose";

// JWKS URI is configured HERE, never read from the token.
const JWKS = createRemoteJWKSet(new URL("https://auth.ledger.dev/.well-known/jwks.json"));

export async function verifyAccessToken(token: string) {
  const { payload } = await jwtVerify(token, JWKS, {
    algorithms: ["RS256"],                      // 1. pin the algorithm (blocks attacks 1 & 2)
    issuer: "https://auth.ledger.dev",          // 2. verify iss
    audience: "https://api.ledger.dev",         // 3. verify aud (blocks cross-service replay)
    clockTolerance: 5,                          // 4. small skew allowance, in seconds
    // exp / nbf are verified by the library automatically
  });

  if (!payload.sub || !payload.mrc) throw ApiError.unauthorized("malformed_token", "Missing required claims");

  // 5. Denylist check for revoked tokens (see §5)
  if (payload.jti && await denylist.has(String(payload.jti)))
    throw ApiError.unauthorized("token_revoked", "This token has been revoked");

  // 6. Global invalidation: reject tokens issued before the user's password change / forced logout
  const cutoff = await userCache.tokensInvalidBefore(String(payload.sub));
  if (cutoff && Number(payload.iat) < cutoff)
    throw ApiError.unauthorized("token_stale", "Re-authenticate");

  return payload;
}
```

Steps 5 and 6 are the interesting ones, and they're the subject of the next section — because notice what they did: **they made verification require a lookup.** That's the whole JWT trade-off, exposed.

---

## 4. Sessions vs JWT — the honest comparison

| | **Session (opaque ID + store)** | **JWT (self-contained)** |
|---|---|---|
| Verification | DB/Redis lookup (~0.3–1 ms) | Signature check (~0.05–0.5 ms), no I/O |
| **Revocation** | **Instant** — delete the row | **Impossible without a lookup**, which negates the benefit |
| Size on the wire | ~32 bytes | 300–1000+ bytes, **on every request** |
| Data freshness | Always current (you read it now) | **Stale until expiry** — a role change doesn't apply |
| Horizontal scaling | Needs a shared store | No shared store |
| Cross-service / cross-domain | Awkward | **Natural** — any service with the public key can verify |
| Failure mode | Store down → nobody can authenticate | Key rotation bugs; stolen token valid until `exp` |
| Storage in browser | `HttpOnly` cookie (safe) | Wherever you put it (usually less safe) |

### When JWT is genuinely the right answer
- **Multiple independent services** must verify identity without calling a central auth service on every request (microservices, service mesh).
- **Cross-domain / third-party** access where a cookie won't travel.
- **Mobile/native clients**, which have no cookie jar you control.
- Very high request volume where an auth lookup per request is a measurable cost.
- **Short-lived, single-purpose tokens**: a 60-second signed URL for a file download, a one-time email-verification link. These are JWT's best use case precisely because revocation is irrelevant at that lifetime.

### When a session is the right answer
- **First-party web app.** Almost always. You need instant logout, you need role changes to apply immediately, and `HttpOnly` cookies are the only XSS-proof storage available.
- Anything where "log this user out everywhere, now" is a product requirement (banking, admin panels, anything with a "sign out all devices" button).
- Small scale, where you'd be adding JWT complexity for a problem you don't have.

> **The senior answer to "sessions or JWT?"**
> *"They solve different problems. JWT buys stateless verification across services; the price is that you can't revoke. If revocation matters — and for a first-party web app it always does — you either accept a validity window or reintroduce a lookup, at which point you've built a slower session with a bigger cookie. So: short-lived JWT access tokens for service-to-service and mobile, and an opaque session or refresh token in an `HttpOnly` cookie for the browser. The hybrid is standard practice for a reason."*

---

## 5. The revocation problem, and the four real answers

You cannot un-issue a signed token. So:

| Strategy | How | Cost |
|---|---|---|
| **1. Short expiry** (5–15 min) | Accept a small window of validity after logout | The window is real. Requires refresh tokens |
| **2. Denylist by `jti`** | Store revoked `jti`s in Redis until their `exp` | A lookup per request — but the set is *small* (only revoked tokens), and TTL cleans it up automatically |
| **3. Global `tokens_invalid_before`** | One timestamp per user; reject tokens with `iat` older | One cheap cached lookup; handles "log out everywhere" and password change in one field |
| **4. Introspection** | Ask the auth server about every token | Fully revocable — and you've rebuilt sessions |

**The production combination** is 1 + 2 + 3:
- 15-minute access tokens
- A `jti` denylist for targeted revocation (a stolen token, a specific device)
- A per-user `tokens_invalid_before` timestamp, cached aggressively, for password change / "sign out everywhere"

That gives you revocation with one small cached lookup — and being able to describe this combination is what a senior answer to *"how do you revoke a JWT?"* sounds like. The junior answer is "you can't."

---

## 6. Access + refresh tokens, with rotation and theft detection

The standard architecture, and the details are where the value is.

```
Login
  → access_token   (JWT, 15 min, sent in Authorization header or held in JS memory)
  → refresh_token  (opaque, 30 days, stored server-side, sent in HttpOnly cookie)

Access token expires
  → POST /auth/refresh  (cookie travels automatically)
  → NEW access token + NEW refresh token; the old refresh token is invalidated
```

### Why refresh tokens are opaque, not JWTs
The refresh token is long-lived and powerful, so you **must** be able to revoke it, which means storing it, which means there's no benefit to it being self-contained. **Opaque + stored is strictly correct here.**

### Rotation and reuse detection — the part that matters

```ts
type RefreshToken = {
  id: string;
  familyId: string;          // all tokens descended from one login
  userId: string;
  tokenHash: string;         // hashed, like an API key
  expiresAt: Date;
  usedAt: Date | null;       // set on first use → single-use
  replacedBy: string | null;
};

export async function refresh(presented: string) {
  const row = await db.refreshTokens.findByHash(sha256(presented));
  if (!row) throw ApiError.unauthorized("invalid_refresh_token", "Re-authenticate");
  if (row.expiresAt < new Date()) throw ApiError.unauthorized("refresh_expired", "Re-authenticate");

  // ★ REUSE DETECTION: a single-use token presented twice means one of two things —
  //   a benign race (double tab), or a stolen token being replayed.
  //   You cannot tell which, so assume theft and burn the whole family.
  if (row.usedAt) {
    await db.refreshTokens.revokeFamily(row.familyId);
    await audit.log("refresh_token_reuse_detected", { userId: row.userId, familyId: row.familyId });
    throw ApiError.unauthorized("refresh_token_reused",
      "This session has been terminated for security reasons. Please sign in again.");
  }

  // Normal path: rotate.
  const next = await db.refreshTokens.create({
    familyId: row.familyId, userId: row.userId, ttlDays: 30,
  });
  await db.refreshTokens.markUsed(row.id, next.id);

  const access = await signAccessToken(row.userId, { ttlSeconds: 900 });
  return { access, refresh: next.plaintext };
}
```

**Why the family burn is correct:** if an attacker steals a refresh token and uses it, the legitimate user's next refresh presents an already-used token — detected, family revoked, user must re-authenticate, attacker also locked out. If instead the *attacker* refreshes second, they're locked out. **Either way the theft is detected and terminated within one refresh cycle**, and the user gets a security notification. This is the OAuth 2.1 recommended behaviour for public clients, and describing it is a strong senior signal.

**The practical wrinkle to mention:** genuine races do happen — two browser tabs refreshing simultaneously, or a mobile app resuming with several queued requests. Mitigations: a short grace window (accept the previous token for ~10 seconds and return the *same* new pair), and client-side single-flight refresh so only one request refreshes while others wait. You'll implement exactly that in the React track ([React L18](../../React/README.md)).

### Browser storage: the final answer

| Where | XSS steals it? | CSRF risk | Verdict |
|---|---|---|---|
| `localStorage` | **Yes, trivially** | No | ❌ Not for long-lived tokens |
| `sessionStorage` | **Yes** | No | ❌ Same problem, shorter |
| JS variable (memory) | Only while the page lives; gone on reload | No | ✅ For the **access** token |
| `HttpOnly` cookie | **No** | Yes → needs `SameSite` + CSRF token | ✅ For the **refresh** token / session |

**The recommended shape for the Ledger Console:**
- **Access token in a JS variable** (memory only). Short-lived. Sent as `Authorization: Bearer`. Lost on reload — which is fine, because…
- **Refresh token in an `HttpOnly`, `Secure`, `SameSite=Lax` cookie**, scoped to `/auth/refresh` via `Path`. On page load, the app calls `/auth/refresh` once to get an access token. XSS cannot read the refresh token; `SameSite` plus the narrow path blunts CSRF; and the refresh endpoint is the only CSRF-sensitive surface, so it also gets a double-submit token.

> **If your API and SPA are on different sites** you need `SameSite=None; Secure` plus exact-origin CORS with `credentials: true` — and you should then treat CSRF defence as mandatory rather than incidental.

---

## 7. Production rules

| Rule | Why |
|---|---|
| **Never `jwt.decode` on the server; only `verify`** | `decode` checks nothing |
| **Always pin `algorithms: [...]`** | Blocks `alg:none` and RS256→HS256 confusion |
| **Always verify `iss`, `aud`, `exp`** | `aud` is what stops a token for service A working on service B |
| **Asymmetric (RS256/ES256) if more than one service verifies** | Otherwise every verifier can mint tokens |
| **Treat `kid` as an opaque allowlist key** | Never a path, URL or raw SQL fragment |
| **Access tokens 5–15 min; refresh tokens 7–30 days** | Bounds the damage of theft |
| **Refresh tokens: opaque, hashed at rest, single-use, rotated, family-revoked on reuse** | Detects theft within one cycle |
| **Nothing sensitive in the payload** | It's readable by anyone holding the token |
| **Keep JWTs small** | They're on every request; oversized ones hit the 8KB header limit (`431`) |
| **Support key rotation via JWKS with overlapping validity** | You will need to rotate; don't make it an outage |
| **Per-user `tokens_invalid_before`** | The cheapest "log out everywhere" |
| **`HttpOnly` + `Secure` + `SameSite` on every auth cookie** | The only XSS-proof storage |
| **Never put a token in a URL** | Logs, `Referer`, history |
| **Log `jti`/`key_id` per request (never the token)** | Post-incident forensics |

> **Spring equivalent:** `spring-boot-starter-oauth2-resource-server` with `JwtDecoder` — and note that Spring's `NimbusJwtDecoder.withJwkSetUri(...)` handles the algorithm pinning and JWKS caching for you. See `spring boot/07-security/26-jwt-and-stateless-auth.md`.

---

## 8. Interview traps

**Q1. "What's in a JWT and is it encrypted?"**
Three base64url parts: header, payload, signature. **Signed, not encrypted** — the payload is readable by anyone. Signature gives integrity and origin, not confidentiality. (JWE encrypts; rarely used.)

**Q2. "How do you revoke a JWT?"**
Never say "you can't." Give the four-part answer from §5: short expiry, `jti` denylist, per-user `tokens_invalid_before`, or introspection — and note that any of them reintroduces a lookup, which is the honest cost of JWT's core benefit.

**Q3. "What's the algorithm confusion attack?"**
Attacker flips `alg` from `RS256` to `HS256` and signs with your *public* key as the HMAC secret. Naïve libraries verify successfully. Defence: pin `algorithms`, and keep key types separate.

**Q4. "Where do you store a JWT in the browser?"**
Not `localStorage`. Access token in memory, refresh token in an `HttpOnly` cookie. Name the threats: XSS reads `localStorage`; `HttpOnly` blocks that but needs `SameSite`/CSRF defence.

**Q5. "Explain refresh token rotation and reuse detection."**
Single-use refresh tokens; each use issues a new pair; a token presented twice means theft (or a race you can't distinguish from theft), so **revoke the whole token family** and force re-authentication. Mention the race mitigation (grace window + client single-flight) to show you've actually shipped it.

**Q6. "HS256 or RS256?"**
Symmetric vs asymmetric, and the deciding question is *"who verifies?"* One service → HS256 is fine. Many services → RS256/ES256, so verifiers can't mint tokens.

**Q7. "Aren't JWTs stateless? Why do you have a Redis lookup?"**
The trap. *"Verification is stateless; revocation isn't. I keep the lookup small — a denylist of revoked `jti`s plus a cached per-user invalidation timestamp — so I get most of the statelessness benefit and real revocation. If I needed full revocation on every request I'd use sessions instead, because at that point JWT is only costing me bytes."*

**Q8. "User's role changes from admin to viewer. When does it take effect?"**
With a 15-minute JWT carrying `role`: **up to 15 minutes later.** That's often unacceptable for a privilege *reduction*. Options: keep authorization data out of the token and look up permissions per request (a session-ish design), bump `tokens_invalid_before` on any role change to force an immediate refresh, or use very short access tokens. **The right answer names the staleness window as a product decision, not a technical detail.**

**Q9. "How does the API know a token wasn't issued by someone else?"**
Signature verification against the issuer's key, **plus** `iss` and `aud` checks. Without `aud`, a valid token minted for a different service in your ecosystem is accepted by yours.

**Q10. "How do you rotate signing keys without downtime?"**
Publish both keys in JWKS with distinct `kid`s → start signing with the new one → wait longer than the max token lifetime → remove the old key. Verifiers pick by `kid` from your allowlisted JWKS, cached with a sane TTL and a fallback refetch on unknown `kid` (rate-limited, so an attacker can't force refetch storms).

---

## 9. Build & break

### Build — a token service you'd ship
`scratch/auth/tokens.ts`:
```ts
signAccessToken(userId, merchantId, role) → 15-min RS256 JWT with jti, iss, aud
verifyAccessToken(token)                  → full checklist from §3
issueRefreshToken(userId, familyId?)      → opaque, hashed, 30 days
rotateRefreshToken(presented)             → new pair + reuse detection (family burn)
revokeAllForUser(userId)                  → bump tokens_invalid_before
revokeToken(jti, exp)                     → add to Redis denylist with TTL = exp - now
```

Tests, each mapping to an attack or a real bug:
```ts
test("rejects alg:none", ...);
test("rejects a token signed with the public key as an HMAC secret", ...);   // confusion
test("rejects a token whose aud is another service", ...);
test("rejects an expired token", ...);
test("accepts a token 3s past exp with clockTolerance 5", ...);
test("rejects a denylisted jti", ...);
test("rejects tokens issued before tokens_invalid_before", ...);
test("refresh rotation invalidates the old token", ...);
test("reusing a refresh token revokes the entire family", ...);
test("two concurrent refreshes within the grace window get the same new pair", ...);
```

### Break — five attacks against your own code
Do these against your own service, in a scratch branch, and delete it afterwards.

1. **`alg: none`.** Craft `{"alg":"none"}` with a payload of `{"sub":"usr_admin","role":"admin"}` and an empty signature. Try it. Then remove the `algorithms` option from `verify` and try again — see it succeed. Put it back.
2. **Algorithm confusion.** Take your public key, use it as an HMAC secret to sign a modified payload with `alg: HS256`, and send it. With a naïve verifier it works.
3. **`decode` instead of `verify`.** Base64-edit the payload to `role: "admin"` and watch a `decode`-based check accept it.
4. **Stolen token after logout.** Log out, then replay the access token you captured before logging out. It still works until `exp`. Now implement the `jti` denylist and watch it stop.
5. **The role-change staleness window.** Demote a user from admin to viewer and keep calling an admin endpoint with the old token. Time how long it keeps working. **That number is your answer to Q8**, and measuring it yourself is what makes the answer credible.

### Explain out loud (2 minutes)
1. What's in a JWT, and why "signed not encrypted" matters.
2. The four attacks, and the one-line defence that stops two of them.
3. The sessions-vs-JWT trade-off, in terms of revocation.
4. Refresh rotation with reuse detection, including the race.
5. Where each token lives in a browser, and the attack each choice addresses.

---

## What's next

You can authenticate your own users. Next: letting a *third party* act on a user's behalf without ever seeing their password — OAuth 2.1 and OIDC, the flow diagrams you should be able to draw from memory, why implicit died, and exactly what PKCE prevents.

Next → **[Lesson 14: OAuth 2.1 & OpenID Connect](14-oauth2-and-oidc.md)**
