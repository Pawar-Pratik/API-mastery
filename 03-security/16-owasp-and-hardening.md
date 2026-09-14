# Lesson 16 — Hardening: OWASP API Top 10 in practice

> **Why this lesson exists:** the OWASP API Security Top 10 is the industry's shared vocabulary for API risk — interviewers reference it by number, security reviews are structured around it, and it's the closest thing to a checklist for "is this API safe to ship?" This lesson goes through all ten with the actual code, plus the two that catch experienced engineers: **SSRF** (how cloud credentials get stolen) and **business-flow abuse** (where nothing is technically broken and you still lose money).

**Time:** ~90 minutes · **Prereq:** Lessons 12–15

---

## 1. The idea in one sentence

> **APIs fail differently from web pages: there's no browser enforcing anything, the attack surface is the *contract itself*, and the top risks are all about authorization and resource limits rather than the injection bugs everyone studies for.**

That last clause is the useful insight. Web app security education is dominated by XSS and SQL injection. **In APIs, seven of the top ten are authorization, configuration and resource problems.** Adjust your instincts accordingly.

---

## 2. The Top 10 (2023 edition), with fixes

### API1 — Broken Object Level Authorization (BOLA)
**Covered in full in [Lesson 15](15-authorization-and-multitenancy.md).** The #1 risk. One-line summary: authenticated caller, someone else's object ID, 200 OK. Fix structurally — scoped repositories, tenancy from the credential, RLS, 404-not-403, authorization matrix in CI.

### API2 — Broken Authentication

The failures, in the order you'll find them in a real codebase:

| Failure | Fix |
|---|---|
| No rate limit on login/token endpoints | Per-IP **and** per-account limits + exponential backoff ([Lesson 19](../04-production/19-rate-limiting.md)) |
| `jwt.decode` instead of `verify`, or no `algorithms` pin | [Lesson 13 §3](13-sessions-and-jwt.md) |
| No `aud`/`iss` verification | A sibling service's token works on yours |
| Weak or absent password policy; passwords hashed with SHA-256 | argon2id (or bcrypt cost ≥12) + a breached-password check (k-anonymity API) |
| Credentials accepted in a query string | Headers only |
| Tokens that never expire | Short access tokens + rotating refresh |
| No account-recovery hardening | Recovery is an auth path — rate limit it, expire tokens in minutes, single-use, and notify on use |
| Timing oracle on user existence | Constant-work comparison + generic message ([Lesson 12 §4](12-authentication-landscape.md)) |

**The one people forget:** *password reset is an authentication mechanism.* A reset token that's long-lived, reusable, or guessable is a full account takeover with no password needed. Make it: 256-bit random, hashed at rest, single-use, 15-minute expiry, invalidated on use *and* on password change, and it must invalidate all existing sessions when redeemed.

### API3 — Broken Object Property Level Authorization (BOPLA)

Two directions, and both matter:

**Writing what you shouldn't** (mass assignment):
```ts
// ❌ the client decides what a payment is
await db.payments.insert({ ...req.body, merchant_id: req.merchantId });
// client sends {"status":"succeeded","amount_minor":1,"fee_minor":0} → fraud

// ✅ explicit allowlist, and server-owned fields set by the server
const input = CreatePayment.parse(req.body);         // .strict(), client-settable fields only
await db.payments.insert({
  id: newId("pi"),
  merchant_id: req.merchantId,                        // from the credential
  amount_minor: input.amount_minor,
  currency: input.currency,
  status: "requires_payment_method",                  // server decides the state machine
  created_at: new Date(),
});
```

**Reading what you shouldn't** (excessive data exposure) — the subtler half:
```ts
// ❌ the entity IS the response. Every column you ever add is published.
res.json(payment);
// leaks: internal_risk_score, processor_raw_response, fee_breakdown, card_fingerprint, merchant_notes

// ✅ an explicit output DTO — a positive list, never a deny list
export function toPaymentDto(p: Payment) {
  return {
    object: "payment",
    id: p.id, amount_minor: p.amountMinor, currency: p.currency,
    status: p.status, description: p.description, metadata: p.metadata,
    customer: p.customerId, created_at: p.createdAt.toISOString(),
  };
}
```

> **Why a positive list and never `delete payment.internalScore`:** a deny list fails on the *next* column someone adds. A positive list fails closed — a new column is simply invisible until someone deliberately adds it. Same argument as the DTO discussion you already know from `spring boot/05-web/16`.

**And the API-specific trap:** don't rely on the client to hide fields. A mobile app that receives the full user object and displays only the name has still *shipped* the SSN to the device, where anyone can read it with a proxy. **"The UI doesn't show it" is not a control.**

### API4 — Unrestricted Resource Consumption

Every unbounded input is a denial-of-service, and often a *bill*.

```ts
// The complete bounds checklist
app.use(express.json({ limit: "100kb" }));              // body size          → 413
const ListQuery = z.object({ limit: z.coerce.number().int().min(1).max(100).default(20) });
// ↑ page size          → 400

// depth & breadth limits (a 10,000-level nested JSON is a parser bomb)
function assertDepth(v: unknown, max = 10, d = 0): void {
  if (d > max) throw ApiError.badRequest("payload_too_deep", `Max nesting depth is ${max}`);
  if (Array.isArray(v)) { if (v.length > 1000) throw ApiError.badRequest("array_too_large", "Max 1000 items");
                          v.forEach(x => assertDepth(x, max, d + 1)); }
  else if (v && typeof v === "object") Object.values(v).forEach(x => assertDepth(x, max, d + 1));
}

// timeouts on EVERY outbound call
const res = await fetch(url, { signal: AbortSignal.timeout(5_000) });

// rate limits, quotas, and concurrency caps
```

| Unbounded thing | Attack | Bound it with |
|---|---|---|
| Body size | Memory exhaustion (and event-loop stall in Node) | `limit`, → `413` |
| Page size / `IN` list length | One query reads the whole table | max 100 / max 50 |
| JSON depth & array length | Parser DoS | explicit depth/breadth checks |
| Regex on user input | **ReDoS** — catastrophic backtracking freezes the event loop | avoid nested quantifiers; use `re2`, or a length cap + timeout |
| File upload size & count | Disk fill | multer `limits`, streaming, pre-signed URLs |
| Requests per second | Bandwidth, cost | rate limits ([Lesson 19](../04-production/19-rate-limiting.md)) |
| Expensive operations (report, export, image resize) | CPU/$$ — **and your cloud bill** | queue + per-tenant concurrency cap |
| Third-party calls you make (SMS/email/AI) | **Financial** DoS — attacker spends your money | per-tenant quotas, spend alerts |
| GraphQL query depth/complexity | One query fans out to millions of resolvers | depth + complexity limits ([Lesson 21](../05-beyond-rest/21-graphql.md)) |

> **That "financial DoS" row is worth volunteering in an interview.** An endpoint that sends an SMS per request, with no per-tenant quota, is an attacker-controlled spending faucet. Real companies have received five-figure bills this way. It's not an availability bug, it's an accounting one.

### API5 — Broken Function Level Authorization (BFLA)
Calling an endpoint your role shouldn't reach. Fixes: role/permission middleware on **every** route (default-deny, not default-allow), no "hidden" admin endpoints relying on obscurity, and the CI route-coverage test from [Lesson 15 §7](15-authorization-and-multitenancy.md).

```ts
// ✅ Default deny: an unregistered route can't be reached without a declared permission.
const router = createRouter({ defaultPolicy: "deny" });
router.post("/payments", { permission: "payments:write" }, createPayment);
router.get ("/admin/merchants", { permission: "platform:admin" }, listMerchants);
// A route registered with no `permission` key fails at startup, not at runtime.
```
Making a missing permission a **startup failure** is the trick — it converts a security omission into a deploy failure.

### API6 — Unrestricted Access to Sensitive Business Flows

The most interesting entry, because **nothing is technically broken.** Every request is authenticated, authorized, validated and within rate limits. The *business flow* is being abused.

| Flow | Abuse |
|---|---|
| Ticket purchase | Bots buy the entire inventory in 2 seconds, resell at 5× |
| Product launch / limited drop | Same |
| Referral bonus | Script creates 10,000 accounts and harvests credits |
| Free trial | Automated signups burn your compute forever |
| Card payment attempts | **Card testing** — validating stolen card numbers against your checkout; you eat the fees and the fraud score |
| Review/rating | Manipulation at scale |
| Password reset email | Using your server as a mailbomb against a third party |

Defences are *business* controls, not technical ones:
- **Detect automation** — device fingerprinting, behavioural signals, proof-of-work, CAPTCHA on the *sensitive step only*
- **Per-identity limits, not per-request** — "3 refunds per customer per day", "1 trial per payment method"
- **Velocity checks** — "10 failed card attempts from one merchant in a minute" → block and alert
- **Human-in-the-loop for high-value irreversible actions** — a payout above a threshold gets manual review
- **Make abuse expensive** — require a verified phone or a card authorization before granting a bonus

> **Card testing is the Ledger-specific one**, and it's worth designing for explicitly: an attacker uses your payment endpoint to test stolen cards, and each attempt costs you a gateway fee and damages your processor risk profile. Defences: strict per-merchant and per-card-fingerprint attempt limits, a rising failure-ratio circuit breaker, and blocking a card fingerprint after N failures across the platform. Bringing this up unprompted in a payments interview is a very strong signal.

### API7 — Server Side Request Forgery (SSRF)

**The one that gets cloud credentials stolen**, and the one experienced engineers still get wrong.

Any endpoint that fetches a URL the user supplied — a webhook URL, an image import, a PDF renderer, a link preview, an OpenAPI import — can be pointed *inward*:

```
POST /v1/webhook-endpoints  { "url": "http://169.254.169.254/latest/meta-data/iam/security-credentials/" }
                              ↑ AWS instance metadata → temporary IAM credentials → your whole account
POST /v1/import             { "url": "http://localhost:6379/" }        → your Redis
POST /v1/import             { "url": "http://10.0.0.5:5432/" }         → internal Postgres
POST /v1/import             { "url": "file:///etc/passwd" }            → local files
POST /v1/import             { "url": "http://metadata.google.internal/computeMetadata/v1/" }
```

The **2019 Capital One breach** — 100M+ records — was an SSRF that reached the EC2 metadata service and retrieved IAM credentials. That's the reference case, and naming it is worth doing.

Why naïve blocklists fail — this is the part that catches people:
- `http://2130706433/` — decimal-encoded 127.0.0.1
- `http://0x7f.0x0.0x0.0x1/` — hex
- `http://[::1]/`, `http://[::ffff:127.0.0.1]/` — IPv6 forms
- `http://127.0.0.1.nip.io/` — a public DNS name that *resolves* to loopback
- `http://evil.com/` that **301-redirects** to `169.254.169.254`
- **DNS rebinding** — resolves to a public IP when you validate, then to an internal IP when you connect (the TOCTOU race)

The defence, in layers:

```ts
import { lookup } from "node:dns/promises";
import net from "node:net";
import ipaddr from "ipaddr.js";

const ALLOWED_PROTOCOLS = new Set(["http:", "https:"]);
const ALLOWED_PORTS = new Set([80, 443]);

function isPublicUnicast(ip: string): boolean {
  const addr = ipaddr.parse(ip);
  const range = addr.range();   // "unicast" is the only acceptable answer
  return range === "unicast";   // excludes private, loopback, linkLocal, uniqueLocal,
}                               // carrierGradeNat, reserved, broadcast, multicast

export async function assertSafeUrl(raw: string): Promise<{ url: URL; ip: string }> {
  const url = new URL(raw);                                        // 1. parse strictly
  if (!ALLOWED_PROTOCOLS.has(url.protocol))
    throw ApiError.badRequest("invalid_url_scheme", "Only http and https are allowed");
  if (url.port && !ALLOWED_PORTS.has(Number(url.port)))
    throw ApiError.badRequest("invalid_url_port", "Only ports 80 and 443 are allowed");
  if (url.username || url.password)
    throw ApiError.badRequest("invalid_url", "Credentials in URLs are not allowed");

  const { address } = await lookup(url.hostname);                  // 2. resolve
  if (!isPublicUnicast(address))                                   // 3. validate the IP, not the name
    throw ApiError.badRequest("url_not_allowed", "URL resolves to a non-public address");

  return { url, ip: address };
}

/** 4. Connect to the VALIDATED IP, with the Host header preserved — closes the rebinding race. */
export async function safeFetch(raw: string) {
  const { url, ip } = await assertSafeUrl(raw);
  return fetch(`${url.protocol}//${ip}${url.pathname}${url.search}`, {
    headers: { host: url.host },
    redirect: "manual",                    // 5. NEVER auto-follow; re-validate each hop yourself
    signal: AbortSignal.timeout(5_000),    // 6. bound it
  });
}
```

The six controls, in priority order:
1. **Allowlist protocols and ports** (no `file:`, `gopher:`, `dict:`, no port 6379/5432/22)
2. **Resolve the hostname, then validate the resulting IP** against private/loopback/link-local/CGNAT ranges — validating the *string* is useless
3. **Connect to the validated IP**, passing `Host` — this defeats DNS rebinding
4. **`redirect: "manual"`** and re-validate every hop
5. **Egress network controls** — the real fix at scale: a dedicated egress proxy or a security group that simply cannot reach `169.254.169.254` or your private subnets
6. **IMDSv2** on AWS (token-required metadata), which would have blocked Capital One

> **The best interview answer names #5 and #6**, because it shows you know that application-layer validation is a mitigation and network-layer isolation is the actual control.

### API8 — Security Misconfiguration

The boring one that fails audits:

```ts
app.disable("x-powered-by");                     // stop advertising your stack
app.use(helmet());                                // HSTS, nosniff, frameguard, CSP, referrer-policy
app.use(cors({ origin: ALLOWLIST, credentials: true, maxAge: 86400 }));  // never reflect
app.set("trust proxy", 1);                        // exactly one proxy hop, not `true`
```

| Misconfiguration | Consequence |
|---|---|
| `Access-Control-Allow-Origin: *` **with** credentials | Any site reads your users' data. (Browsers reject this combination — but "reflect whatever `Origin` says" achieves the same hole and browsers accept it) |
| Stack traces / SQL / hostnames in error responses | Reconnaissance ([Lesson 10](../02-rest-design/10-errors-and-problem-details.md)) |
| `trust proxy: true` | Anyone forges `X-Forwarded-For` and bypasses IP limits |
| Debug endpoints in production (`/debug`, `/actuator/env`, `/graphql` playground) | Config and secret disclosure |
| Default credentials on Redis/Elastic/Mongo/admin panels | Direct data access. Still the #1 cloud data-leak cause |
| Verbose `Server:`/`X-Powered-By` headers | Version-specific exploit targeting |
| No TLS, or TLS 1.0/1.1 accepted, or an expired cert | Interception; and expiry is a self-inflicted outage |
| Permissive S3 buckets / open cloud storage | The most common data leak in the industry |
| Missing `Vary` on negotiated responses | Cross-user cache poisoning ([Lesson 04](../01-foundations/04-headers-and-payloads.md)) |

### API9 — Improper Inventory Management

*"We didn't know that endpoint was still live."*

The pattern: `api-v1.company.com` was replaced by v2 two years ago but still runs, unpatched, on an old host, without the WAF, still connected to production data. Attackers scan for exactly this.

The controls:
- **An API inventory** — every environment, version and host, generated from your specs and gateway config rather than maintained by hand
- **Retire, don't just deprecate** — a sunset date you actually honour ([Lesson 11](../02-rest-design/11-versioning-and-evolution.md))
- **Non-production must never touch production data.** Staging is usually less monitored and more permissive; giving it real data makes it your weakest link
- **Document and gate internal APIs too.** "It's internal" is not an access control
- **Monitor for undocumented endpoints** — diff your gateway's observed routes against your OpenAPI spec, and alert on the difference

### API10 — Unsafe Consumption of APIs

You harden your own endpoints and then blindly trust a third party's response. **Data from an upstream API is untrusted input.**

```ts
// ❌ trusting a partner's response shape and content
const { balance, redirect_url } = await fetch(partnerUrl).then(r => r.json());
await db.accounts.update({ balance });        // they sent a string, or a negative number
res.redirect(redirect_url);                    // open redirect, courtesy of your partner

// ✅ validate, bound and constrain
const Resp = z.object({
  balance_minor: z.number().int().min(0),
  redirect_url: z.string().url().refine(u => new URL(u).host.endsWith(".partner.com")),
});
const parsed = Resp.parse(await res.json());
```
Also: bound the response size (a partner streaming 5GB is your OOM), always set timeouts, never auto-follow their redirects into your internal network (SSRF again), and treat their strings as untrusted when they'll be rendered or interpolated.

---

## 3. Injection, briefly but properly

Injection dropped off the API-specific top ten because ORMs and parameterisation made it rarer — but it still happens, and each variant has a specific fix.

| Type | Fix |
|---|---|
| **SQL** | Parameterised queries, always. Allowlist any *identifier* (column/table/direction) you interpolate ([Lesson 08](../02-rest-design/08-collections-and-pagination.md)) |
| **NoSQL** | Mongo: reject objects where you expect scalars — `{"password": {"$ne": null}}` is a classic auth bypass. Validate types before querying |
| **Command** | Never build a shell string. `spawn(cmd, [args])`, never `exec(\`cmd ${input}\`)` |
| **Path traversal** | Never join user input into a path. Generate your own names; if you must, `path.resolve` and assert the result stays under the base dir |
| **XXE** | Disable external entities in every XML parser (`noent: false`, `resolveExternals: false`). This is why XML parsing is riskier than JSON parsing |
| **Log injection** | Escape newlines in logged user input, or an attacker forges log lines to hide their tracks |
| **Header/CRLF injection** | Never build a header from raw input ([Lesson 04](../01-foundations/04-headers-and-payloads.md)) |
| **SSTI / prototype pollution** | Don't render user input as a template; block `__proto__`/`constructor` keys in parsed JSON |
| **XSS via API** | Your JSON API can still feed XSS: if a client renders `payment.description` as HTML, your unescaped storage becomes their vulnerability. Set `X-Content-Type-Options: nosniff`, serve `application/json`, and never serve user content from your API's origin |

---

## 4. The pre-launch hardening checklist

Copy this into Ledger's repo. It's the artifact a security reviewer wants to see.

```
TRANSPORT & CONFIG
[ ] HTTPS only, HSTS with includeSubDomains, TLS 1.2+ (prefer 1.3)
[ ] Certificate expiry alerting (30/14/7 days)
[ ] helmet() (or equivalent) on every response
[ ] x-powered-by / Server headers removed
[ ] trust proxy set to the exact number of hops
[ ] CORS: explicit origin allowlist, no reflection, Vary: Origin, credentials only where needed

AUTH
[ ] Rate limits on login/token/reset, per-IP AND per-account, with backoff
[ ] Passwords: argon2id (or bcrypt ≥12) + breached-password check
[ ] Tokens: algorithms pinned, iss/aud/exp verified, short-lived access, rotating refresh
[ ] API keys: hashed at rest, prefixed, public-id split, constant-time compare, show-once
[ ] Password reset: 256-bit, hashed, single-use, ≤15 min, invalidates sessions
[ ] MFA available for privileged roles

AUTHORIZATION
[ ] Tenancy derived only from the credential
[ ] Scoped repositories; unscoped DB access not importable from handlers
[ ] Row-level security enabled on tenant tables
[ ] Permission declared on every route; missing permission = startup failure
[ ] 404 (not 403) for cross-tenant
[ ] Nested routes validate the full parent chain
[ ] Actor × endpoint authorization matrix in CI, failing on uncovered routes

INPUT & OUTPUT
[ ] Every input validated with a .strict() schema at the boundary
[ ] Body size, array length and nesting depth bounded
[ ] Output DTOs are positive allowlists — entities never serialized directly
[ ] Content-Type validated on writes (415 otherwise)
[ ] No user input interpolated into SQL identifiers, paths, shell commands or headers

RESOURCE LIMITS
[ ] Rate limits per key/tenant, with RateLimit-* headers and Retry-After
[ ] Per-tenant quotas on expensive operations and third-party spend
[ ] Timeouts on every outbound call; timeouts decrease downstream
[ ] Concurrency caps on heavy work; queue + shed rather than queue forever

SSRF & OUTBOUND
[ ] User-supplied URLs: protocol/port allowlist, resolve-then-validate-IP, connect-to-IP
[ ] redirect: manual, each hop re-validated
[ ] Egress restricted at the network layer; metadata endpoint unreachable; IMDSv2 enforced
[ ] Upstream responses validated, size-bounded and timed out

SECRETS & DATA
[ ] No secrets in code, images or logs; scanning in CI (gitleaks/trufflehog)
[ ] Secrets in a manager with rotation (Vault / AWS SM / GCP SM)
[ ] Authorization, Cookie, Set-Cookie, Idempotency-Key redacted in logs — with a test
[ ] PII minimised; encryption at rest; documented retention and deletion paths
[ ] Card data: never stored (tokenize via the processor); PCI scope kept minimal

OPERATIONS
[ ] No stack traces, SQL or internal hostnames in any response
[ ] X-Request-Id on every response; structured logs keyed by it
[ ] Alerts on: 401/403 spikes, 5xx rate, latency p99, auth failures per account, rate-limit hits
[ ] Audit log for privileged actions (immutable, exportable)
[ ] Dependency scanning + automated patching (Dependabot/Renovate)
[ ] API inventory: every version and environment known; retired versions actually off
[ ] Non-production never holds production data
[ ] Documented incident response: who, how to revoke, how to notify
```

---

## 5. Interview traps

**Q1. "What's the OWASP API Top 10 and which matters most?"**
Name it as the API-specific list (distinct from the web Top 10) and lead with the insight: **most entries are authorization, configuration and resource-limit problems, not injection.** #1 is BOLA. If you can name 5–6 by concept you're well ahead.

**Q2. "What is SSRF and how do you prevent it?"**
Your server is tricked into fetching an attacker-chosen URL, typically to reach cloud metadata or internal services. **Cite Capital One.** Then the layered defence: protocol/port allowlist → resolve → validate the *IP* → connect to that IP with the `Host` header → `redirect: manual` → **egress network controls + IMDSv2**. Explaining *why* string blocklists fail (decimal/hex/IPv6 encodings, `nip.io`, redirects, DNS rebinding) is what separates a real answer.

**Q3. "Your endpoint accepts a webhook URL. What could go wrong?"**
SSRF (internal reachability), then the second-order problems most candidates miss: it's an **outbound amplification vector** (an attacker registers a victim's URL and makes you DDoS them), it needs signature verification so receivers can trust you, it needs retry limits and a dead-letter path, and the URL must be re-validated on *every* delivery, not just at registration (DNS changes).

**Q4. "How do you stop someone from abusing your API to test stolen credit cards?"**
API6. Per-merchant and per-card-fingerprint attempt limits, a rising failure-ratio circuit breaker, platform-wide fingerprint blocking after N failures, velocity checks, and bot detection at the sensitive step. Note that **nothing is technically broken** — this is a business-flow control, not a vulnerability fix.

**Q5. "What's the difference between mass assignment and excessive data exposure?"**
Two directions of the same failure: mass assignment is the client writing fields it shouldn't; excessive exposure is the server returning fields it shouldn't. Both fixed by explicit DTOs — a `.strict()` input schema and a positive-allowlist output mapper. Both are API3.

**Q6. "Your mobile app hides the SSN field in the UI. Is that fine?"**
No. If the API returned it, it's on the device and readable with a proxy in 30 seconds. **Filter on the server.** "The UI doesn't show it" is not a control.

**Q7. "An unbounded `limit` parameter — what's the worst case?"**
`?limit=10000000` reads the whole table, allocates it in memory, and in Node `JSON.stringify` blocks the event loop so *every other request* stalls. It's an availability bug, not a performance one. Fix: documented max, and honest `has_more`.

**Q8. "How do you prevent a financial DoS?"**
Per-tenant quotas on anything that costs money downstream (SMS, email, AI tokens, image processing), hard spend caps with alerting, and a circuit breaker on your own spend. Name the failure mode: *"an unmetered endpoint that sends an SMS is an attacker-controlled faucet on my bank account."*

**Q9. "You find a critical vulnerability in production. Walk me through the next hour."**
Structure matters more than heroics: **contain** (revoke the credential / disable the endpoint / block the pattern at the WAF) → **assess blast radius** from logs → **fix** → **verify** → **notify** (customers, legal/compliance if data was touched; note that GDPR is 72 hours) → **post-mortem with a systemic fix and a regression test**. The signal is that containment comes before root-cause analysis, and that you know a regression test is part of the fix.

**Q10. "How would you find these bugs in someone else's API?"**
Read the OpenAPI spec, then: enumerate every object-taking endpoint and try another tenant's ID (BOLA); send extra fields to every write (BOPLA); call every write endpoint as the lowest role (BFLA); send an unbounded `limit` and a deeply nested body (API4); point every URL-accepting field at `169.254.169.254` (SSRF); check response headers and error bodies (API8); and look for `v1` hosts that outlived `v2` (API9). That ordered list *is* an API pentest methodology, and reciting it is a strong close.

---

## 6. Build & break

### Build — the hardening middleware stack
```ts
import express from "express";
import helmet from "helmet";

export function harden(app: express.Express) {
  app.disable("x-powered-by");
  app.set("trust proxy", 1);                              // exactly one hop

  app.use(helmet({
    contentSecurityPolicy: { directives: { defaultSrc: ["'none'"], frameAncestors: ["'none'"] } },
    hsts: { maxAge: 31_536_000, includeSubDomains: true, preload: true },
    referrerPolicy: { policy: "no-referrer" },
  }));

  app.use(cors({
    origin: (origin, cb) => cb(null, !origin || ALLOWED_ORIGINS.includes(origin)),
    credentials: true,
    exposedHeaders: ["X-Request-Id", "RateLimit-Remaining", "RateLimit-Reset", "ETag"],
    maxAge: 86_400,
  }));

  app.use(requestId());
  app.use(rateLimitByIp({ windowMs: 60_000, max: 300 }));   // before auth
  app.use(express.json({ limit: "100kb", verify: keepRawBody }));
  app.use(depthLimiter({ maxDepth: 10, maxArray: 1000 }));
  return app;
}
```

### Build — the SSRF guard, with the tests that prove it
Implement §API7's `assertSafeUrl`/`safeFetch`, then write tests for **every** bypass:
```ts
test.each([
  "http://169.254.169.254/latest/meta-data/",   // AWS IMDS
  "http://metadata.google.internal/",            // GCP
  "http://127.0.0.1:6379/",                      // Redis
  "http://localhost/",
  "http://2130706433/",                          // decimal loopback
  "http://0x7f000001/",                          // hex loopback
  "http://[::1]/",                               // IPv6 loopback
  "http://[::ffff:169.254.169.254]/",            // IPv4-mapped IPv6
  "http://10.0.0.5/", "http://192.168.1.1/", "http://172.16.0.1/",
  "http://100.64.0.1/",                          // CGNAT
  "file:///etc/passwd",
  "gopher://127.0.0.1:6379/_SET%20x%20y",
  "http://user:pass@169.254.169.254/",
  "http://127.0.0.1.nip.io/",                    // public DNS → loopback
])("blocks %s", async url => {
  await expect(assertSafeUrl(url)).rejects.toThrow();
});

test("blocks a public URL that redirects to metadata", async () => {
  // requires redirect: "manual" + re-validation
});
```

### Break — attack your own API, methodically
Run the §Q10 methodology against Ledger and write down what you find. Specifically:
1. Every endpoint taking an ID → try tenant B's ID.
2. Every write → add `{"status":"succeeded","merchant_id":"...","balance":999,"role":"owner"}`.
3. Every write → call it as `viewer`.
4. `?limit=99999999`, a 10MB body, and a 5,000-level nested object.
5. Every URL field → the SSRF list above.
6. `curl -I` every endpoint → look for `X-Powered-By`, missing HSTS, missing `nosniff`.
7. Force a 500 → check the body for a stack trace.
8. Grep your own logs for `Bearer `, `sk_live`, `Cookie:`.

**Every finding becomes a permanent test.** That's the deliverable.

### Explain out loud (2 minutes)
1. Why API security differs from web app security.
2. Five of the Top 10 by concept.
3. SSRF: the attack, why blocklists fail, and the two controls that actually work.
4. Mass assignment vs excessive data exposure, and the one fix for both.
5. An example of business-flow abuse where nothing is technically broken.

---

## Module 3 complete — checkpoint

Cold, no notes:

- [ ] Authn vs authz, and which one actually gets breached
- [ ] The seven auth mechanisms and the client-type decision table
- [ ] Why API keys get SHA-256 and passwords get argon2id
- [ ] The three rules of HMAC request signing, including raw-body capture
- [ ] What's in a JWT; why "signed, not encrypted" matters
- [ ] The four attacks on naïve JWT verification, and the pin that stops two
- [ ] The four ways to revoke a JWT, and what each costs
- [ ] Refresh rotation with reuse detection and family revocation
- [ ] Where each token lives in a browser, and the threat each choice addresses
- [ ] Draw the auth code + PKCE flow and explain every hop
- [ ] What PKCE prevents, in one sentence
- [ ] OAuth vs OIDC, and why an access token can't authenticate a user
- [ ] BOLA: what it is, why it's #1, and four structural defences
- [ ] RBAC vs ABAC vs ReBAC, and where each breaks
- [ ] The tenant-isolation ladder and when you'd move up it
- [ ] SSRF end to end, including the bypasses and the network-layer fix
- [ ] Five of the OWASP API Top 10 with fixes

---

## What's next

Your API is designed and defended. Module 4 makes it **fast and reliable under real load** — starting with the largest performance lever HTTP gives you and the one almost every API ignores: caching, done correctly, including the difference between `no-cache` and `no-store` that everyone gets wrong.

Next → **[Lesson 17: Caching & HTTP performance](../04-production/17-caching-and-performance.md)**
