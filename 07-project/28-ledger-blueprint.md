# Lesson 28 — Ledger blueprint

> **Why this lesson exists:** every technique in Modules 1–6 needs somewhere to land. **Ledger** is a payments API — Stripe-shaped, yours — chosen because the payments domain *forces* every hard API concept for real reasons. This is the complete blueprint: domain, data model, API surface, architecture and the decisions with their justifications, so that when an interviewer asks *"walk me through something you built"* you have a system you can defend at any depth.

**Time:** ~60 minutes to read and internalise · **Prereq:** Modules 1–6

---

## 1. What Ledger is

A payments platform for merchants. A merchant integrates Ledger, creates payments, money moves, they get webhooks, and funds land in a balance they can pay out.

```
   Merchant server ──► POST /v1/payments (Idempotency-Key)
                             │
                       payment_intent created (requires_payment_method)
                             │
   Customer ──► pays ──► confirm ──► processor (mocked) ──► succeeded
                             │
                    ┌────────┴────────┐
              ledger entries      webhook event
              (double-entry)      payment.succeeded
                    │                  │
              balance updated    delivered w/ HMAC, retries, DLQ
                    │
              POST /v1/payouts ──► money out
```

**The deliberate scope boundary:** the actual card processor is **mocked** (a `FakeProcessor` with configurable latency and failure modes). Real card processing requires PCI compliance and a licence, and none of the *API engineering* you're learning depends on it. Say this explicitly in an interview — knowing what to fake and why is a scoping judgement, not a shortcut.

### Why this domain, again

| The domain forces… | Which teaches… |
|---|---|
| Never charge twice on a retry | **Idempotency keys** ([L18](../04-production/18-reliability-and-idempotency.md)) |
| Money must be exact | Integer minor units, decimal discipline ([L05](../01-foundations/05-data-formats.md)) |
| Tell the merchant asynchronously | **Webhooks** with signing, retries, DLQ ([L23](../05-beyond-rest/23-realtime-and-webhooks.md)) |
| A payment can't go backwards | **State machines in the WHERE clause** ([L09](../02-rest-design/09-writes-patch-and-bulk.md)) |
| Merchant A never sees merchant B | **Multi-tenancy, BOLA** ([L15](../03-security/15-authorization-and-multitenancy.md)) |
| 4M payments per merchant | **Cursor pagination + indexes** ([L08](../02-rest-design/08-collections-and-pagination.md)) |
| Balances must always reconcile | **Append-only ledger** — conflict-free by design ([L09](../02-rest-design/09-writes-patch-and-bulk.md)) |
| Never break a 2019 integration | **Versioning & evolution** ([L11](../02-rest-design/11-versioning-and-evolution.md)) |
| One merchant can't starve others | **Rate limits, bulkheads** ([L19](../04-production/19-rate-limiting.md)) |
| Card testing must be blocked | **Business-flow abuse** ([L16](../03-security/16-owasp-and-hardening.md)) |

---

## 2. Domain model

### Entities and relationships
```
merchant 1──* api_key
         1──* user            (dashboard logins, with roles)
         1──* customer 1──* payment_method
         1──* payment  1──* refund
                       1──* ledger_entry
         1──* payout
         1──* webhook_endpoint 1──* webhook_delivery
         1──* event
```

### The payment state machine — the core of the domain

```
                       ┌──────────────────────────┐
   create ────────────►│ requires_payment_method  │
                       └────────────┬─────────────┘
                          attach pm │
                       ┌────────────▼─────────────┐
                       │   requires_confirmation  │
                       └────────────┬─────────────┘
                            confirm │
                       ┌────────────▼─────────────┐
                  ┌────│       processing         │────┐
                  │    └──────────────────────────┘    │
      capture_method=  │                               │ processor declines
      manual │         │ automatic & ok                │
    ┌────────▼──────┐  │                    ┌──────────▼──────┐
    │requires_capture│ │                    │     failed      │  (terminal)
    └────────┬──────┘  │                    └─────────────────┘
     capture │         │
    ┌────────▼─────────▼───┐   refund (partial)  ┌──────────────────────┐
    │      succeeded       │────────────────────►│ partially_refunded   │
    └────────┬─────────────┘                     └──────────┬───────────┘
             │ refund (full)                                │ refund remainder
             │            ┌──────────────┐                  │
             └───────────►│   refunded   │◄─────────────────┘   (terminal)
                          └──────────────┘
   Any pre-success state ──cancel──► canceled  (terminal)
```

**Every transition is enforced in SQL, not in application code:**
```sql
UPDATE payments SET status='succeeded', captured_at=now(), version=version+1
 WHERE id=$1 AND merchant_id=$2 AND status='requires_capture'
 RETURNING *;
-- 0 rows → 404 if it doesn't exist, else 409 with the current status.
```
That's the atomic-transition rule from [Lesson 09](../02-rest-design/09-writes-patch-and-bulk.md), and it's why Ledger cannot double-capture even under concurrent requests.

### Double-entry ledger — why balances are append-only

The naïve design is a `balance` column you `UPDATE`. Don't. Three reasons, and they're the design's best interview story:
1. **Concurrent updates lose money.** Two simultaneous `UPDATE balance SET amount = amount + X` are safe in SQL, but `SELECT` → compute → `UPDATE` isn't, and that's what application code does.
2. **You can never answer "why is the balance this?"** A number with no history is unauditable — unacceptable in a financial system.
3. **You can't reconcile.** With entries, `SUM(entries) == balance` is a check you can run continuously; with a column there's nothing to check against.

```sql
-- Every money movement writes BALANCED entries. They must sum to zero.
-- payment succeeded: 4999 usd, fee 175
INSERT INTO ledger_entries (account, direction, amount_minor, currency, payment_id, ...) VALUES
  ('merchant:mrc_1:available', 'credit', 4824, 'usd', 'pi_1', ...),   -- net to merchant
  ('platform:fees',            'credit',  175, 'usd', 'pi_1', ...),   -- our fee
  ('processor:acme_uk',        'debit',  4999, 'usd', 'pi_1', ...);   -- source
-- SUM(credits) - SUM(debits) = 0, always. A reconciliation job asserts this.
```
Balance is a **query** (`SUM`), optionally with a materialised snapshot for speed. Never a mutable field.

---

## 3. Data model (the real DDL)

```sql
-- ─────────────── tenancy ───────────────
CREATE TABLE merchants (
  id            uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  name          text NOT NULL,
  country       char(2) NOT NULL,
  default_currency char(3) NOT NULL,
  status        text NOT NULL DEFAULT 'active',    -- active|suspended
  api_version   text NOT NULL,                     -- pinned at signup — L11
  created_at    timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE api_keys (
  id            text PRIMARY KEY,                  -- 'ak_...' internal id
  public_id     text NOT NULL UNIQUE,              -- lookup half — O(1) verify, L12
  merchant_id   uuid NOT NULL REFERENCES merchants(id),
  key_hash      text NOT NULL,                     -- SHA-256 of the secret half
  last4         text NOT NULL,
  env           text NOT NULL,                     -- live|test  ← isolation
  scopes        text[] NOT NULL,
  last_used_at  timestamptz,
  revoked_at    timestamptz,
  created_at    timestamptz NOT NULL DEFAULT now()
);

-- ─────────────── customers ───────────────
CREATE TABLE customers (
  id            text PRIMARY KEY,                  -- 'cus_...'
  merchant_id   uuid NOT NULL REFERENCES merchants(id),
  email         text,
  name          text,
  phone         text,
  metadata      jsonb NOT NULL DEFAULT '{}',
  version       int  NOT NULL DEFAULT 1,           -- ETag/If-Match — L09
  deleted_at    timestamptz,
  created_at    timestamptz NOT NULL DEFAULT now(),
  updated_at    timestamptz NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX ON customers (merchant_id, lower(email)) WHERE deleted_at IS NULL;

-- ─────────────── payments ───────────────
CREATE TABLE payments (
  id                text PRIMARY KEY,              -- 'pi_...'
  merchant_id       uuid NOT NULL REFERENCES merchants(id),
  customer_id       text REFERENCES customers(id),
  payment_method_id text REFERENCES payment_methods(id),
  amount_minor      bigint NOT NULL CHECK (amount_minor > 0),
  currency          char(3) NOT NULL,
  amount_refunded_minor bigint NOT NULL DEFAULT 0
      CHECK (amount_refunded_minor >= 0 AND amount_refunded_minor <= amount_minor),
  status            text NOT NULL,
  capture_method    text NOT NULL DEFAULT 'automatic',   -- automatic|manual
  description       text,
  metadata          jsonb NOT NULL DEFAULT '{}',
  failure_code      text,
  processor         text,
  processor_ref     text,
  version           int NOT NULL DEFAULT 1,
  captured_at       timestamptz,
  created_at        timestamptz NOT NULL DEFAULT now(),
  updated_at        timestamptz NOT NULL DEFAULT now()
);
-- Indexes: tenant column FIRST, plus the unique tiebreaker — L08
CREATE INDEX ON payments (merchant_id, created_at DESC, id DESC);
CREATE INDEX ON payments (merchant_id, status, created_at DESC, id DESC);
CREATE INDEX ON payments (merchant_id, amount_minor DESC, id DESC);
CREATE INDEX ON payments (merchant_id, customer_id, created_at DESC, id DESC);

CREATE TABLE refunds (
  id            text PRIMARY KEY,                  -- 're_...'
  merchant_id   uuid NOT NULL REFERENCES merchants(id),
  payment_id    text NOT NULL REFERENCES payments(id),
  amount_minor  bigint NOT NULL CHECK (amount_minor > 0),
  currency      char(3) NOT NULL,
  reason        text,                              -- requested_by_customer|duplicate|fraudulent
  status        text NOT NULL,                     -- pending|succeeded|failed
  created_at    timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX ON refunds (merchant_id, payment_id);

-- ─────────────── the ledger (append-only) ───────────────
CREATE TABLE ledger_entries (
  id            bigserial PRIMARY KEY,
  merchant_id   uuid NOT NULL REFERENCES merchants(id),
  account       text NOT NULL,                     -- merchant:<id>:available | :pending | platform:fees
  direction     text NOT NULL CHECK (direction IN ('debit','credit')),
  amount_minor  bigint NOT NULL CHECK (amount_minor > 0),
  currency      char(3) NOT NULL,
  transaction_id text NOT NULL,                    -- groups the balanced set
  payment_id    text REFERENCES payments(id),
  refund_id     text REFERENCES refunds(id),
  payout_id     text REFERENCES payouts(id),
  created_at    timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX ON ledger_entries (merchant_id, account, currency, created_at);
CREATE INDEX ON ledger_entries (transaction_id);
-- NO UPDATE, NO DELETE. Corrections are new, offsetting entries.

-- ─────────────── idempotency (L18) ───────────────
CREATE TABLE idempotency_keys (
  key             text NOT NULL,
  merchant_id     uuid NOT NULL,
  endpoint        text NOT NULL,
  request_hash    text NOT NULL,
  state           text NOT NULL,                   -- in_progress|completed
  response_status int,
  response_body   jsonb,
  resource_id     text,
  locked_until    timestamptz,
  created_at      timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (merchant_id, endpoint, key)
);
CREATE INDEX ON idempotency_keys (created_at);      -- 24h TTL sweep

-- ─────────────── events & webhooks (L23) ───────────────
CREATE TABLE events (
  id            text PRIMARY KEY,                  -- 'evt_...' (ULID → time-sortable)
  merchant_id   uuid NOT NULL REFERENCES merchants(id),
  sequence      bigserial NOT NULL,                -- monotonic; clients detect gaps
  type          text NOT NULL,                     -- payment.succeeded, refund.created, ...
  api_version   text NOT NULL,
  livemode      boolean NOT NULL,
  data          jsonb NOT NULL,
  created_at    timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX ON events (merchant_id, sequence);
CREATE INDEX ON events (merchant_id, type, created_at DESC);

CREATE TABLE outbox (                               -- same transaction as the state change
  id            bigserial PRIMARY KEY,
  event_id      text NOT NULL REFERENCES events(id),
  processed_at  timestamptz
);
CREATE INDEX ON outbox (id) WHERE processed_at IS NULL;

CREATE TABLE webhook_endpoints (
  id              text PRIMARY KEY,                -- 'we_...'
  merchant_id     uuid NOT NULL REFERENCES merchants(id),
  url             text NOT NULL,
  secret          text NOT NULL,                   -- encrypted at rest
  previous_secret text,                            -- rotation window
  enabled_events  text[] NOT NULL,                 -- ['payment.*']
  api_version     text NOT NULL,                   -- pinned per endpoint — L11
  status          text NOT NULL DEFAULT 'active',  -- active|failing|disabled
  consecutive_failures int NOT NULL DEFAULT 0,
  created_at      timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE webhook_deliveries (
  id              text PRIMARY KEY,                -- 'wd_...'
  endpoint_id     text NOT NULL REFERENCES webhook_endpoints(id),
  event_id        text NOT NULL REFERENCES events(id),
  attempt         int NOT NULL DEFAULT 0,
  status          text NOT NULL,                   -- pending|delivered|failed|dead
  next_attempt_at timestamptz,
  response_status int,
  response_body   text,                            -- truncated to 2KB
  duration_ms     int,
  created_at      timestamptz NOT NULL DEFAULT now(),
  UNIQUE (endpoint_id, event_id)
);
CREATE INDEX ON webhook_deliveries (next_attempt_at) WHERE status = 'pending';

-- ─────────────── row-level security (L15) ───────────────
ALTER TABLE payments  ENABLE ROW LEVEL SECURITY;
ALTER TABLE payments  FORCE  ROW LEVEL SECURITY;
CREATE POLICY tenant ON payments
  USING (merchant_id = current_setting('app.merchant_id', true)::uuid);
-- repeat for customers, refunds, payouts, ledger_entries, events, webhook_*
```

**Five details worth noticing**, because each one is a lesson made concrete:
1. `merchant_id` is the **leading column of every index** — it's the most selective predicate on every query.
2. `version` columns on mutable entities power `ETag`/`If-Match`.
3. `CHECK (amount_refunded_minor <= amount_minor)` — an invariant the database enforces, so no application bug can over-refund.
4. `UNIQUE (endpoint_id, event_id)` — a dispatcher bug can't create duplicate delivery records.
5. `events.sequence` is a `bigserial`, so clients can detect gaps and know they missed something.

---

## 4. The API surface

Full URL set in [Lesson 07 §6](../02-rest-design/07-resource-modelling-and-urls.md). The additions specific to this blueprint:

```
# Payment lifecycle
POST   /v1/payments                       Idempotency-Key required  → 201
POST   /v1/payments/{id}/confirm          Idempotency-Key required  → 200
POST   /v1/payments/{id}/capture          Idempotency-Key required  → 200
POST   /v1/payments/{id}/cancel                                     → 200
POST   /v1/payments/{id}/refunds          Idempotency-Key required  → 201

# Money
GET    /v1/balance                        no-store — L17
GET    /v1/balance/transactions           cursor-paginated ledger view
POST   /v1/payouts                        Idempotency-Key required  → 202

# Events & webhooks
GET    /v1/events                         ?after=evt_x&type=payment.*   ← the recovery path
GET    /v1/events/{id}
GET    /v1/webhook-endpoints/{id}/deliveries
POST   /v1/webhook-endpoints/{id}/deliveries/{did}/retry
POST   /v1/webhook-endpoints/{id}/rotate-secret

# Platform
GET    /v1/openapi.json
GET    /healthz         (liveness — no dependency checks)
GET    /readyz          (readiness — checks DB + Redis)
GET    /metrics         (internal auth only)
```

### Event types
```
payment.created              payment.succeeded         payment.failed
payment.canceled             payment.captured          payment.processing
refund.created               refund.succeeded          refund.failed
payout.created               payout.paid               payout.failed
customer.created             customer.updated          customer.deleted
```
Merchants subscribe with globs (`payment.*`). Events are **fat with a thin escape hatch** ([L23](../05-beyond-rest/23-realtime-and-webhooks.md)): the full object plus `id`/`type` so sophisticated receivers can re-fetch.

---

## 5. Architecture

```
                    ┌───────────────────────────────────────┐
   Console (React) ─┤                                        │
   Merchant servers ┤   API  (Express + TypeScript)          │
   Partners (OAuth) ┤   ├─ correlation → helmet → cors        │
                    │   ├─ rate limit (Redis + Lua)           │
                    │   ├─ authenticate → authorize           │
                    │   ├─ validate (OpenAPI + zod)           │
                    │   ├─ idempotency                        │
                    │   └─ handlers → scoped repos            │
                    └──────────┬───────────────┬─────────────┘
                               │               │
                    ┌──────────▼──────┐  ┌─────▼──────────┐
                    │   Postgres 16   │  │    Redis 7     │
                    │  + RLS          │  │ rate limits,   │
                    │  + outbox       │  │ cache, locks   │
                    └──────────┬──────┘  └────────────────┘
                               │ outbox polling
                    ┌──────────▼──────────────────────────┐
                    │  Workers                             │
                    │  ├─ webhook dispatcher + sender      │
                    │  ├─ payout processor                 │
                    │  ├─ idempotency-key sweeper (24h)    │
                    │  └─ reconciliation (SUM(entries)==0) │
                    └──────────┬──────────────────────────┘
                               │
                    ┌──────────▼──────┐
                    │ FakeProcessor   │  configurable latency,
                    │ (mocked)        │  decline rates, timeouts
                    └─────────────────┘
```

### Project layout
```
ledger/
├── openapi.yaml                     ← the contract. Written first
├── src/
│   ├── app.ts                       middleware assembly (order matters — L02)
│   ├── config.ts                    env validation with zod, fail fast at boot
│   ├── middleware/                  correlation, auth, ratelimit, idempotency, errors
│   ├── domain/                      PURE logic, no I/O: money, state machine, cursors, hmac
│   ├── repos/                       scoped repositories — tenant is mandatory (L15)
│   ├── services/                    orchestration + transactions
│   ├── routes/                      thin handlers: parse → call service → map to DTO
│   ├── dto/                         input schemas (.strict()) + output mappers (allowlist)
│   ├── workers/                     webhook sender, payouts, sweeper, reconciliation
│   └── lib/                         db, redis, logger, metrics, tracing, resilience
├── test/
│   ├── setup.ts                     testcontainers + real migrations
│   ├── fixtures.ts                  builders
│   ├── integration/                 per-endpoint suites
│   ├── authz-matrix.test.ts         actor × endpoint + route-coverage meta-test
│   └── properties/                  idempotency, concurrency, reconciliation
├── migrations/
├── docs/  errors.md · api-policy.md · api-architecture.md · review-findings.md
└── load/  k6 scripts
```

**The `domain/` boundary is deliberate:** anything that can be pure *is* pure — money arithmetic, the state-transition table, cursor encoding, HMAC signing, backoff calculation. Those get fast unit tests. Everything touching I/O lives in `repos/` and `services/` and gets integration tests. That split is what makes the test pyramid from [Lesson 26](../06-craft/26-testing-apis.md) natural rather than forced.

---

## 6. The decisions, with justifications

This table **is** your interview answer. Every row is a defensible choice, not a default.

| Decision | Choice | Why |
|---|---|---|
| API style | REST | Third-party developers expect it; reads are cacheable; per-endpoint limits matter ([L24](../05-beyond-rest/24-choosing-an-api-style.md)) |
| Versioning | `/v1` frozen + `Ledger-Version` date header pinned per merchant | Stripe's model: clients never break ([L11](../02-rest-design/11-versioning-and-evolution.md)) |
| IDs | Prefixed ULIDs (`pi_01HQ8ZK...`) | Opaque, non-sequential, time-sortable, good index locality, self-describing in logs ([L07](../02-rest-design/07-resource-modelling-and-urls.md)) |
| Money | `amount_minor` bigint + ISO 4217 `currency` | Floats are wrong; currency is required to interpret the integer ([L05](../01-foundations/05-data-formats.md)) |
| Auth | API keys (server) · session cookie + in-memory access token (Console) · OAuth 2.1 + PKCE (partners) · HMAC (webhooks) | Four client types, four correct mechanisms ([L12](../03-security/12-authentication-landscape.md)) |
| Tenancy | From the credential only; scoped repos + Postgres RLS | Structural BOLA prevention ([L15](../03-security/15-authorization-and-multitenancy.md)) |
| Pagination | Cursor, `has_more`, `total_count: null` | O(1) at 4M rows; no drift ([L08](../02-rest-design/08-collections-and-pagination.md)) |
| Updates | `PATCH` (merge-patch semantics) + `If-Match`; no `PUT` | Full replacement isn't wanted, and half-`PUT` is worse than none ([L09](../02-rest-design/09-writes-patch-and-bulk.md)) |
| Errors | RFC 9457 `problem+json` + stable `code` + `request_id` | Machine-actionable ([L10](../02-rest-design/10-errors-and-problem-details.md)) |
| Idempotency | Required header on every money-moving POST; 24h TTL; 5xx releases the key | Retries are inevitable ([L18](../04-production/18-reliability-and-idempotency.md)) |
| Balances | Append-only double-entry; balance is a `SUM` | Conflict-free, auditable, reconcilable |
| State transitions | Guarded in `UPDATE … WHERE` | Atomic; no race |
| Events | Outbox in the same transaction; at-least-once; `sequence` for gap detection | Can't atomically write DB + queue ([L18](../04-production/18-reliability-and-idempotency.md)) |
| Webhooks | HMAC over `timestamp.rawBody`; 8 attempts / ~3 days with jitter; DLQ + auto-disable; pollable event log | The full design from [L23](../05-beyond-rest/23-realtime-and-webhooks.md) |
| Ordering | **Not guaranteed**, documented; `sequence` provided | Ordered delivery ⇒ head-of-line blocking |
| Rate limiting | Token bucket in Redis via Lua; per key + merchant + endpoint + global; cost-weighted | Bursts are legitimate; atomic; bulk metered per item ([L19](../04-production/19-rate-limiting.md)) |
| Caching | `private, no-cache` + ETag on reads; **`no-store` on `/balance`** | Money must be exact ([L17](../04-production/17-caching-and-performance.md)) |
| Contract | OpenAPI 3.1, written first; response validation in CI; `oasdiff` gate | Drift is structurally impossible ([L25](../06-craft/25-openapi-contract-first.md)) |
| Observability | `X-Request-Id`, pino, RED metrics, OpenTelemetry, business metrics | Debuggable at 3am ([L20](../04-production/20-observability.md)) |
| Real card processing | **Out of scope** — mocked | Needs PCI + a licence; teaches nothing about API design |

### The five things you'll be able to say in an interview

1. *"I built a payments API with idempotency keys — here's the storage schema, here's why 5xx releases the key, and here's the test that fires ten concurrent identical requests and asserts exactly one row exists."*
2. *"Balances are append-only double-entry, so concurrent writes commute and there's nothing to lose. A reconciliation worker asserts `SUM(credits) − SUM(debits) = 0` continuously."*
3. *"Tenancy is enforced three ways: scoped repositories so an unscoped query can't be written, Postgres RLS as a backstop, and an authorization matrix test that fails CI if any route lacks coverage."*
4. *"Webhooks are at-least-once with HMAC signing, jittered backoff over three days, a dead-letter path, auto-disable with notification — and a pollable event log, so a merchant who was down for six hours can self-recover without contacting us."*
5. *"I deliberately don't guarantee webhook ordering, because guaranteeing it means serial delivery per endpoint, which turns one stuck event into a multi-day backlog. Instead I ship a sequence number and document that clients should treat a webhook as a signal to read current state."*

Each of those is a **mechanism plus a trade-off plus evidence**. That combination is what "senior" sounds like.

---

## 7. Deliberate scope boundaries

Knowing what you *didn't* build, and why, is as strong a signal as what you did.

| Not built | Reason |
|---|---|
| Real card processing | PCI DSS + licensing; teaches no API design |
| 3D Secure / SCA flows | Would be genuinely interesting; add later as a stretch goal |
| Multi-currency FX conversion | Adds rate-sourcing complexity, not API complexity |
| Subscriptions / recurring billing | A whole second domain. **A good phase-2 extension** |
| Dispute/chargeback lifecycle | Same |
| A full admin/back-office UI | The Console covers the merchant view |

If asked *"what would you add next?"*: **subscriptions**, because they'd force scheduled jobs, proration arithmetic, dunning retries, and a much harder state machine — a natural extension that reuses the whole foundation.

---

## 8. Definition of done

```
FUNCTIONAL
[ ] Full payment lifecycle: create → confirm → capture → refund → cancel
[ ] Customers + payment methods
[ ] Append-only ledger; balance as a SUM; payouts
[ ] Events + webhooks with signing, retries, DLQ, manual retry, secret rotation
[ ] API keys with scopes, test/live isolation, rotation
[ ] OAuth 2.1 + PKCE for partner apps

QUALITY
[ ] openapi.yaml written first; responses validated against it in CI
[ ] Authorization matrix + route-coverage meta-test passing
[ ] Idempotency + concurrency property tests, repeated 20× in CI
[ ] Reconciliation test: ledger sums to zero after a randomised transaction sequence
[ ] Schemathesis nightly with zero 500s
[ ] k6 load test with documented p99 and a known knee

OPERATIONS
[ ] Docker Compose for local dev; migrations in CI
[ ] Structured logs, RED metrics, traces, business metrics
[ ] /healthz, /readyz, /metrics
[ ] Rate limits with headers, per tier
[ ] Deployed to a real URL with HTTPS
[ ] Published docs (Scalar/Redoc) with a quickstart and an error table

EVIDENCE (for interviews)
[ ] docs/api-architecture.md — the decision doc, including rejections and triggers
[ ] docs/errors.md — every code, with a `retryable` column
[ ] docs/review-findings.md — your own review of your own work
[ ] Measured numbers: offset-vs-keyset, 200-vs-304 bytes, load-test p99
```

That **EVIDENCE** section is what turns a project into interview leverage. Anyone can say "I built a payments API." Very few can hand over a decision document that lists what they rejected and the trigger that would change their mind.

---

## What's next

The blueprint is the *what*. Next is the *how*: a phase-by-phase build order where each phase maps to specific lessons, so you're never building something you haven't just learned — and each phase ends in something demonstrable.

Next → **[Lesson 29: Ledger build log](29-ledger-build-log.md)**
