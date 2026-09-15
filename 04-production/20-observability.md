# Lesson 20 — Observability & debugging APIs

> **Why this lesson exists:** everything before this lesson is about building a good API. This one is about being able to **fix** it. The difference between a 5-minute incident and a 5-hour one is almost never talent — it's whether the system was built to answer questions. Interviewers probe this with *"your p99 spiked, what do you do?"*, and the answer they're listening for is a **method**, not a guess.

**Time:** ~85 minutes · **Prereq:** Module 4 so far

---

## 1. The idea in one sentence

> **Monitoring tells you *that* something is wrong; observability lets you ask *why* — including questions you didn't anticipate when you wrote the code.**

That distinction is the whole lesson. A dashboard of pre-built charts is monitoring. The ability to ask *"show me p99 latency for `POST /payments`, for merchant mrc_9s2k, on API version 2019-08-14, where the payment processor was `stripe_uk`, in the last 20 minutes"* — without deploying anything — is observability.

---

## 2. The three pillars, and what each is actually for

| Pillar | Answers | Cardinality | Cost |
|---|---|---|---|
| **Metrics** | *"Is something wrong, and how wrong?"* | **Low** — must be bounded | Cheap |
| **Logs** | *"What exactly happened in this one request?"* | Unbounded | Expensive at volume |
| **Traces** | *"Where did the time go, across services?"* | Sampled | Moderate |

The mistake to avoid is using one for another's job:
- **Metrics for debugging individual requests** → you add `payment_id` as a label, cardinality explodes, and your metrics bill or your Prometheus dies.
- **Logs for aggregate questions** → "what's my p99?" becomes a log query that scans terabytes.
- **Traces for everything** → you sample at 100%, and the cost is untenable.

Use each for its job. **And the thing that ties all three together is one ID.**

---

## 3. Correlation: the single most valuable thing you can add

If you do nothing else from this lesson, do this.

```ts
import { AsyncLocalStorage } from "node:async_hooks";
import { randomUUID } from "node:crypto";

type Ctx = {
  requestId: string;
  traceId: string;
  merchantId?: string;
  principalId?: string;
  route?: string;
};

export const ctx = new AsyncLocalStorage<Ctx>();
export const current = () => ctx.getStore();

export function correlation(req: Request, res: Response, next: NextFunction) {
  // Accept an inbound id from the gateway, but validate its shape — never trust it raw.
  const inbound = req.header("x-request-id");
  const requestId = inbound && /^[\w-]{8,64}$/.test(inbound) ? inbound : randomUUID();

  // W3C traceparent: 00-<32 hex traceId>-<16 hex spanId>-<flags>
  const traceparent = req.header("traceparent");
  const traceId = traceparent?.split("-")[1] ?? requestId.replaceAll("-", "").padEnd(32, "0").slice(0, 32);

  res.setHeader("X-Request-Id", requestId);            // on EVERY response, errors included

  ctx.run({ requestId, traceId }, () => next());
}

/** Enrich the context once identity is known. */
export function enrichContext(req: Request, _res: Response, next: NextFunction) {
  const store = current();
  if (store) {
    store.merchantId = req.principal?.merchantId;
    store.principalId = req.principal?.kind === "user" ? req.principal.userId : req.principal?.keyId;
    store.route = `${req.method} ${req.route?.path ?? req.path}`;
  }
  next();
}
```

`AsyncLocalStorage` is the key mechanism: it gives you implicit per-request context **without threading a `ctx` parameter through every function**. Every log line, metric and span in that request automatically carries the same IDs.

**The payoff is the support workflow.** A customer says *"my request failed at 14:03."* You ask for the `X-Request-Id` from the response — which you returned because of the line above — and one query gives you every log line, the full trace, and the exact error. **Without it, that ticket is archaeology.**

> Propagate `traceparent` and `x-request-id` on every *outbound* call too, or the chain breaks at your service boundary and you can't follow a request through the system.

---

## 4. Structured logging

```ts
// ❌ Unparseable at scale. You cannot query this.
console.log(`Payment ${id} failed for merchant ${m}: ${err.message}`);

// ✅ Queryable, aggregatable, alertable
logger.error({
  event: "payment_failed",
  payment_id: id,
  merchant_id: m,
  amount_minor: 4999,
  currency: "usd",
  processor: "acme_uk",
  error_code: "card_declined",
  decline_reason: "insufficient_funds",
  duration_ms: 342,
  attempt: 2,
}, "payment failed");
```

The second version lets you ask *"decline rate by processor, by hour, for merchants on the business tier"* — a question nobody anticipated. The first version lets you `grep`.

```ts
import pino from "pino";

export const logger = pino({
  level: process.env.LOG_LEVEL ?? "info",
  // Every line automatically carries the correlation IDs.
  mixin: () => {
    const c = current();
    return c ? { request_id: c.requestId, trace_id: c.traceId,
                 merchant_id: c.merchantId, route: c.route } : {};
  },
  redact: {
    // Non-negotiable. Test this.
    paths: [
      "req.headers.authorization", "req.headers.cookie", "req.headers['x-api-key']",
      "req.headers['idempotency-key']", "res.headers['set-cookie']",
      "*.password", "*.token", "*.secret", "*.card_number", "*.cvv", "*.ssn",
    ],
    censor: "[REDACTED]",
  },
  formatters: { level: label => ({ level: label }) },
});
```

### What to log, and at which level

| Level | Use for | Alert on it? |
|---|---|---|
| `error` | **Your** faults — unhandled exceptions, 5xx, failed writes, downstream failures after retries | **Yes** |
| `warn` | Recoverable problems — a retry succeeded, a circuit opened, a deprecated endpoint was called, a rate limit was hit | Trend, not per-event |
| `info` | Business events — payment created, refund issued, key revoked, webhook delivered | No; these are your audit and analytics feed |
| `debug` | Development detail | Off in production (or sampled) |

**The rule that keeps your on-call sane:** *4xx is the client's fault → `warn` or `info`. 5xx is your fault → `error`.* Logging validation failures at `error` means a customer with a typo pages you at 3am, and after a week you stop trusting the alerts entirely — which is worse than having none.

### What to never log
Tokens, passwords, API keys, card numbers, CVV, full PAN, government IDs, session cookies, full request bodies on auth endpoints. **Write a test that asserts your redaction works**, because this regresses silently the moment someone logs a new object:
```ts
test("logs never contain credentials", async () => {
  const lines = await captureLogs(() => api.post("/v1/payments", body, { apiKey: "sk_live_SECRET" }));
  const blob = lines.join("\n");
  for (const needle of ["sk_live_SECRET", "Bearer ", "password", body.card?.number]) {
    if (needle) expect(blob).not.toContain(needle);
  }
});
```

### Sampling
At high volume, log every error and warning, but **sample** info-level access logs (say 1–10%) — while always keeping 100% of a trace's logs when that trace is sampled, so a sampled trace is never missing its logs. Also always log 100% for a specific merchant when you're debugging them (a per-tenant debug flag is a genuinely useful feature to build).

---

## 5. Metrics: the four that matter

### RED (for request-driven services) — use this
| Metric | Meaning |
|---|---|
| **R**ate | Requests per second |
| **E**rrors | Failed requests per second (and as a ratio) |
| **D**uration | Latency **distribution** |

(**USE** — Utilisation, Saturation, Errors — is the equivalent for resources: CPU, memory, connection pools, queue depth. Google's **Four Golden Signals** are latency, traffic, errors, saturation. They're the same idea with different names; know all three names.)

```ts
import { Counter, Histogram, Gauge } from "prom-client";

const httpRequests = new Counter({
  name: "http_requests_total",
  help: "Total HTTP requests",
  labelNames: ["method", "route", "status_class"] as const,   // ← BOUNDED labels only
});

const httpDuration = new Histogram({
  name: "http_request_duration_seconds",
  help: "Request duration",
  labelNames: ["method", "route", "status_class"] as const,
  // Buckets chosen to bracket YOUR actual SLO, not library defaults.
  buckets: [0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10],
});

const inFlight = new Gauge({ name: "http_requests_in_flight", help: "Concurrent requests" });

export function metrics(req: Request, res: Response, next: NextFunction) {
  const end = httpDuration.startTimer();
  inFlight.inc();
  res.on("finish", () => {
    // Use the ROUTE PATTERN, never the raw URL.
    const route = req.route?.path ? `${req.baseUrl}${req.route.path}` : "unmatched";
    const labels = { method: req.method, route, status_class: `${Math.floor(res.statusCode / 100)}xx` };
    end(labels);
    httpRequests.inc(labels);
    inFlight.dec();
  });
  next();
}
```

### The two metric mistakes

**1. Cardinality explosion.** Every unique label combination is a separate time series.
```ts
// ❌ one series per payment. Millions of series. Your metrics store dies.
labelNames: ["method", "route", "payment_id", "merchant_id"]

// ✅ bounded: ~10 methods × ~40 routes × 5 status classes = 2,000 series
labelNames: ["method", "route", "status_class"]
```
**The rule: labels must have low, bounded cardinality.** No IDs, no emails, no URLs, no user agents. **Raw URLs are the classic trap** — `/v1/payments/pi_3Nx8` as a label is one series per payment. Always use the route *pattern*.

High-cardinality questions ("what happened to merchant mrc_9s2k?") belong in **logs and traces**, which are designed for it. That division is the answer to *"why not just put merchant_id in the metric?"*

**2. Averages.** An average latency of 200ms is consistent with "everyone gets 200ms" and with "90% get 20ms and 10% get 2s." **Only the second is an incident**, and the average hides it.
```
p50 =  40 ms     ← the typical experience
p95 = 180 ms
p99 = 4,200 ms   ← 1 in 100 requests is terrible. THIS is what users complain about
```
Always histograms, always percentiles, and **always p99** — because at 1M requests/day, p99 is 10,000 unhappy requests. And the follow-up worth knowing: **percentiles don't average across instances**, so you must aggregate histogram *buckets* (which is exactly why Prometheus histograms exist) rather than averaging each instance's p99.

### The business metrics that matter more than the technical ones
```ts
const paymentsCreated  = new Counter({ name: "payments_created_total", labelNames: ["currency", "status"] });
const paymentAmount    = new Histogram({ name: "payment_amount_minor", labelNames: ["currency"] });
const webhookDeliveries= new Counter({ name: "webhook_deliveries_total", labelNames: ["result", "attempt"] });
const circuitState     = new Gauge({ name: "circuit_breaker_open", labelNames: ["dependency"] });
const queueDepth       = new Gauge({ name: "outbox_pending_total" });
const dbQueriesPerReq  = new Histogram({ name: "db_queries_per_request", buckets: [1,2,5,10,20,50,100] });
```

That last one is the N+1 detector from [Lesson 17](17-caching-and-performance.md). And the most important one is often the business metric: **a drop in `payments_created_total` is a better outage signal than a rise in 5xx**, because it catches the failures where nothing errored — a bad deploy that silently rejects valid input, or a config change that points you at an empty database. *"Alert on the business metric, not just the technical one"* is a genuinely senior thing to say.

---

## 6. Distributed tracing

A trace is one request's journey; a span is one operation within it.

```
Trace 4bf92f3577b34da6a3ce929d0e0e4736                       total: 847 ms
├─ POST /v1/payments                                    [████████████] 847 ms
│  ├─ authenticate                                      [█]             12 ms
│  ├─ validate                                          [▏]              1 ms
│  ├─ idempotency.claim            (postgres INSERT)    [█]              8 ms
│  ├─ db.customers.find            (postgres SELECT)    [█]              6 ms
│  ├─ processor.charge             (HTTP acme_uk)       [█████████]    780 ms  ← 92%
│  │  ├─ attempt 1  timeout                             [██████]       500 ms
│  │  └─ attempt 2  ok                                  [███]          280 ms
│  ├─ db.payments.insert           (postgres INSERT)    [█]              9 ms
│  └─ outbox.insert                (postgres INSERT)    [▏]              4 ms
```

**Read that trace and the answer is immediate:** 92% of the time is one downstream call, and 500ms of it was a timed-out first attempt. No amount of database optimisation would have helped. **That's why tracing exists** — it tells you *where* the time went, which is a question logs and metrics cannot answer.

```ts
import { trace, SpanStatusCode } from "@opentelemetry/api";
const tracer = trace.getTracer("ledger-api");

export async function chargeCard(payment: Payment) {
  return tracer.startActiveSpan("processor.charge", async span => {
    // Low-cardinality on metrics; HIGH-cardinality is FINE and desirable on spans.
    span.setAttributes({
      "payment.id": payment.id,
      "payment.amount_minor": payment.amountMinor,
      "payment.currency": payment.currency,
      "merchant.id": payment.merchantId,
      "processor.name": "acme_uk",
    });
    try {
      const result = await processor.charge(payment);
      span.setAttribute("processor.result", result.status);
      return result;
    } catch (e) {
      span.recordException(e as Error);
      span.setStatus({ code: SpanStatusCode.ERROR, message: (e as Error).message });
      throw e;
    } finally {
      span.end();
    }
  });
}
```

Key points:
- **Use OpenTelemetry.** It's the vendor-neutral standard; you can switch from Jaeger to Datadog to Honeycomb without re-instrumenting. This is the correct answer to "which tracing tool?"
- **Context propagates via the `traceparent` header** (W3C Trace Context). Auto-instrumentation handles most of it; you must not strip the header at your gateway.
- **Sample intelligently:** head-based sampling (1–10%) is cheap; **tail-based** sampling (decide *after* the trace completes) lets you keep 100% of errors and slow requests and 1% of the boring ones. Tail-based is what you want, and naming it is a differentiator.
- **High-cardinality attributes belong here**, not on metrics. This is the division of labour that makes the three pillars coherent.

---

## 7. Health checks — and the mistake almost everyone makes

Three endpoints, three different jobs:

```ts
// 1. LIVENESS — "is this process alive?" Kubernetes RESTARTS the pod if this fails.
//    ⚠️ It must NOT check dependencies.
app.get("/healthz", (_req, res) => res.status(200).json({ status: "ok" }));

// 2. READINESS — "should this instance receive traffic?" K8s removes it from the LB.
//    It MAY check the dependencies this instance needs to serve.
app.get("/readyz", async (_req, res) => {
  const checks = await Promise.allSettled([
    withTimeout(db.query("SELECT 1"), 1000),
    withTimeout(redis.ping(), 500),
  ]);
  const ok = checks.every(c => c.status === "fulfilled");
  res.status(ok ? 200 : 503).json({
    status: ok ? "ready" : "not_ready",
    checks: { postgres: checks[0].status, redis: checks[1].status },
  });
});

// 3. DEEP HEALTH — for humans and dashboards. Never wired to an orchestrator.
app.get("/health/detail", requireInternalAuth, async (_req, res) => {
  res.json({
    version: process.env.GIT_SHA, uptime_s: process.uptime(),
    dependencies: await checkAll(),           // includes non-critical ones
    circuit_breakers: breakerStates(),
    outbox_pending: await countOutbox(),
  });
});
```

> **The mistake, and it causes real outages:** putting a database check in your **liveness** probe. Your database has a 30-second blip. Every pod's liveness probe fails. Kubernetes restarts **every pod simultaneously**. Now the database is fine but your entire fleet is cold-starting, caches are empty, connection pools are re-establishing, and the outage is ten times longer than the original blip. **Liveness = "is my process wedged?" Readiness = "can I serve right now?"** Explaining this distinction is one of the highest-signal things you can say about operating services, because it's specific, counter-intuitive, and only learned by having done it.

Also: **readiness must fail during graceful shutdown**, *before* you stop accepting connections, so the LB drains you first. Otherwise every deploy drops in-flight requests.

---

## 8. SLOs, error budgets and alerting

### Define an SLO, not a vague target
```
SLI  (indicator): the proportion of requests to POST /v1/payments that
                  return non-5xx within 500 ms
SLO  (objective): 99.9% over a rolling 30 days
Budget:           0.1% of 30 days ≈ 43 minutes of "bad" per month
```

**The error budget is the useful part**, because it converts reliability into a decision-making tool: budget remaining → ship features. Budget exhausted → stop feature work and fix reliability. That's an engineering-management answer that few candidates give, and it lands well.

### Alert on symptoms, not causes
| ❌ Cause-based (noisy) | ✅ Symptom-based (actionable) |
|---|---|
| CPU > 80% | p99 latency > 1s for 5 minutes |
| Disk > 70% | 5xx rate > 1% for 5 minutes |
| A pod restarted | Error budget burning at >10× the sustainable rate |
| Redis memory high | `payments_created_total` dropped >50% vs the same hour last week |

High CPU with healthy latency is not an incident — it's a machine doing work. **Alert on what users experience.** Then use **multi-window burn-rate alerts** (a fast window to catch sudden severe burn, a slow window to catch slow leaks) so you get paged for real problems and not for noise. Every alert must be **actionable** and carry a **runbook link**; an alert nobody can act on trains people to ignore alerts, which is worse than silence.

---

## 9. The debugging playbook

This is the deliverable of the lesson. Memorise the shape — it's the answer to every *"how would you debug…"* question.

### "The API is slow"
```
1. WHO?      All clients or one? (per-merchant p99)  → one client = their query shape or their load
2. WHERE?    Split the latency: DNS / TCP / TLS / server-think / transfer   (curl -w, Lesson 02)
3. WHICH?    All endpoints or one?  (p99 by route)
4. WHEN?     Did it start at a deploy? A cron? A traffic change? (overlay deploy markers)
5. INSIDE?   Open a slow trace. Where did the 90% go?
                 → downstream call  → its own latency, or a retry storm (Lesson 18)
                 → database         → query count (N+1?), missing index, lock waits, pool saturation
                 → CPU/event loop   → blocking work, GC, a big JSON.stringify
                 → queue wait       → saturation; check in-flight gauge and pool metrics
6. CONFIRM?  Form a hypothesis, then find the metric that would DISPROVE it.
```

### "Requests are failing"
```
1. Which status? 4xx (client) vs 5xx (you) — completely different investigations
2. 5xx: one route or all? one instance or all?  (metrics by route AND instance)
   → one instance → that host: bad deploy, full disk, wedged process. Drain it
   → all instances → a shared dependency, a deploy, or a config change
3. Grab a request_id from a failing request → its logs → its trace → the exception
4. 4xx spike: usually a client deploy, an expired credential, or YOUR breaking change
   → check per-client 4xx rates. One client = their bug. All clients = yours (Lesson 11)
5. Check the error CODE distribution, not just the status (Lesson 10)
```

### "A customer reports a specific failure"
```
1. Ask for the X-Request-Id (you returned it — Lesson 04)
2. One query: all logs for that request_id → the exact error and code
3. Open the trace for its trace_id → which span failed and why
4. Check that merchant's rate-limit and quota state
5. Reproduce with their exact request shape in a test → then write the regression test
```

**If any step in those playbooks is impossible in your system, that's your next observability task.** That framing — *"the gap in the playbook is the backlog item"* — is a genuinely useful thing to say in an interview.

---

## 10. Production rules

| Rule | Why |
|---|---|
| **`X-Request-Id` on every response, errors included** | Turns support archaeology into one query |
| **Propagate `traceparent` and `x-request-id` on outbound calls** | Otherwise the chain breaks at your boundary |
| **`AsyncLocalStorage` for implicit context** | Every log/metric/span gets IDs without threading a parameter |
| **Structured JSON logs; never string interpolation** | You cannot query prose |
| **Redact credentials, and test the redaction** | It regresses silently |
| **4xx at warn/info, 5xx at error** | Otherwise client typos page you and you stop trusting alerts |
| **Metric labels must be bounded — route patterns, never raw URLs or IDs** | Cardinality explosion kills your metrics store |
| **Histograms and percentiles; never averages** | Averages hide the tail that users feel |
| **High-cardinality detail on traces and logs, not metrics** | That's what each is designed for |
| **Liveness must not check dependencies** | A DB blip becomes a full fleet restart |
| **Readiness fails first during graceful shutdown** | So the LB drains before you stop serving |
| **Alert on symptoms with runbook links; delete alerts nobody acts on** | Alert fatigue is worse than no alerts |
| **Track at least one business metric as a health signal** | Catches silent failures that never throw |
| **Log query count per request** | Automatic N+1 detection |
| **Tail-based sampling: 100% of errors and slow traces, ~1% of the rest** | Full fidelity where it matters, affordable everywhere else |

> **Spring equivalent:** Micrometer + Actuator (`/actuator/health/liveness`, `/readiness`) + Micrometer Tracing (Brave/OTel) + MDC for correlation. Identical concepts — see `spring boot/09-production/35-observability.md`. Note MDC is the thread-local analogue of `AsyncLocalStorage`.

---

## 11. Interview traps

**Q1. "Monitoring vs observability?"**
Monitoring answers pre-defined questions ("is CPU high?"); observability lets you ask new ones without shipping code ("p99 for this route, this merchant, this API version, right now"). The practical enabler is **high-cardinality context on logs and traces**, tied together by one ID.

**Q2. "Your p99 spiked to 4s while p50 stayed at 40ms. Walk me through it."**
Use the §9 playbook and say it as a method: who / where / which / when / inside / confirm. Concrete branches: geography (RTT) if it's client-specific; a downstream retry storm; a lock or pool saturation; GC or event-loop blocking; and *"if p99 ≈ exactly my timeout value, I'm measuring the timeout, not the work."* Finish with *"and I'd look for the metric that would disprove my hypothesis before shipping a fix."*

**Q3. "Why not use averages?"**
An average of 200ms fits both "everyone gets 200ms" and "90% get 20ms, 10% get 2s" — only the second is an incident. Use histograms and percentiles. Bonus: percentiles can't be averaged across instances, which is why you aggregate buckets.

**Q4. "What's cardinality and why does it matter?"**
Each unique label combination is a time series. Adding an ID label creates one series per entity — millions — which kills your metrics backend and your bill. Bounded labels on metrics; high-cardinality on traces and logs.

**Q5. "Should a health check verify the database?"**
**Readiness yes, liveness no.** Then give the failure mode: a DB blip fails every liveness probe, Kubernetes restarts the whole fleet, and a 30-second dependency blip becomes a 10-minute cold-start outage.

**Q6. "A customer says a request failed at 14:03. What do you need?"**
The `X-Request-Id` — which means you must have returned it on the error response. Then logs by that ID, the trace by `trace_id`, their rate-limit state, and a reproduction. If you can't do that, the observability gap *is* the bug.

**Q7. "What do you alert on?"**
Symptoms users feel: error rate, latency percentiles, error-budget burn rate, and a business metric (payment volume). Not CPU, not disk, not pod restarts. Every alert needs an action and a runbook.

**Q8. "What's an SLO and an error budget?"**
An SLI is the measurement, an SLO is the target over a window, and the error budget is `1 − SLO` — the amount of unreliability you're allowed. Its value is as a decision rule: budget left → ship features; budget gone → fix reliability.

**Q9. "How do you trace a request across five services?"**
W3C Trace Context: a `traceparent` header carrying trace ID, span ID and flags, propagated on every hop; each service creates child spans. Use OpenTelemetry so it's vendor-neutral. Tail-based sampling to keep all errors and slow traces affordably.

**Q10. "Your logs cost more than your compute. What do you do?"**
A real problem at scale. Sample info-level logs (keeping 100% of errors and of sampled traces), drop redundant fields, shorten retention with a cheap archive tier for compliance, move high-volume per-request detail from logs into traces, and audit for accidental debug logging in hot paths. **And check your metric cardinality** — that's usually the bigger bill.

---

## 12. Build & break

### Build — the observability layer for Ledger
1. `correlation` + `AsyncLocalStorage` context, with `X-Request-Id` on every response.
2. `pino` with `mixin` and `redact`, plus the credential-leak test.
3. Prometheus metrics: RED with **bounded** labels, plus `payments_created_total`, `db_queries_per_request`, `circuit_breaker_open`, `outbox_pending_total`.
4. OpenTelemetry with auto-instrumentation for HTTP and Postgres, and manual spans around processor calls.
5. `/healthz`, `/readyz`, `/health/detail` — with the liveness/readiness split respected.
6. `/metrics` behind internal auth (it leaks your topology otherwise).

### Build — the incident drill
Break something on purpose, then debug it **using only your telemetry** — no reading the code. Time yourself.

| Injected fault | What you should be able to see |
|---|---|
| Add 2s of latency to 5% of processor calls | p99 spike with a flat p50; a trace showing the processor span dominating |
| Introduce an N+1 in the list endpoint | `db_queries_per_request` jumps from 2 to 200 |
| Make Redis time out | Readiness fails; the rate limiter fails open; a metric records it |
| Deploy a bad validation rule | A 4xx spike, with `error_code` distribution showing exactly which code |
| Silently point at an empty read replica | **Zero 5xx**, but `payments_created_total` drops — proving why business metrics matter |
| Put the DB check in liveness, then blip the DB | Watch the whole fleet restart. **Do this once; you'll never forget it** |

For each: write down the *first* signal you noticed and how long the diagnosis took. That log is worth more than any dashboard.

### Break — three anti-patterns to feel
1. **Cardinality bomb.** Add `payment_id` as a metric label, generate 100k payments, and watch your Prometheus memory (or your bill). Then remove it.
2. **Averages.** Chart mean latency while injecting 2s into 1% of requests. The mean barely moves. Chart p99 — it's obvious. Now you know why the mean is useless.
3. **Alert fatigue.** Alert on CPU > 70% and run a load test. Count the pages you'd have received for a perfectly healthy service.

### Explain out loud (2 minutes)
1. The three pillars and what each is for.
2. Why correlation IDs are the highest-value addition.
3. The cardinality rule, and where high-cardinality data belongs.
4. Liveness vs readiness, and the outage the confusion causes.
5. The "API is slow" playbook, as six steps.

---

## Module 4 complete — checkpoint

Cold, no notes:

- [ ] `no-cache` vs `no-store` vs `must-revalidate`
- [ ] The `private, no-cache, ETag, Vary` recipe, and why it fits authenticated data
- [ ] How a 304 works and what it saves
- [ ] What `Vary` prevents (and why it's a security control)
- [ ] Cache stampede, penetration and invalidation — with fixes
- [ ] Why you would refuse to cache a balance
- [ ] Detect and fix a server-side N+1
- [ ] The fundamental ambiguity, and why exactly-once doesn't exist
- [ ] Design idempotency-key storage on a whiteboard, including the 5xx rule
- [ ] Backoff with full jitter; retry amplification; retry budgets
- [ ] What a circuit breaker buys, and who it protects
- [ ] Bulkheads and load shedding
- [ ] The dual-write problem and the outbox pattern
- [ ] Five rate-limit algorithms and their failure modes
- [ ] Why the Redis limiter must be a Lua script
- [ ] The 429 contract, and `RateLimit-*` on every response
- [ ] Fail-open vs fail-closed, and where each belongs
- [ ] The three pillars, cardinality, and percentiles vs averages
- [ ] Liveness vs readiness
- [ ] The "API is slow" playbook

---

## What's next

REST is not the only shape an API can take, and *"REST vs GraphQL vs gRPC"* is a guaranteed interview question. Module 5 covers each alternative properly — what it optimises, what it costs, and how to choose — starting with the one that trades away everything HTTP gave you in exchange for letting clients ask for exactly what they need.

Next → **[Lesson 21: GraphQL, honestly](../05-beyond-rest/21-graphql.md)**
