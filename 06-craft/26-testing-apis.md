# Lesson 26 — Testing APIs properly

> **Why this lesson exists:** most API test suites test the happy path exhaustively and the failure paths not at all — which is exactly backwards, because the happy path is what you'd notice manually and the failure paths are what break in production at 3am. This lesson is about testing the things that actually go wrong: concurrency, retries, authorization, boundaries and contracts. It's also a reliable interview differentiator, because *how* someone tests reveals what they've been burned by.

**Time:** ~85 minutes · **Prereq:** Modules 2–4, Lesson 25

---

## 1. The idea in one sentence

> **Test the properties that must hold, not the code paths you happened to write — because a test suite that asserts "this returns 200" tells you nothing about whether money can be charged twice.**

---

## 2. The API testing pyramid

```
        ╱ E2E ╲                few    real deployed stack, real browser
      ╱─────────╲
    ╱  Contract   ╲            some   does my API match the spec / consumers' expectations?
  ╱─────────────────╲
╱    Integration      ╲        many   ★ THE MOST VALUABLE LAYER FOR AN API
╲─────────────────────╱               real DB, real HTTP, mocked third parties
  ╲     Unit        ╱          many   pure logic: money math, state machines, cursors
    ╲─────────────╱
```

**The API-specific correction to the classic pyramid:** for an API, the **integration layer is where the value is**, not the unit layer. The reason is that most API bugs live in the *seams* — routing, middleware order, serialization, transactions, the SQL you actually run, authorization checks. A unit test with a mocked repository proves your handler calls the repository; it proves nothing about whether the query is tenant-scoped, whether the transaction rolls back, or whether the middleware ran in the right order.

> **Say this in an interview:** *"For an API I invert the usual advice slightly — I want a lot of integration tests against a real database via real HTTP, because the bugs that reach production are in the seams: middleware ordering, tenancy scoping, transaction boundaries, serialization. Unit tests are for the pure logic — money arithmetic, state machines, cursor encoding — where they're fast and precise."*

### What belongs at each layer

| Layer | Test | Don't test |
|---|---|---|
| **Unit** | Money math, cursor encode/decode, HMAC signing, state-machine transitions, backoff calculation, policy functions | Anything needing a DB or HTTP |
| **Integration** | Every endpoint end-to-end: status codes, headers, body shape, DB effects, authorization, idempotency, concurrency, transactions | Third-party APIs (mock them) |
| **Contract** | Response conformance to OpenAPI; consumer expectations (Pact) | Business logic |
| **E2E** | A few critical journeys against a deployed environment | Everything else — it's slow and flaky |
| **Load** | Throughput, p99 under load, behaviour at saturation | Correctness |

---

## 3. Integration tests: the setup that makes them worth having

**Use a real database.** An in-memory fake or SQLite substitute means you never test the SQL you actually run — no row-level security, no real transaction semantics, no `ON CONFLICT`, no index behaviour, no Postgres-specific types.

```ts
// test/setup.ts — Testcontainers: a real Postgres, per test run
import { PostgreSqlContainer, StartedPostgreSqlContainer } from "@testcontainers/postgresql";

let container: StartedPostgreSqlContainer;

beforeAll(async () => {
  container = await new PostgreSqlContainer("postgres:16-alpine").start();
  process.env.DATABASE_URL = container.getConnectionUri();
  await runMigrations();                 // ← run your REAL migrations, not a schema dump
}, 60_000);

afterAll(() => container?.stop());
```

**Run your real migrations** rather than loading a schema snapshot: it tests the migrations themselves, which are the riskiest code you deploy.

### Isolation between tests — pick one strategy deliberately

| Strategy | Speed | Isolation | Note |
|---|---|---|---|
| **Truncate all tables** between tests | Fast | Good | Simple and predictable. **My default** |
| **Transaction + rollback** per test | Fastest | Good | Breaks if the code under test manages its own transactions — which yours does |
| **Unique tenant per test** | Fast | **Excellent** | No cleanup needed, and it *tests your tenancy scoping as a side effect* |
| Fresh database per test | Slow | Perfect | Only for migration tests |

**The unique-tenant strategy deserves attention** because it does double duty: every test creates its own merchant, so tests can run in parallel *and* any test that accidentally reads across tenants fails. That's a nice property to get for free.

```ts
// test/fixtures.ts — a fluent builder beats fixture files
export async function aMerchant(overrides: Partial<Merchant> = {}) {
  const m = await db.merchants.insert({ id: newId("mrc"), name: "Test Co", ...overrides });
  return {
    ...m,
    apiKey: (await createApiKey(m.id, ["payments:read", "payments:write"])).plaintext,
    async withPayment(o: Partial<Payment> = {}) {
      return db.payments.insert({
        id: newId("pi"), merchant_id: m.id,
        amount_minor: 4999, currency: "usd", status: "succeeded", ...o,
      });
    },
  };
}

// Usage reads like the scenario it describes:
const merchant = await aMerchant();
const payment = await merchant.withPayment({ status: "requires_capture" });
```
Builders beat static fixture files because a test states **only what it cares about** — so when you add a required column, you change one builder rather than 200 fixtures.

### Mocking third parties — at the HTTP boundary, not the module boundary

```ts
import nock from "nock";

test("returns 502 when the processor is unavailable", async () => {
  nock("https://api.processor.test").post("/charges").reply(503, { error: "unavailable" });

  const res = await api(merchant.apiKey).post("/v1/payments", validBody, { idempotencyKey: key });

  expect(res.status).toBe(502);
  expect(res.body.code).toBe("upstream_error");
  // ★ The assertion that matters: no phantom payment was recorded.
  expect(await countPayments(merchant.id)).toBe(0);
});

afterEach(() => { nock.cleanAll(); });
beforeAll(() => { nock.disableNetConnect(); nock.enableNetConnect("127.0.0.1"); });
```

Two rules:
1. **Mock at the HTTP layer** (nock/MSW/WireMock), not by stubbing your own client module. Mocking HTTP tests your client's timeout, retry, error-mapping and parsing code; stubbing the module skips all of it — which is precisely where the bugs are.
2. **`disableNetConnect()`** so an un-mocked call fails loudly instead of silently hitting the internet (or a real third party) from CI.

---

## 4. What to actually test — the properties that matter

This is the heart of the lesson. Every item below has caused a real production incident.

### The complete per-endpoint checklist
```
For every endpoint:
[ ] Happy path: correct status, headers (Location/ETag/X-Request-Id), body shape
[ ] Every documented error status, with the right `code`
[ ] Unauthenticated             → 401 + WWW-Authenticate
[ ] Wrong role                  → 403
[ ] Another tenant's object     → 404 (never 403, never 200)
[ ] Unknown body field          → 422 unknown_field
[ ] Unknown query param         → 400 unknown_filter
[ ] Missing required field       → 422 with the field named
[ ] Wrong Content-Type          → 415
[ ] Oversized body              → 413
[ ] Boundary values             → min, min-1, max, max+1, 0, negative, empty string, null
[ ] Unicode / emoji / very long strings in text fields
[ ] The response validates against the OpenAPI spec
[ ] No credentials or stack traces anywhere in the response or logs
```

### The four property tests that catch real bugs

**1. Idempotency**
```ts
test("a retried create does not double-charge", async () => {
  const key = crypto.randomUUID();
  const a = await api(k).post("/v1/payments", body, { idempotencyKey: key });
  const b = await api(k).post("/v1/payments", body, { idempotencyKey: key });

  expect(a.status).toBe(201);
  expect(b.status).toBe(200);
  expect(b.body).toEqual(a.body);                     // byte-identical replay
  expect(await countPayments(merchant.id)).toBe(1);   // ← THE assertion
});

test("concurrent identical requests create exactly one payment", async () => {
  const key = crypto.randomUUID();
  const results = await Promise.all(
    Array.from({ length: 10 }, () => api(k).post("/v1/payments", body, { idempotencyKey: key })));

  const created = results.filter(r => r.status === 201);
  expect(created).toHaveLength(1);                    // exactly one winner
  expect(await countPayments(merchant.id)).toBe(1);
});
```
Run the concurrent one **20 times in CI** (`test.each(Array(20))`) — race conditions are probabilistic, and a single pass proves nothing.

**2. Concurrency / lost updates**
```ts
test("concurrent patches with the same ETag: one wins, one gets 412", async () => {
  const { headers } = await api(k).get(`/v1/customers/${id}`);
  const etag = headers.etag;

  const [a, b] = await Promise.all([
    api(k).patch(`/v1/customers/${id}`, { name: "A" }, { ifMatch: etag }),
    api(k).patch(`/v1/customers/${id}`, { name: "B" }, { ifMatch: etag }),
  ]);

  expect([a.status, b.status].sort()).toEqual([200, 412]);
});

test("double capture is impossible", async () => {
  const p = await merchant.withPayment({ status: "requires_capture" });
  const [a, b] = await Promise.all([capture(p.id), capture(p.id)]);
  expect([a.status, b.status].sort()).toEqual([200, 409]);
  expect((await getPayment(p.id)).captured_count).toBe(1);
});
```

**3. Authorization matrix** — the actor × endpoint table from [Lesson 15 §7](../03-security/15-authorization-and-multitenancy.md), **plus the meta-test that every registered route appears in it.** That meta-test is the highest-value single test in an API codebase, because it makes a security omission a build failure rather than a discovery.

**4. Transaction integrity**
```ts
test("a failed webhook enqueue does not leave a phantom payment", async () => {
  jest.spyOn(outbox, "insert").mockRejectedValueOnce(new Error("boom"));
  const res = await api(k).post("/v1/payments", body, { idempotencyKey: key });
  expect(res.status).toBe(500);
  expect(await countPayments(merchant.id)).toBe(0);    // rolled back together
  expect(await countOutbox()).toBe(0);
});
```
This is the test that proves your outbox is genuinely in the same transaction ([Lesson 18](../04-production/18-reliability-and-idempotency.md)) rather than merely adjacent to it.

### Boundary testing, systematically
```ts
describe.each([
  ["below minimum",   { amount_minor: 49 },              422, "too_small"],
  ["at minimum",      { amount_minor: 50 },              201, null],
  ["zero",            { amount_minor: 0 },               422, "too_small"],
  ["negative",        { amount_minor: -100 },            422, "too_small"],
  ["float",           { amount_minor: 49.99 },           422, "invalid_type"],
  ["string",          { amount_minor: "4999" },          422, "invalid_type"],
  ["above int64",     { amount_minor: 9e18 },            422, "too_large"],
  ["unsafe integer",  { amount_minor: 9007199254740993 },422, "too_large"],
  ["unknown currency",{ currency: "xyz" },               422, "unsupported_value"],
  ["emoji metadata",  { metadata: { note: "💸🎉" } },     201, null],
  ["long metadata",   { metadata: { note: "x".repeat(501) } }, 422, "too_long"],
  ["null description",{ description: null },             422, "invalid_type"],
])("amount %s", (_n, patch, status, code) => { /* ... */ });
```
**The `9007199254740993` case is the Lesson 05 precision bug, as a test.** Worth having permanently.

---

## 5. Contract testing — two different things

### Provider contract testing: does my API match my spec?
```ts
// Cheapest version: validate every test response against OpenAPI (Lesson 25)
import jestOpenAPI from "jest-openapi";
jestOpenAPI("./openapi.yaml");

test("GET /payments/:id matches the spec", async () => {
  const res = await api(k).get(`/v1/payments/${p.id}`);
  expect(res).toSatisfyApiSpec();          // ← one line, catches all drift
});
```

**Property-based / fuzz testing from the spec** — genuinely high-value and rarely done:
```bash
# Schemathesis generates thousands of requests from your OpenAPI schema
# and asserts that responses conform and nothing 500s.
schemathesis run openapi.yaml \
  --base-url http://localhost:3000/v1 \
  --header "Authorization: Bearer $TEST_KEY" \
  --checks all --hypothesis-max-examples 500
```
It finds the inputs you'd never think of: empty arrays, deeply nested objects, unicode edge cases, integer overflow, null in unexpected places. **Any 500 it finds is a real bug**, because a 500 means unhandled input. This is the single highest return-on-effort testing tool for an API, and mentioning it is a differentiator.

### Consumer-driven contract testing: does my API match what *clients expect*?
```ts
// Pact — the consumer (Ledger Console) declares its expectations
await provider.addInteraction({
  state: "a succeeded payment pi_1 exists for merchant mrc_1",
  uponReceiving: "a request for payment pi_1",
  withRequest: { method: "GET", path: "/v1/payments/pi_1" },
  willRespondWith: {
    status: 200,
    body: like({ id: "pi_1", amount_minor: 4999, status: "succeeded" }),
  },
});
```
The consumer publishes a pact; the **provider's** CI verifies it can satisfy every consumer's pact. So *"which of my clients does this change break?"* is answered by a build, not a support ticket.

**When Pact is worth it:** several internal consumers you can coordinate with. **When it isn't:** a public API with thousands of unknown integrators — you can't collect pacts from strangers, so OpenAPI + `oasdiff breaking` + usage instrumentation is the right tooling there instead. Knowing which situation calls for which is the real answer.

---

## 6. Load and resilience testing

```js
// k6 — load test with SLO-shaped thresholds
import http from "k6/http";
import { check } from "k6";

export const options = {
  stages: [
    { duration: "1m", target: 50 },    // ramp
    { duration: "3m", target: 50 },    // steady — measure HERE
    { duration: "1m", target: 300 },   // spike — find the knee
    { duration: "2m", target: 0 },     // recovery — does it come back?
  ],
  thresholds: {
    "http_req_duration{endpoint:get_payment}": ["p(95)<200", "p(99)<500"],
    "http_req_failed": ["rate<0.01"],
    "checks": ["rate>0.99"],
  },
};

export default function () {
  const res = http.get(`${__ENV.BASE}/v1/payments/${__ENV.PID}`, {
    headers: { Authorization: `Bearer ${__ENV.KEY}` },
    tags: { endpoint: "get_payment" },
  });
  check(res, { "200": r => r.status === 200, "has request id": r => !!r.headers["X-Request-Id"] });
}
```

**What you're actually looking for** — not "can it handle 1000 rps", but:

| Question | What you learn |
|---|---|
| Where's the **knee**? | The load at which p99 starts climbing non-linearly — your real capacity |
| What breaks **first**? | DB connections, event loop, memory, a downstream. That's your bottleneck |
| Does it **degrade or collapse**? | Graceful (429/503 with `Retry-After`) or cascading failure ([Lesson 18](../04-production/18-reliability-and-idempotency.md)) |
| Does it **recover**? | After the spike, does latency return to baseline, or stay wedged? |
| Do the **limits work**? | Do rate limits engage before the database falls over? |

That last one is the point most people miss: **a load test that doesn't verify your rate limiter engages before your database saturates hasn't tested your protection at all.**

### Resilience testing — inject the failures from Lesson 18
```ts
describe("resilience", () => {
  test("processor timeout → 502, and no phantom payment", ...);
  test("processor 503 → circuit opens after 5 failures", ...);
  test("circuit open → fails fast (<50ms), not after the full timeout", ...);
  test("Redis down → rate limiter fails open on /payments", ...);
  test("Redis down → auth endpoints fail closed", ...);
  test("DB connection pool exhausted → 503, not a hang", ...);
  test("readiness fails while liveness stays healthy when the DB is down", ...);
});
```
Use **toxiproxy** to inject latency, bandwidth limits and connection resets at the network layer, so you test real socket behaviour rather than a mocked rejection.

---

## 7. CI strategy and flakiness

```yaml
# Fast feedback first, expensive checks later.
on: [pull_request]
jobs:
  fast:         # < 2 min — must pass before anything else runs
    - typecheck
    - lint
    - spectral lint openapi.yaml
    - oasdiff breaking base/openapi.yaml openapi.yaml     # ← the compatibility gate
    - unit tests
  integration:  # < 10 min
    - testcontainers integration suite (parallel, unique tenant per test)
    - authorization matrix + route-coverage meta-test
    - response validation against OpenAPI
  nightly:
    - schemathesis --hypothesis-max-examples 2000
    - k6 load test against staging
    - pact provider verification
    - dependency + secret scanning
```

### Flakiness: treat it as a bug, not weather
A flaky test is worse than no test, because it trains the team to re-run CI instead of reading failures.

| Cause | Fix |
|---|---|
| Shared state between tests | Unique tenant per test; truncate between |
| Time dependence | Inject a clock; never `sleep` — poll for a condition with a timeout |
| Order dependence | Randomise test order in CI so you *find* it |
| Real network calls | `nock.disableNetConnect()` |
| Race in the code under test | **This is a real bug.** The flake is the feature — do not retry it away |
| `Date.now()` / timezone | Freeze time; run CI in a non-UTC timezone to catch assumptions |

**Never add a blanket `retry: 3` to your test runner.** It hides genuine race conditions — which are exactly the bugs your concurrency tests exist to find.

---

## 8. Production rules

| Rule | Why |
|---|---|
| **Integration tests against a real database, via real HTTP** | API bugs live in the seams, not the functions |
| **Run your real migrations in test setup** | Migrations are the riskiest code you ship |
| **Unique tenant per test** | Parallel-safe, no cleanup, and it tests tenancy as a side effect |
| **Mock third parties at the HTTP boundary; `disableNetConnect`** | Tests your client's timeouts, retries and parsing |
| **Test every documented error path, and assert the `code`** | Error handling is the most-read, least-tested code |
| **Idempotency and concurrency tests, repeated 20× in CI** | Races are probabilistic; one pass proves nothing |
| **Assert *effects*, not just statuses** (`countPayments() === 1`) | A 200 tells you nothing about how many rows exist |
| **Authorization matrix + route-coverage meta-test** | Makes a missing authz check a build failure |
| **Validate every test response against OpenAPI** | One line; catches all drift |
| **Property-based fuzzing from the spec (Schemathesis) nightly** | Finds the inputs you'd never write; every 500 is a real bug |
| **Assert that logs and responses contain no credentials** | Redaction regresses silently |
| **Load test for the knee and for recovery, not a target number** | Capacity is a curve, not a figure |
| **Fix flakes; never retry them away** | A flake is often a real race |
| **`oasdiff breaking` as a required PR check** | Compatibility as a build gate |

> **Spring equivalent:** `@SpringBootTest` + `MockMvc`/`WebTestClient` + Testcontainers + WireMock + Pact-JVM, with `@Sql` or per-test tenants for isolation. Identical philosophy — see `spring boot/08-testing/30-spring-integration-testing.md`.

---

## 9. Interview traps

**Q1. "How do you test an API?"**
Lead with the layer inversion: *"heavy on integration tests against a real DB via real HTTP, because API bugs are in the seams — middleware order, tenancy scoping, transactions, serialization. Unit tests for pure logic. Contract tests against OpenAPI so docs can't drift. A few E2E journeys. Load tests for the knee, not a number."* Then name the four property tests: idempotency, concurrency, authorization matrix, transaction integrity.

**Q2. "What do most API test suites miss?"**
Concrete list: concurrency (two simultaneous writes), idempotency (the retry), cross-tenant authorization (the *negative* test), boundary values, the documented error paths, and behaviour under downstream failure. *"Teams test that Alice can read her payment; almost nobody tests that Bob can't."*

**Q3. "How do you test a race condition?"**
`Promise.all` of N identical requests, then assert on the **effect** (exactly one row created; statuses `[201, 409]`), and run it repeatedly in CI. Add artificial latency at the critical point to widen the window and make the race reliably reproducible.

**Q4. "Real database or mocks?"**
Real, via Testcontainers. A mocked repository proves your handler calls a function; it can't prove the query is tenant-scoped, the transaction rolls back, `ON CONFLICT` behaves, or RLS applies. Mock only what you don't own — third-party HTTP.

**Q5. "What's consumer-driven contract testing and when do you use it?"**
Consumers publish expectations (pacts); the provider's CI verifies it satisfies all of them, so you learn which client a change breaks at build time. Worth it with several coordinatable internal consumers; not applicable to a public API with unknown integrators — there you use OpenAPI + breaking-change detection + per-client usage instrumentation.

**Q6. "How do you load test properly?"**
Ramp, steady, spike, recover — and look for the knee, the first component to break, whether degradation is graceful, whether it recovers, and **whether your rate limiter engages before the database saturates.** A single "we hit 1,000 rps" number is not a load test.

**Q7. "A test is flaky. What do you do?"**
Investigate, never retry. Then the diagnosis order: shared state, time dependence, order dependence, real network, or a genuine race in the code. Emphasise the last one — *"a flaky concurrency test is usually telling the truth."*

**Q8. "How do you test that you didn't leak a secret?"**
An explicit assertion: capture logs and responses during a request made with a known sentinel credential, then assert the sentinel doesn't appear anywhere. Plus a regex sweep for stack-trace and SQL signatures in error bodies ([Lesson 10](../02-rest-design/10-errors-and-problem-details.md)).

**Q9. "How much coverage do you need?"**
Reframe it, because the number is the wrong metric: *"I care about which properties are covered, not the percentage. 100% line coverage with no concurrency, authorization or idempotency tests is a worse suite than 60% with all three. The coverage metric I'd actually enforce is route coverage in the authorization matrix — every endpoint must have an explicit authorization test or CI fails."*

---

## 10. Build & break

### Build — the suite for Ledger
1. **Testcontainers setup** with real migrations and a unique merchant per test.
2. **Per-endpoint integration tests** using the §4 checklist.
3. **The four property suites:** idempotency (including concurrent, ×20), concurrency (`If-Match` and double-capture), the authorization matrix with the route-coverage meta-test, and transaction integrity.
4. **`toSatisfyApiSpec()`** on every response.
5. **Resilience suite** with nock/toxiproxy for the §6 list.
6. **A k6 script** with SLO thresholds, run against staging nightly.
7. **Schemathesis** nightly with 2,000 examples.
8. **The no-secrets-in-logs test.**

### Break — five experiments where the test finds the bug
1. **Remove the atomic idempotency claim** (use read-then-write). Run the concurrent test 20×. Watch it fail intermittently — and note that a single run passed. That's why you repeat.
2. **Remove `AND merchant_id = $2`** from one repository method. Watch the authorization matrix catch it immediately.
3. **Add a response field not in the spec.** Watch `toSatisfyApiSpec()` fail. This is drift, caught.
4. **Run Schemathesis against an endpoint with a hand-rolled validator.** Count the 500s it finds. Every one is a real unhandled input.
5. **Delete the outbox from the payment transaction.** Watch the transaction-integrity test find the phantom payment.

Then do the reverse — the exercise that actually teaches you: **delete a test and try to introduce the bug it guarded.** If you can't tell which test would have caught a given bug, that's a gap.

### Explain out loud (2 minutes)
1. The API pyramid, and why integration is the valuable layer here.
2. The four property tests, and the assertion that matters in each.
3. Provider vs consumer-driven contract testing, and when each applies.
4. What you look for in a load test.
5. Why you never retry a flaky test.

---

## What's next

You can design, secure, operate, specify and test an API. The last craft lesson turns all of it into a **repeatable 15-minute review** — the checklist you run against your own work and anyone else's, which is also the fastest way to revise the entire track.

Next → **[Lesson 27: The complete API checklist & code review guide](27-api-review-checklist.md)**
