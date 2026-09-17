# Lesson 29 — Ledger build log

> **Why this lesson exists:** knowing *what* to build (Lesson 28) and knowing the techniques (Modules 1–6) still leaves the hardest question: **in what order?** Build it wrong and you spend three weeks on scaffolding before anything works, lose momentum, and abandon it. This build log is ordered so that **something demonstrable exists after every phase**, and each phase only uses lessons you've already read.

**Time:** 8 phases · roughly 40–60 hours total · **Prereq:** Lesson 28

---

## How to use this

- **One phase at a time. Finish it.** Each phase ends with a "Demonstrate" step — an actual `curl` command that works. Don't start the next phase until it does.
- **Type the code.** The snippets here are anchors, not a codebase to copy. Referenced lessons have the full versions.
- **Write the test in the same sitting as the feature.** Not later. Later doesn't come.
- **Keep a `DECISIONS.md`** as you go: every time you make a choice, write one line about why. At the end you'll have `docs/api-architecture.md` for free, and it'll be honest rather than reconstructed.
- **Commit per phase**, with a message naming what now works. Your git log becomes a build narrative you can walk an interviewer through.

---

## Phase 0 — Foundations (3–4 h)

**Goal:** an empty but correct skeleton. Everything after this is features.

```bash
mkdir ledger && cd ledger && npm init -y
npm i express zod pino pino-http pg redis ulid dotenv
npm i -D typescript tsx @types/node @types/express vitest supertest @types/supertest \
        @testcontainers/postgresql node-pg-migrate @stoplight/spectral-cli
npx tsc --init
```

`tsconfig.json` — non-negotiable settings:
```json
{
  "compilerOptions": {
    "target": "ES2022", "module": "NodeNext", "moduleResolution": "NodeNext",
    "strict": true,
    "noUncheckedIndexedAccess": true,       // ← catches real bugs; see TS track
    "exactOptionalPropertyTypes": true,     // ← matters for PATCH semantics (L09)
    "outDir": "dist", "rootDir": "src",
    "esModuleInterop": true, "skipLibCheck": true
  }
}
```

**Build:**
1. `docker-compose.yml` with Postgres 16 and Redis 7.
2. `src/config.ts` — validate env with zod and **fail at boot** if anything's missing. A service that starts with a missing secret and fails on first request is strictly worse than one that won't start.
3. `src/lib/db.ts`, `src/lib/redis.ts`, `src/lib/logger.ts` (pino with `redact`, from [L20](../04-production/20-observability.md)).
4. `src/app.ts` with middleware in the **correct order** ([L02 §11](../01-foundations/02-journey-of-a-request.md)):
   ```ts
   app.use(correlation());            // 1. first — every log carries the ID
   app.use(pinoHttp({ logger }));     // 2.
   app.use(helmet());                 // 3.
   app.use(cors(corsOptions));        // 4. BEFORE auth (preflights are unauthenticated)
   app.use(express.json({ limit: "100kb" }));   // 5. BOUNDED
   app.use("/v1", v1Router);
   app.use(errorHandler);             // LAST — the only place errors are formatted
   ```
5. `/healthz` and `/readyz`, with the liveness/readiness split respected ([L20 §7](../04-production/20-observability.md)).
6. `src/middleware/errors.ts` — the RFC 9457 handler and `ApiError` class ([L10 §6](../02-rest-design/10-errors-and-problem-details.md)).
7. Testcontainers setup running real migrations ([L26 §3](../06-craft/26-testing-apis.md)).

**Demonstrate:**
```bash
curl -i localhost:3000/healthz            # 200, and an X-Request-Id header
curl -i localhost:3000/v1/nope            # 404 problem+json with code + request_id
curl -i -X POST localhost:3000/v1/nope -H 'content-type: text/plain' -d x   # 415
```
**Test:** the error handler returns a consistent shape for 404, 415, 413 and 500, and **no stack trace appears in any response.**

> **Lessons used:** 02 (middleware order), 04 (headers), 10 (errors), 20 (logging, health), 26 (test setup).

---

## Phase 1 — Contract & tenancy (5–6 h)

**Goal:** the contract exists, and nobody can write an unscoped query.

**Build:**
1. **`openapi.yaml` first.** Payments and customers only for now. Full `Problem` schema, `$ref`'d parameters and responses, `additionalProperties: false`, real examples ([L25](../06-craft/25-openapi-contract-first.md)).
2. Spectral in CI with your house rules.
3. Migrations for `merchants`, `api_keys`, `customers`, `payments`.
4. **API keys** ([L12 §2](../03-security/12-authentication-landscape.md)): `public_id` + secret split, SHA-256 hash, prefix, `timingSafeEqual`, show-once, test/live envs.
5. `authenticate` middleware producing `req.principal`, with `req.merchantId` **derived only from the credential**.
6. **Scoped repositories** ([L15 §3](../03-security/15-authorization-and-multitenancy.md)) — plus the ESLint rule that makes it stick:
   ```json
   // .eslintrc — the guarantee that survives your next teammate
   "no-restricted-imports": ["error", {
     "patterns": [{ "group": ["**/lib/db"], "message":
       "Handlers must use req.repos (tenant-scoped). Import db only in src/repos/." }]
   }]
   ```
7. Enable **RLS** on all tenant tables, with `SET LOCAL app.merchant_id` inside each transaction.
8. `GET /v1/me`, `GET /v1/customers`, `POST /v1/customers`.

**Demonstrate:**
```bash
curl -H "Authorization: Bearer sk_test_..." localhost:3000/v1/me           # 200
curl localhost:3000/v1/me                                                   # 401 + WWW-Authenticate
# Two merchants, two keys — B must not see A's customer:
curl -H "Authorization: Bearer $KEY_B" localhost:3000/v1/customers/$A_CUSTOMER   # 404
```
**Test:** the **authorization matrix** ([L15 §7](../03-security/15-authorization-and-multitenancy.md)) with the route-coverage meta-test. It has two routes now — add to it every phase, and it will never be a retrofit.

> **Lessons used:** 07 (URLs), 11 (strict schemas), 12 (API keys), 15 (tenancy), 25 (OpenAPI).

---

## Phase 2 — Payments & the state machine (6–7 h)

**Goal:** the core domain, with transitions that can't race.

**Build:**
1. `src/domain/money.ts` — the `Money` type from [L05 §8](../01-foundations/05-data-formats.md), with the `split` remainder test.
2. `src/domain/payment-state.ts` — the transition table as **pure data**:
   ```ts
   export const TRANSITIONS = {
     requires_payment_method: ["requires_confirmation", "canceled"],
     requires_confirmation:   ["processing", "canceled"],
     processing:              ["succeeded", "requires_capture", "failed"],
     requires_capture:        ["succeeded", "canceled"],
     succeeded:               ["partially_refunded", "refunded"],
     partially_refunded:      ["partially_refunded", "refunded"],
     refunded: [], failed: [], canceled: [],
   } as const satisfies Record<PaymentStatus, readonly PaymentStatus[]>;

   export const canTransition = (from: PaymentStatus, to: PaymentStatus) =>
     (TRANSITIONS[from] as readonly string[]).includes(to);
   ```
   Pure, exhaustively unit-testable, and the **single source of truth** — the DDL, the docs and the error messages all derive from it.
3. Input DTOs with `.strict()`; output mappers as **positive allowlists** ([L16 §API3](../03-security/16-owasp-and-hardening.md)).
4. `POST /v1/payments` → 201 + `Location`.
5. `GET /v1/payments/{id}` with `ETag` + `If-None-Match` → 304.
6. `PATCH /v1/payments/{id}` (metadata/description only) with `If-Match` → 412/428.
7. `POST /v1/payments/{id}/confirm|capture|cancel` — **every transition guarded in the `WHERE` clause** ([L09 §4](../02-rest-design/09-writes-patch-and-bulk.md)):
   ```ts
   const { rows } = await db.query(
     `UPDATE payments SET status=$3, version=version+1, updated_at=now()
       WHERE id=$1 AND merchant_id=$2 AND status=$4 RETURNING *`,
     [id, merchantId, "succeeded", "requires_capture"]);
   if (rows.length === 0) { /* 404 if absent, else 409 with current_status */ }
   ```
8. `FakeProcessor` with configurable latency, decline rate and timeout — you need it to build Phase 5 honestly.

**Demonstrate:** the full lifecycle by `curl`: create → confirm → capture → read. Then try an illegal transition and get a `409` naming both statuses.

**Test:** mass assignment is rejected (`{"status":"succeeded","merchant_id":"..."}` → 422); double capture concurrently → `[200, 409]` with `captured_count === 1`; boundary values on `amount_minor` including `9007199254740993`.

> **Lessons used:** 03 (methods/status), 05 (money), 09 (writes, concurrency), 16 (mass assignment), 17 (ETag).

---

## Phase 3 — Collections at scale (4–5 h)

**Goal:** a list endpoint that survives 4 million rows.

**Build:**
1. `src/domain/cursor.ts` — encode/decode with the **query fingerprint** ([L08 §3](../02-rest-design/08-collections-and-pagination.md)).
2. `GET /v1/payments` with allowlisted filters (`status`, `currency`, `amount[gte]`, `created[gte]`, `customer`), allowlisted sorts, bounded `limit`, `limit+1` for `has_more`, and **unknown query params → 400**.
3. The composite indexes from the blueprint, **with `merchant_id` leading**.
4. `GET /v1/payments/{id}/refunds`, `GET /v1/customers`, `GET /v1/events` — same machinery, reused.

**Demonstrate — and this is the phase where you generate your interview numbers:**
```bash
# Seed 3M payments for one merchant
npm run seed -- --merchant $M --count 3000000

# Then measure. Write these four numbers in DECISIONS.md.
psql -c "EXPLAIN ANALYZE SELECT * FROM payments WHERE merchant_id='$M'
         ORDER BY created_at DESC, id DESC LIMIT 20 OFFSET 0"
psql -c "EXPLAIN ANALYZE ... OFFSET 2000000"
psql -c "EXPLAIN ANALYZE SELECT * FROM payments WHERE merchant_id='$M'
         AND (created_at,id) < ('...', '...') ORDER BY created_at DESC, id DESC LIMIT 20"
psql -c "EXPLAIN ANALYZE SELECT count(*) FROM payments WHERE merchant_id='$M'"
```

**Test:** iterate every page of 3M rows via cursors and assert **zero duplicates and zero gaps**; then insert rows mid-iteration and assert the invariant still holds (the drift test — offset pagination fails this and cursors pass).

> **Lessons used:** 08 (the whole lesson), 17 (indexes, N+1).

---

## Phase 4 — Idempotency & the ledger (5–6 h)

**Goal:** retries are safe, and money reconciles.

**Build:**
1. `idempotency_keys` migration + the middleware from [L18 §5](../04-production/18-reliability-and-idempotency.md). All five details: tenant+endpoint scoping, canonical request hash, atomic `INSERT … ON CONFLICT` claim, stored response, **5xx releases the key**.
2. Apply it to `POST /payments`, `/confirm`, `/capture`, `/refunds`, `/payouts` — required, not optional.
3. `ledger_entries` migration; `postTransaction()` that writes a **balanced set** in one transaction and asserts `SUM(credits) − SUM(debits) === 0` before committing.
4. `GET /v1/balance` (a `SUM`, with `Cache-Control: no-store`) and `GET /v1/balance/transactions` (cursor-paginated).
5. `POST /v1/payments/{id}/refunds` — atomically increments `amount_refunded_minor`, transitions the payment, and posts reversing entries. Note the DB `CHECK` makes over-refunding impossible even if your logic is wrong.
6. The idempotency-key sweeper worker (24h TTL).

**Demonstrate:**
```bash
KEY=$(uuidgen)
curl -X POST localhost:3000/v1/payments -H "Idempotency-Key: $KEY" -d "$BODY"   # 201
curl -X POST localhost:3000/v1/payments -H "Idempotency-Key: $KEY" -d "$BODY"   # 200, identical body
curl -X POST localhost:3000/v1/payments -H "Idempotency-Key: $KEY" -d "$OTHER"  # 422 idempotency_key_reused
curl localhost:3000/v1/balance                                                   # reflects exactly one payment
```

**Test — the two that matter most in the whole project:**
```ts
test.each(Array.from({length: 20}, (_, i) => i))("concurrent create #%i → exactly one", async () => {
  const key = crypto.randomUUID();
  const rs = await Promise.all(Array.from({length: 10}, () => post(body, key)));
  expect(rs.filter(r => r.status === 201)).toHaveLength(1);
  expect(await countPayments(m.id)).toBe(1);
});

test("ledger reconciles after a randomised transaction sequence", async () => {
  await runRandomOps(500);                            // payments, refunds, payouts, cancels
  const { credits, debits } = await sumLedger(m.id);
  expect(credits - debits).toBe(0);                   // ← the invariant
  expect(await computeBalance(m.id)).toBe(await expectedBalanceFromEvents(m.id));
});
```

> **Lessons used:** 09 (atomicity), 17 (no-store on balance), 18 (idempotency, outbox thinking).

---

## Phase 5 — Events & webhooks (7–8 h)

**Goal:** the hardest and most impressive part of the project.

**Build:**
1. `events` + `outbox` migrations. **Every state change writes an event row and an outbox row in the same transaction** as the change ([L18 §7](../04-production/18-reliability-and-idempotency.md)).
2. `GET /v1/events?after=&type=` — cursor-paginated. **Build this before webhooks**; it's the recovery path everything else leans on.
3. `webhook_endpoints` CRUD with `assertSafeUrl` (the full SSRF guard from [L16 §API7](../03-security/16-owasp-and-hardening.md)) plus a verification challenge.
4. Dispatcher worker: polls the outbox, matches endpoints by event-type glob, creates delivery rows.
5. Sender worker: HMAC over `timestamp.rawBody`, 8s timeout, the backoff schedule with jitter, per-endpoint concurrency cap of 5, truncated response storage.
6. Dead-letter + auto-disable after 8 consecutive failures + a notification.
7. `GET .../deliveries`, `POST .../deliveries/{id}/retry`, `POST .../rotate-secret` (two live secrets).
8. **A receiver, too** — a small separate app that verifies signatures, dedupes on event ID, and acks fast ([L23 §8](../05-beyond-rest/23-realtime-and-webhooks.md)). Building both sides is what makes the signing rules stick.

**Demonstrate:** run the receiver on another port, create a payment, watch the signed webhook arrive and verify. Then stop the receiver, create another payment, and watch retries follow the schedule. Restart it and watch delivery succeed. Then `GET /v1/events?after=` and confirm nothing was lost.

**Test:** signature verification rejects a tampered body and a stale timestamp; the receiver dedupes a replayed delivery; a 5xx receiver retries and a 4xx receiver doesn't; `410 Gone` disables the endpoint immediately; a slow endpoint doesn't starve the others (the bulkhead).

> **Lessons used:** 12 (HMAC), 16 (SSRF), 18 (outbox, backoff), 23 (the whole lesson).

---

## Phase 6 — Limits, resilience, observability (5–6 h)

**Goal:** it survives load and you can debug it.

**Build:**
1. `token_bucket.lua` + layered limits (key → merchant → endpoint → global), cost-weighted, `RateLimit-*` on every response, `Retry-After` on 429, `scope` in the error body ([L19](../04-production/19-rate-limiting.md)). **Fail open** generally; **fail closed** on `/auth/*`.
2. Resilience around `FakeProcessor`: timeout → bulkhead → circuit breaker → retry with full jitter and a budget, **in that nesting order** ([L18 §10](../04-production/18-reliability-and-idempotency.md)).
3. Metrics: RED with bounded labels, plus `payments_created_total`, `db_queries_per_request`, `circuit_breaker_open`, `outbox_pending_total`, `webhook_deliveries_total`.
4. OpenTelemetry with auto-instrumentation plus manual spans around processor calls.
5. `/metrics` behind internal auth. Prometheus + Grafana in Compose.
6. Cache headers everywhere ([L17 §7](../04-production/17-caching-and-performance.md)) — including the deliberate `no-store` on `/balance`.

**Demonstrate:** `k6` with the ramp/steady/spike/recover profile. Find your knee. Watch the rate limiter engage **before** Postgres saturates — if it doesn't, your limits are set wrong, and that's the finding.

**Test:** the resilience suite from [L26 §6](../06-craft/26-testing-apis.md) — circuit opens after 5 failures, open circuit fails in <50ms, Redis down fails open on payments and closed on auth, pool exhaustion returns 503 rather than hanging.

> **Lessons used:** 17, 18, 19, 20, 26.

---

## Phase 7 — OAuth, docs, deploy (6–7 h)

**Goal:** it's a real, public, documented product.

**Build:**
1. **OAuth 2.1 authorization server** ([L14 §10](../03-security/14-oauth2-and-oidc.md)): `/oauth/authorize` + `/oauth/token`, exact redirect-URI matching, 60s single-use codes bound to `client_id`+`redirect_uri`+`code_challenge`, PKCE required, rotating refresh tokens with **family revocation on reuse**, a consent screen, and a connected-apps page.
2. Scope enforcement on every route (`requireScope`).
3. Session auth for the Console: access token in memory + refresh in an `HttpOnly` cookie ([L13 §6](../03-security/13-sessions-and-jwt.md)).
4. Version pinning: `Ledger-Version` header, defaulting to the merchant's pinned version.
5. Docs: Scalar or Redoc from `openapi.yaml`, plus **hand-written guides** — a 5-minute quickstart, "handling failed payments", and "receiving webhooks" (the failure guides are the valuable ones).
6. `docs/errors.md` with the `retryable` column; `docs/api-policy.md` with the compatibility policy.
7. Deploy: Dockerfile (multi-stage, non-root), migrations on deploy, HTTPS, real domain. Fly.io / Railway / Render are all fine.
8. CI: the pipeline from [L26 §7](../06-craft/26-testing-apis.md), including `oasdiff breaking` as a **required** check.

**Demonstrate:** the OAuth flow end to end in a browser. Then hand someone the docs URL and a test key and see if they can make a successful call in under 5 minutes without asking you anything. **That's the real test of an API**, and it will find problems nothing else does.

> **Lessons used:** 11, 13, 14, 25, 26, 27.

---

## Phase 8 — Review, harden, present (4–5 h)

**Goal:** turn the project into interview leverage.

**Build:**
1. **Run the 10-pass review on your own code** ([L27](../06-craft/27-api-review-checklist.md)). Write graded findings into `docs/review-findings.md`. **Fix every 🔴 and 🟠.** You will find things — that's the exercise working.
2. Run the attack list from [L15 §10](../03-security/15-authorization-and-multitenancy.md) and [L16 §6](../03-security/16-owasp-and-hardening.md). Every finding becomes a permanent test.
3. Schemathesis with 2,000 examples. **Fix every 500.**
4. `docs/api-architecture.md` from your `DECISIONS.md` — including the **"explicitly rejected"** and **"triggers that would change this"** sections. Those two sections are what make it a senior artifact.
5. A README with: what it is, why the domain, an architecture diagram, the measured numbers, and a 60-second quickstart.
6. **The 5-minute demo script.** Write it down and rehearse it:
   ```
   1. Here's the docs site. Here's a test key. (10s)
   2. Create a payment with curl.                                    (30s)
   3. Retry it with the same Idempotency-Key — 200, identical body,
      one row. Here's the concurrent test that runs 10 at once.      (60s)
   4. Here's the webhook arriving, signed. Here's the delivery log.  (60s)
   5. I stop the receiver. Watch the retry schedule. Here's the DLQ,
      and here's the event log the merchant uses to self-recover.    (60s)
   6. Balance is a SUM of append-only entries — here's the
      reconciliation test asserting it sums to zero.                 (30s)
   7. As merchant B, I request A's payment: 404, not 403. Here's the
      authorization matrix that fails CI if a route isn't covered.    (30s)
   ```

**The presentation rule:** lead with the **hard** parts (idempotency, webhooks, tenancy, reconciliation), never with CRUD. Interviewers have seen a thousand CRUD demos. Nobody shows them a concurrent-idempotency test that asserts exactly one row exists.

---

## The build order, and why it's this order

| Phase | Why here |
|---|---|
| 0 Foundations | Error handling and correlation must exist before features, or you retrofit them into 40 handlers |
| 1 Contract & tenancy | Tenancy is structural. Adding it later means auditing every query |
| 2 Payments | The domain core; needs tenancy but nothing else |
| 3 Collections | Needs data to exist before pagination is meaningful |
| 4 Idempotency & ledger | Needs writes to exist to be retried |
| 5 Events & webhooks | Needs state changes to emit; the outbox needs transactions |
| 6 Limits & observability | Needs traffic to limit and behaviour to observe |
| 7 OAuth & deploy | The last layer before it's public |
| 8 Review | Only meaningful once there's something to review |

**The two phases people get wrong:** they leave **tenancy** for later (Phase 1 exists for a reason — retrofitting it is a full audit) and they leave **observability** to the end (Phase 0's correlation IDs must be first, or every log line lacks context).

---

## If you have less time

| Available | Build phases | You still get |
|---|---|---|
| ~15 h | 0, 1, 2, 4 | Payments + tenancy + idempotency. **The single most interviewable part** |
| ~25 h | + 3, 5 | Pagination at scale + the full webhook system |
| ~40 h | + 6, 7 | Production-grade and deployed |
| ~50 h | + 8 | Interview-ready with evidence |

**If you only build one thing, build Phase 4.** Idempotency with a concurrent test asserting exactly one row is the highest-value 5 hours in this entire track.

---

## What's next

You've built it. Module 8 converts everything — the 27 technique lessons and the project — into interview performance: a 200+ question rapid-fire bank, a repeatable framework for design rounds, and 40 production scenarios.

Next → **[Lesson 30: Rapid-fire master Q&A](../08-interview/30-rapid-fire-master-qa.md)**
