# Lesson 18 — Reliability: timeouts, retries, idempotency, circuit breakers

> **Why this lesson exists:** this is the most important lesson in the track, and **idempotency is the single most likely senior API interview question you will face.** The reason is that everything here follows from one unavoidable fact — *a network failure never tells you whether the operation happened* — and reasoning correctly from that fact is exactly what separates an engineer who has run a payment system from one who has read about one.

**Time:** ~100 minutes · **Prereq:** Lessons 03, 09

---

## 1. The idea in one sentence

> **When a request fails you do not know whether it succeeded — so the client must retry, and therefore the server must be able to recognise a retry. Every reliability mechanism in this lesson exists to make that safe.**

---

## 2. The fundamental ambiguity

```
Client                                      Server
  │  POST /v1/payments  {amount: 4999}        │
  │──────────────────────────────────────────▶│  ✅ charged the card
  │                                            │  ✅ wrote the row
  │                        ✗ timeout           │  ✅ sent the webhook
  │◀ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ (nothing) ─ ─ ─ ─ ─│  ✅ returned 201... into a dead socket
```

The client sees a timeout. **From the client's position, these four scenarios are indistinguishable:**

1. The request never arrived → nothing happened.
2. It arrived, but failed early → nothing happened.
3. It arrived and **fully succeeded**; the response was lost → **the customer was charged.**
4. It arrived, succeeded partially (charged, but the DB write failed) → **inconsistent state.**

The client's only two options are both wrong without help:
- **Don't retry** → in cases 1 and 2, the customer's payment silently didn't happen.
- **Retry** → in case 3, the customer is charged twice.

> **This is not an engineering oversight; it is a theoretical limit.** It's the Two Generals Problem: you cannot achieve certainty about a remote party's state over an unreliable channel with a finite number of messages. **"Exactly-once delivery" over a network does not exist.**
>
> What *does* exist is **at-least-once delivery plus idempotent processing**, which produces an *effectively-once* outcome. Saying that sentence correctly is a strong senior signal — and it's the honest answer when someone claims their queue does exactly-once.

---

## 3. Timeouts

**Every** network call needs a timeout. A call without one holds a socket, a thread or an event-loop handle **forever**, and that's how a slow dependency becomes a total outage: resources are consumed by requests nobody is waiting for.

### The timeout budget

Timeouts must **decrease** as you go downstream:

```
Client              30 s   (user patience)
  └─ API Gateway    25 s
      └─ Your API   20 s   ← your total budget
          ├─ Postgres        3 s
          ├─ Redis         200 ms
          └─ Payment processor  10 s
               └─ (its own retries must fit inside your 10 s)
```

**Why the ordering matters:** if Postgres' timeout were 30s while the gateway's is 25s, then at second 25 the gateway gives up and the client sees a 504 — while your process keeps holding a database connection for another 5 seconds, doing work for a request nobody will read. Under load, that's how you exhaust a connection pool.

**Deadline propagation** is the mature version: pass the remaining budget downstream so nobody starts work that can't finish in time.

```ts
export type Deadline = { at: number };                       // epoch ms

export function remaining(d: Deadline): number {
  return Math.max(0, d.at - Date.now());
}

export async function callDownstream(url: string, d: Deadline, minMs = 50) {
  const budget = remaining(d);
  if (budget < minMs) throw ApiError.gatewayTimeout("deadline_exceeded",
    "Not enough time budget remains to attempt this call");

  return fetch(url, {
    signal: AbortSignal.timeout(Math.min(budget, 10_000)),
    headers: { "x-request-deadline-ms": String(budget) },    // let them budget too
  });
}
```
gRPC has deadlines built in; over HTTP you pass a header. Either way, **the point is that a request carries its remaining time**, so a downstream service can refuse work it cannot deliver.

### The three timeouts people forget
| Timeout | Default in most stacks | Why it matters |
|---|---|---|
| **Connect timeout** | often none | A blackholed host hangs until TCP gives up (~2 min) |
| **Idle/socket timeout** | often none | The peer accepted but sends nothing — a "slow loris" |
| **Total/overall timeout** | often none | Individual timeouts can still sum past your budget |

And the pairing that produces mystery 502s (from Lesson 02): **the load balancer's idle timeout must be *shorter* than your app's keep-alive timeout**, or the app will hand back a connection the LB has already discarded.

---

## 4. Retries — and how they become an outage

### The three rules

**Rule 1: only retry what's safe to repeat.**

| Response | Retry? | Why |
|---|---|---|
| Network error / timeout | **Only if idempotent** | You don't know if it happened |
| `408`, `429` | Yes, after `Retry-After` | Explicitly transient |
| `500`, `502`, `503`, `504` | **Only if idempotent** | Server-side, probably transient |
| `400`, `401`, `403`, `404`, `409`, `422` | **Never** | It will fail identically forever |

**Rule 2: exponential backoff with full jitter.** Not optional — this is the difference between recovery and a self-inflicted DDoS.

```ts
export type RetryOpts = { maxAttempts?: number; baseMs?: number; maxMs?: number };

export async function withRetry<T>(
  fn: (attempt: number) => Promise<T>,
  isRetryable: (e: unknown) => boolean,
  { maxAttempts = 4, baseMs = 200, maxMs = 10_000 }: RetryOpts = {},
): Promise<T> {
  let lastError: unknown;
  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return await fn(attempt);
    } catch (e) {
      lastError = e;
      if (!isRetryable(e) || attempt === maxAttempts) throw e;

      // Exponential backoff with FULL jitter: random in [0, cap]. Not "exp + small random".
      const cap = Math.min(maxMs, baseMs * 2 ** (attempt - 1));
      const delay = Math.random() * cap;
      await sleep(delay);
    }
  }
  throw lastError;
}
```

**Why *full* jitter and not `delay + random(0..100ms)`?** Because the failure mode you're preventing is **synchronisation**. If 5,000 clients all fail at the same instant (your service restarted), they all wake at ~200ms, then ~400ms, then ~800ms — arriving in coordinated waves that re-kill the service each time. Full jitter spreads them uniformly across the whole window and flattens the load. AWS published the analysis; "full jitter" is the specific term to use.

**Rule 3: cap total attempts *and* keep a retry budget.**

The subtle killer is **retry amplification**. Three services deep, each retrying 3×, means one client request becomes **27** requests at the bottom. Your database sees 27× load *precisely when it is already struggling*. This is how a small blip becomes a full outage — and it's why:

- **Retry at one layer only** (usually the edge-most client), not at every hop.
- **Keep a retry budget**: cap retries at ~10% of total requests. If more than 10% of traffic is retries, stop retrying — the problem isn't transient.

```ts
/** Retry budget: a sliding-window ratio guard. Refuse to retry when the system is broadly unhealthy. */
class RetryBudget {
  private attempts = 0; private retries = 0; private windowStart = Date.now();
  constructor(private ratio = 0.1, private windowMs = 10_000) {}

  allow(): boolean {
    if (Date.now() - this.windowStart > this.windowMs) {
      this.attempts = 0; this.retries = 0; this.windowStart = Date.now();
    }
    return this.retries < Math.max(5, this.attempts * this.ratio);
  }
  record(isRetry: boolean) { this.attempts++; if (isRetry) this.retries++; }
}
```

> **The interview line:** *"Retries without jitter and a budget don't add resilience — they add a load multiplier that activates exactly when you're already failing."*

---

## 5. Idempotency keys — the main event

The mechanism that makes retrying a `POST` safe. Learn this well enough to design the storage on a whiteboard.

### The contract

```http
POST /v1/payments
Idempotency-Key: 8f14e45f-ea45-4b1a-9e21-3c5f2a1d0b77
Content-Type: application/json

{ "amount_minor": 4999, "currency": "usd", "customer": "cus_9s2k" }
```

The server guarantees:
1. **First request with this key** → process it, store the result against the key.
2. **Any later request with the same key and the same body** → return the **stored** result. Do not re-execute.
3. **Same key, *different* body** → `422 idempotency_key_reused`. The client has a bug; do not guess which one they meant.
4. **Same key, original still in flight** → `409 idempotency_conflict` (or block briefly), never execute twice.

### The storage design

```sql
CREATE TABLE idempotency_keys (
  key             text        NOT NULL,
  merchant_id     uuid        NOT NULL,           -- scope per tenant: keys are client-chosen
  endpoint        text        NOT NULL,           -- so the same key on a different route is distinct
  request_hash    text        NOT NULL,           -- SHA-256 of the canonical request body
  state           text        NOT NULL,           -- 'in_progress' | 'completed'
  response_status int,
  response_body   jsonb,
  resource_id     text,                            -- what was created, for auditing
  created_at      timestamptz NOT NULL DEFAULT now(),
  locked_until    timestamptz,                     -- crash recovery for stuck in_progress rows
  PRIMARY KEY (merchant_id, endpoint, key)
);
CREATE INDEX ON idempotency_keys (created_at);     -- for TTL cleanup
```

Five design decisions in that schema, each an interview answer:

1. **Scope the key by `merchant_id`.** Keys are chosen by clients; two merchants will eventually pick the same UUID, and one must not see the other's response. **This is a cross-tenant leak if you get it wrong**, and it's the detail most candidates miss.
2. **Include the `endpoint`.** The same key on `POST /payments` and `POST /refunds` are different operations.
3. **Store `request_hash`.** Without it, a client that reuses a key with a different body silently gets the *old* response — a genuinely dangerous outcome ("I asked to charge ₹100 and it said ₹5000 succeeded").
4. **Store the actual response**, status and body, so the replay is byte-identical to what the first attempt returned.
5. **`state` + `locked_until`** so a concurrent duplicate can be detected and a crashed request can be recovered rather than blocking that key forever.

### The implementation

```ts
import { createHash } from "node:crypto";

const TTL_HOURS = 24;
const LOCK_SECONDS = 60;

export function idempotent(endpoint: string, opts: { required?: boolean } = {}) {
  return async function (req: Request, res: Response, next: NextFunction) {
    const key = req.header("idempotency-key");

    if (!key) {
      if (opts.required) {
        throw ApiError.badRequest("idempotency_key_required",
          "Provide an Idempotency-Key header (a UUID) for this operation");
      }
      return next();
    }
    if (key.length < 8 || key.length > 255) {
      throw ApiError.badRequest("invalid_idempotency_key", "Key must be 8–255 characters");
    }

    const requestHash = createHash("sha256")
      .update(JSON.stringify(canonicalize(req.body)))     // stable key order → stable hash
      .digest("hex");

    // ── Atomically claim the key, or discover it already exists. ──
    // ON CONFLICT DO NOTHING + RETURNING gives us "did I win the race?" in ONE statement.
    const claimed = await db.oneOrNone(
      `INSERT INTO idempotency_keys
         (key, merchant_id, endpoint, request_hash, state, locked_until)
       VALUES ($1,$2,$3,$4,'in_progress', now() + interval '${LOCK_SECONDS} seconds')
       ON CONFLICT (merchant_id, endpoint, key) DO NOTHING
       RETURNING key`,
      [key, req.merchantId, endpoint, requestHash],
    );

    if (!claimed) {
      const existing = await db.one(
        `SELECT * FROM idempotency_keys WHERE merchant_id=$1 AND endpoint=$2 AND key=$3`,
        [req.merchantId, endpoint, key],
      );

      // Same key, different request → the client has a bug. Never guess.
      if (existing.request_hash !== requestHash) {
        throw new ApiError(422, "idempotency_key_reused",
          "This Idempotency-Key was already used with a different request body");
      }

      if (existing.state === "completed") {
        // ── THE REPLAY: return the stored response verbatim. ──
        return res.status(existing.response_status)
                  .set("Idempotent-Replayed", "true")
                  .json(existing.response_body);
      }

      // Still in progress: either genuinely concurrent, or a crashed attempt.
      if (existing.locked_until && existing.locked_until > new Date()) {
        throw new ApiError(409, "idempotency_conflict",
          "A request with this Idempotency-Key is currently in progress. Retry shortly.",
          { retry_after: 2 });
      }
      // Lock expired → take it over (the previous attempt died mid-flight).
      await db.query(
        `UPDATE idempotency_keys SET locked_until = now() + interval '${LOCK_SECONDS} seconds'
          WHERE merchant_id=$1 AND endpoint=$2 AND key=$3`,
        [req.merchantId, endpoint, key],
      );
    }

    // ── Capture the response so we can replay it later. ──
    const originalJson = res.json.bind(res);
    res.json = (body: unknown) => {
      // Only persist a definitive outcome. 5xx must remain retryable!
      if (res.statusCode < 500) {
        void db.query(
          `UPDATE idempotency_keys
              SET state='completed', response_status=$4, response_body=$5, locked_until=NULL
            WHERE merchant_id=$1 AND endpoint=$2 AND key=$3`,
          [key, req.merchantId, endpoint, res.statusCode, body],
        );
      } else {
        void db.query(
          `DELETE FROM idempotency_keys WHERE merchant_id=$1 AND endpoint=$2 AND key=$3`,
          [key, req.merchantId, endpoint],
        );                                    // release the key so a retry can proceed
      }
      return originalJson(body);
    };

    next();
  };
}

v1.post("/payments", idempotent("POST /payments", { required: true }), createPayment);
v1.post("/payments/:id/refunds", idempotent("POST /payments/:id/refunds", { required: true }), createRefund);
```

### The five decisions you must be able to defend

| Question | Answer, with reasoning |
|---|---|
| **Who generates the key?** | **The client**, as a UUIDv4, once per *logical operation* — and critically it must **not** change between retries. A server-generated key is useless: the retry wouldn't have it |
| **How long do you keep keys?** | **24 hours** is the industry norm (Stripe's). Long enough to cover any realistic retry; short enough to bound storage. Document it, because after expiry a "retry" becomes a new charge |
| **What about 5xx responses?** | **Release the key.** A 500 might mean nothing happened, so the client must be able to retry. Storing the 500 as the final answer makes a transient failure permanent |
| **What about 4xx?** | **Store them.** A validation error is deterministic — replaying it is correct and saves work |
| **Concurrent duplicates?** | `409` with `Retry-After`, driven by the atomic `INSERT ... ON CONFLICT`. Never a read-then-write check — that's the race you're fixing |

> **The most commonly missed detail, and a great thing to volunteer:** *"5xx must release the key. If you cache a 500 against the key, the client's retry gets the same 500 forever, and you've turned a blip into a permanent failure for that operation."*

### Idempotency you get for free (design-level)

Sometimes you don't need a key at all, and noticing that is the strongest answer:

| Technique | Example |
|---|---|
| **Natural unique constraint** | `UNIQUE (merchant_id, external_reference)` → the second insert fails with a conflict you translate to a replay |
| **Client-supplied ID with `PUT`** | `PUT /webhook-endpoints/we_abc` — idempotent by HTTP semantics ([Lesson 03](../01-foundations/03-http-methods-and-status.md)) |
| **State-guarded transitions** | `UPDATE ... WHERE status='requires_capture'` — the second call affects 0 rows ([Lesson 09](../02-rest-design/09-writes-patch-and-bulk.md)) |
| **Append-only ledgers** | Insert an entry with a deterministic ID; a duplicate violates the primary key |

---

## 6. Circuit breakers

Retries help with *blips*. They make **sustained** failures worse. A circuit breaker is the mechanism that stops retrying a dependency that is genuinely down.

```
CLOSED ──── failures exceed threshold ────▶ OPEN
   ▲                                          │
   │                                    after cooldown
   │                                          ▼
   └──── probe succeeds ──── HALF_OPEN ◀──────┘
                                │
                          probe fails → OPEN again
```

```ts
type State = "closed" | "open" | "half_open";

export class CircuitBreaker {
  private state: State = "closed";
  private failures = 0;
  private successes = 0;
  private openedAt = 0;

  constructor(private readonly o = {
    failureThreshold: 5,      // consecutive failures to open
    cooldownMs: 30_000,       // how long to stay open
    halfOpenProbes: 2,        // successes needed to close again
    name: "downstream",
  }) {}

  async call<T>(fn: () => Promise<T>, fallback?: () => Promise<T>): Promise<T> {
    if (this.state === "open") {
      if (Date.now() - this.openedAt < this.o.cooldownMs) {
        metrics.inc("circuit.rejected", { name: this.o.name });
        if (fallback) return fallback();
        throw new ApiError(503, "service_unavailable",
          `${this.o.name} is unavailable`, { retry_after: Math.ceil(this.o.cooldownMs / 1000) });
      }
      this.state = "half_open";                 // let one probe through
      this.successes = 0;
    }

    try {
      const result = await fn();
      this.onSuccess();
      return result;
    } catch (e) {
      this.onFailure();
      if (fallback && this.state === "open") return fallback();
      throw e;
    }
  }

  private onSuccess() {
    this.failures = 0;
    if (this.state === "half_open" && ++this.successes >= this.o.halfOpenProbes) {
      this.state = "closed";
      metrics.inc("circuit.closed", { name: this.o.name });
    }
  }
  private onFailure() {
    if (this.state === "half_open" || ++this.failures >= this.o.failureThreshold) {
      this.state = "open";
      this.openedAt = Date.now();
      metrics.inc("circuit.opened", { name: this.o.name });
    }
  }
}
```

**Why it's valuable, stated precisely:** without a breaker, every request to a dead dependency waits the full timeout (say 10s) before failing. At 100 req/s that's **1,000 concurrent requests** parked in your process, consuming memory and connections, all doomed. The breaker **fails fast** — 1ms instead of 10s — so your service stays responsive for everything *else* it does. That's the point: a circuit breaker protects **you** from your dependency, not the dependency from you.

Refinements worth naming: use a **failure *ratio* over a sliding window** rather than consecutive failures (a busy service always has some failures), don't count 4xx as circuit failures (those are the caller's fault), and add a small amount of jitter to the cooldown so all instances don't probe simultaneously.

### Bulkheads
Cap concurrency **per dependency**, so a slow one can't consume every worker:
```ts
const limits = {
  paymentProcessor: pLimit(20),    // max 20 concurrent
  emailService:     pLimit(5),
  reportGenerator:  pLimit(2),
};
await limits.paymentProcessor(() => breaker.call(() => processor.charge(...)));
```
The name comes from ship compartments: a breach floods one compartment, not the hull. Without it, a 30-second email provider consumes all your capacity and your payments endpoint goes down too — **failure in an unimportant dependency taking out an important one** is the exact thing bulkheads prevent.

### Load shedding
When you're saturated, **reject fast** rather than queueing forever:
```ts
const MAX_QUEUE = 100;
app.use((req, res, next) => {
  if (inFlight > MAX_QUEUE) {
    return res.status(503)
      .set("Retry-After", "2")
      .json(problem("service_unavailable", "Server is at capacity. Retry shortly."));
  }
  inFlight++; res.on("finish", () => inFlight--);
  next();
});
```
A queue that grows without bound converts a capacity problem into a **total** outage: every request times out, every timeout triggers a retry, and the retries add load. Shedding early keeps you *partially* available — which is strictly better. Prioritise if you can: shed health-check-adjacent and low-value traffic before payments.

---

## 7. The dual-write problem and the outbox pattern

The reliability problem inside your own handler:

```ts
// ❌ Not atomic. Two systems, one of which will fail.
await db.payments.insert(payment);      // succeeds
await queue.publish("payment.created"); // ← process crashes here
// Result: the payment exists, but no webhook is ever sent. Silently.
```

You cannot transactionally write to a database *and* a message broker. The fix is the **transactional outbox**:

```sql
BEGIN;
  INSERT INTO payments (...) VALUES (...);
  INSERT INTO outbox (id, event_type, payload, created_at)
       VALUES (gen_random_uuid(), 'payment.created', $1, now());
COMMIT;                        -- both, or neither. One transaction, one database.
```
A separate worker polls `outbox` (or tails the WAL via CDC/Debezium), publishes, and marks rows sent. If publishing fails, it retries. **This gives at-least-once delivery** — so consumers must be idempotent, which brings you right back to §5. That closed loop is the whole architecture of a reliable event system, and being able to draw it is a strong systems answer.

---

## 8. Production rules

| Rule | Why |
|---|---|
| **Every outbound call has connect, idle and total timeouts** | An unbounded call holds resources forever |
| **Timeouts decrease downstream; propagate a deadline** | No work for requests nobody awaits |
| **LB idle timeout < app keep-alive timeout** | Otherwise intermittent, unexplainable 502s |
| **Retry only idempotent operations, and only on transient failures** | Retrying a `POST` without a key duplicates side effects |
| **Exponential backoff with *full* jitter** | Prevents synchronised retry waves |
| **Retry at one layer only; keep a retry budget (~10%)** | Retry amplification is 27× three layers deep |
| **Require `Idempotency-Key` on every money-moving `POST`** | The only way a client can safely retry |
| **Scope idempotency keys by tenant + endpoint** | Otherwise a shared key leaks across tenants |
| **Hash the request body against the key** | Prevents "same key, different request" returning the wrong answer |
| **Release the key on 5xx; store it on 4xx** | Keeps transient failures retryable, deterministic ones cheap |
| **Claim the key with `INSERT ... ON CONFLICT`, never read-then-write** | The check must be atomic |
| **Circuit breaker + bulkhead per dependency** | Fail fast; contain the blast radius |
| **Shed load with `503` + `Retry-After` when saturated** | Partial availability beats total collapse |
| **Outbox for anything that must happen after a commit** | You cannot atomically write to a DB and a queue |
| **Assume at-least-once; make consumers idempotent** | Exactly-once doesn't exist |

> **Spring equivalent:** Resilience4j (`@Retry`, `@CircuitBreaker`, `@Bulkhead`, `@TimeLimiter`) and `spring-retry`. Same concepts, annotation-driven — see `spring boot/09-production/34-resilience-and-integration.md`.

---

## 9. Interview traps

**Q1. "A client `POST`s a payment, times out, and retries. How do you prevent a double charge?"**
**The** question. Structure:
1. Name the ambiguity — the client cannot know whether it succeeded, so it *must* retry.
2. `Idempotency-Key` header, client-generated UUID, stable across retries.
3. The storage: key + tenant + endpoint + request hash + state + stored response, with an atomic claim via `INSERT ... ON CONFLICT`.
4. The four outcomes: first → execute; replay with same body → stored response; same key different body → 422; in flight → 409.
5. The details that prove you've built it: 24h TTL, **5xx releases the key**, tenant scoping, and canonicalised body hashing.

**Q2. "Why can't you just make `POST` idempotent automatically?"**
Because the server can't tell a retry from a genuine second request — two identical "charge ₹4999" calls might be one retry or two real purchases. **Only the client knows**, which is why the key must come from the client.

**Q3. "What's wrong with retrying without jitter?"**
Synchronisation. All failing clients wake simultaneously and arrive in coordinated waves that re-kill the recovering service. Full jitter (`random(0, cap)`) spreads them uniformly.

**Q4. "Explain retry amplification."**
Nested retries multiply: 3 layers × 3 attempts = 27 requests at the bottom, arriving exactly when the bottom is already failing. Fixes: retry at one layer, retry budgets, and circuit breakers.

**Q5. "What does a circuit breaker actually buy you?"**
Failing *fast*. Without it, every doomed request waits the full timeout, so at 100 req/s with a 10s timeout you have 1,000 requests parked in your process. The breaker protects **you** from your dependency. Mention half-open probing and using a failure ratio rather than a raw count.

**Q6. "Does your message queue guarantee exactly-once delivery?"**
No queue does, over a network. **At-least-once delivery + idempotent consumers = effectively once.** (Kafka's "exactly-once semantics" is real but narrower than the name: it covers atomic read-process-write *within* Kafka via transactions and idempotent producers — it does not make an external side effect like charging a card exactly-once.) Being precise here is a strong signal.

**Q7. "You write to the DB and then publish an event, and the process crashes in between."**
The dual-write problem. Transactional outbox: write the event into the same DB transaction, then a worker publishes and marks it sent. At-least-once, so consumers must be idempotent.

**Q8. "Your payment processor starts taking 30 seconds. Describe everything that fails, in order."**
Excellent question — answer it as a cascade: (1) requests pile up waiting; (2) your connection/worker pool saturates; (3) *unrelated* endpoints start timing out because there's no capacity left; (4) clients time out and retry, doubling load; (5) health checks fail and the orchestrator restarts your pods, dropping in-flight work; (6) the restarts empty your caches, so recovery is slower still. Then the fixes, mapped: timeouts, bulkhead, circuit breaker, load shedding, and **health checks that don't depend on the downstream** ([Lesson 20](20-observability.md)).

**Q9. "How long do you keep idempotency keys, and what happens after that?"**
24 hours (the industry norm). After expiry the key is forgotten, so a "retry" is treated as a new request — which is why the window must exceed any realistic client retry horizon, and why it must be documented.

**Q10. "Client sends the same key with a different body. What do you return?"**
`422 idempotency_key_reused`. **Never** guess: returning the stored response would tell them "your ₹100 charge succeeded" when they asked for ₹5,000, and executing the new body would break the guarantee. Reject it as a client bug.

---

## 10. Build & break

### Build — the idempotency middleware
Implement §5 fully for Ledger, then write the tests. **These tests are the deliverable:**

```ts
test("first request executes and stores the result", ...);
test("replay with the same key and body returns the stored response byte-for-byte", ...);
test("replay does not create a second payment", ...);           // count rows!
test("same key, different body → 422 idempotency_key_reused", ...);
test("concurrent identical requests → one 201 and one 409", async () => {
  const [a, b] = await Promise.all([post(body, key), post(body, key)]);
  const codes = [a.status, b.status].sort();
  expect(codes).toEqual([201, 409]);
  expect(await countPayments()).toBe(1);                         // ← the real assertion
});
test("a 500 releases the key so a retry can succeed", ...);
test("a 422 is stored and replayed", ...);
test("keys are scoped per merchant: same key, two tenants → two payments", ...);
test("body hash is canonical: key order does not change the hash", ...);
test("an expired in_progress lock can be taken over", ...);
```

### Build — a resilient outbound client
Compose everything: timeout → bulkhead → circuit breaker → retry with jitter → budget.
```ts
const breaker = new CircuitBreaker({ failureThreshold: 5, cooldownMs: 30_000, halfOpenProbes: 2, name: "processor" });
const limit = pLimit(20);
const budget = new RetryBudget(0.1, 10_000);

export async function charge(req: ChargeRequest, deadline: Deadline) {
  return limit(() =>
    withRetry(
      attempt => {
        budget.record(attempt > 1);
        return breaker.call(() => callDownstream("/charge", deadline));
      },
      e => isTransient(e) && budget.allow(),
      { maxAttempts: 3, baseMs: 200, maxMs: 4_000 },
    ),
  );
}
```
Note the **ordering**: the breaker is *inside* the retry (so each attempt consults it), and the bulkhead is *outside* (so retries don't each claim a new slot). Getting that order right is a real design detail — reversing it means retries bypass the concurrency cap.

### Break — five failures to induce
1. **The double charge.** Remove idempotency, add a 100ms artificial delay, then fire the same request twice with `Promise.all`. Count rows: 2. Add the key. Count rows: 1.
2. **The retry storm.** 200 clients retrying without jitter against a service that returns 503 for 5 seconds. Graph requests/second — you'll see distinct spikes. Add full jitter and watch it flatten into noise.
3. **The saturation cascade.** Point a dependency at `https://httpstat.us/200?sleep=30000` with no timeout and no bulkhead. Load your *unrelated* health endpoint and watch it die too. Add a 3s timeout, then a bulkhead, then a breaker, measuring recovery at each step.
4. **Cached 500.** Store 5xx responses against the idempotency key, then retry. Watch the operation become permanently impossible. That's why 5xx releases the key.
5. **The dual write.** Insert a row, then `process.exit(1)` before publishing. Confirm the payment exists with no event. Implement the outbox and repeat — the event is now recovered by the worker.

### Explain out loud (2.5 minutes)
1. The fundamental ambiguity, and why exactly-once doesn't exist.
2. The full idempotency-key design, including the four outcomes and the 5xx rule.
3. Backoff with full jitter, and retry amplification.
4. What a circuit breaker buys, and who it protects.
5. The dual-write problem and the outbox.

---

## What's next

You can survive failure. Next: surviving *success* — one client sending 10,000 requests a second. Rate limiting algorithms, the correct 429 contract, and how to be fair to tenants who share your infrastructure.

Next → **[Lesson 19: Rate limiting, quotas & fairness](19-rate-limiting.md)**
