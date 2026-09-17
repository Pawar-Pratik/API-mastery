# Lesson 27 — The complete API checklist & code review guide

> **Why this lesson exists:** knowledge you can't apply under time pressure isn't knowledge yet. This lesson turns Modules 1–6 into a **repeatable 15-minute review** you can run against your own design, a colleague's PR, or a whiteboard in an interview. It doubles as the fastest revision pass over the entire track — read it end to end and you're re-testing every concept in order.

**Time:** ~60 minutes first pass, then it's a reference you return to · **Prereq:** Modules 1–6

---

## 1. The idea in one sentence

> **A good review isn't a list of opinions — it's an ordered sweep through the places APIs actually fail, so that the important problems are found first and stated with a reason.**

---

## 2. The 15-minute review, in order

The ordering matters: **each pass is more expensive to fix later than the one before it.** URL design is cheap to change before launch and permanent after; a missing authorization check is a breach whenever it ships.

### Pass 1 — The URL list (2 min)
Read only the route list. Before any implementation.

```
[ ] Nouns, not verbs (except deliberate action sub-paths, POST-only, verb last)
[ ] Plural collections, consistently
[ ] One casing convention, consistently
[ ] Max one level of nesting
[ ] Versioned, at the leftmost path segment
[ ] Filters in the query string, not the path
[ ] Consistent ID scheme: opaque, prefixed, non-sequential
[ ] Predictable — after 3 endpoints, can I guess the 4th?
```
**The test:** hide the docs and ask someone to guess three endpoints. If they can't, the design has failed regardless of how good the code is. → [Lesson 07](../02-rest-design/07-resource-modelling-and-urls.md)

### Pass 2 — Methods and status codes (2 min)
```
[ ] GET is genuinely side-effect free
[ ] PUT is a full replacement (or PUT isn't offered at all)
[ ] PATCH's body format is documented (merge-patch preferred)
[ ] 201 carries Location; 204 carries no body
[ ] 401 carries WWW-Authenticate; 405 carries Allow; 429/503 carry Retry-After
[ ] No 200-with-an-error-body, anywhere
[ ] 4xx vs 5xx is correct (client's fault vs mine) — retry logic depends on it
[ ] 404 (not 403) for cross-tenant resources
[ ] Long operations return 202 + a job resource, not a 90-second GET
```
→ [Lesson 03](../01-foundations/03-http-methods-and-status.md), [Lesson 10](../02-rest-design/10-errors-and-problem-details.md)

### Pass 3 — Authorization (3 min) — **the highest-value pass**
This is where breaches live. Spend the most time here.
```
[ ] Tenancy derived ONLY from the credential — never body, query, header or cursor
[ ] Every data-access function requires a tenant argument (or uses a scoped repo)
[ ] Nested routes validate the FULL parent chain
[ ] Endpoint-level permission on every route; default-deny
[ ] Object-level check inside every handler that takes an ID
[ ] Field-level: input schemas are .strict() and contain only client-settable fields
[ ] No server-controlled fields accepted (id, status, balance, role, merchant_id, created_at)
[ ] Scopes enforced, not merely granted
[ ] An authorization matrix test exists, and CI fails on uncovered routes
```
**The single question to ask of any handler:** *"if I change the ID in this URL to another tenant's, what happens?"* If the answer isn't "404", stop the review. → [Lesson 15](../03-security/15-authorization-and-multitenancy.md)

### Pass 4 — Writes and concurrency (2 min)
```
[ ] Idempotency-Key required on every money-moving / side-effecting POST
[ ] Idempotency scoped by tenant + endpoint, with a request-body hash
[ ] 5xx releases the key; 4xx is stored
[ ] The key claim is atomic (INSERT … ON CONFLICT), never read-then-write
[ ] ETag + If-Match on mutable resources; 412 on stale, 428 if required and absent
[ ] The version predicate is in the UPDATE … WHERE, not in application code
[ ] State transitions are guarded in the WHERE clause, not by an `if`
[ ] Bulk: 207 with per-item index, per-item idempotency, per-item rate metering
```
→ [Lesson 09](../02-rest-design/09-writes-patch-and-bulk.md), [Lesson 18](../04-production/18-reliability-and-idempotency.md)

### Pass 5 — Collections (2 min)
```
[ ] Cursor pagination (offset only for small, bounded sets)
[ ] A unique tiebreaker in ORDER BY and in the cursor
[ ] Bounded limit with a documented max and default
[ ] Filterable and sortable fields are ALLOWLISTED
[ ] An index exists for every filter/sort combination exposed (verified with EXPLAIN)
[ ] Unknown query parameters → 400, never silently ignored
[ ] Empty result → 200 with [], never 404
[ ] Envelope object at the top level, never a bare array
[ ] No exact total count on a large collection
```
→ [Lesson 08](../02-rest-design/08-collections-and-pagination.md)

### Pass 6 — Errors (1 min)
```
[ ] One central error handler; zero inline error shapes
[ ] Stable snake_case `code` on every error
[ ] One code per distinct client action (token_expired ≠ invalid_api_key)
[ ] request_id in every error body AND header
[ ] All validation errors returned at once, with field paths
[ ] No stack traces, SQL, hostnames or upstream bodies leaked
[ ] A published error table with a `retryable` column
```
→ [Lesson 10](../02-rest-design/10-errors-and-problem-details.md)

### Pass 7 — Reliability and limits (2 min)
```
[ ] Timeouts on every outbound call; decreasing downstream
[ ] Retries only for idempotent operations on transient failures
[ ] Exponential backoff with FULL jitter; capped attempts; a retry budget
[ ] Circuit breaker + bulkhead per dependency
[ ] Load shedding: 503 + Retry-After when saturated
[ ] Rate limits keyed on identity (IP only for anonymous), enforced atomically
[ ] RateLimit-* on every response; Retry-After on every 429
[ ] Body size, array length, nesting depth all bounded
[ ] Per-tenant quotas on anything that costs money downstream
[ ] Outbox for anything that must happen after a commit
```
→ [Lessons 18](../04-production/18-reliability-and-idempotency.md), [19](../04-production/19-rate-limiting.md)

### Pass 8 — Caching and performance (1 min)
```
[ ] Explicit Cache-Control on every response
[ ] Authenticated data: private (or no-store) + Vary: Authorization
[ ] ETag on GETs, checked BEFORE the expensive work
[ ] Compression on text responses
[ ] Connection reuse (keep-alive) on outbound clients
[ ] No N+1: query count per request is logged and bounded
[ ] Nothing cached that can't afford to be stale (never a balance)
```
→ [Lesson 17](../04-production/17-caching-and-performance.md)

### Pass 9 — Evolution (1 min)
```
[ ] Unknown request-body fields rejected; unknown query params rejected
[ ] Response consumers documented as tolerant readers
[ ] Enums documented as open; clients told to handle unknown values
[ ] No field's meaning ever repurposed
[ ] Deprecation + Sunset + a migration Link on anything retiring
[ ] Per-client usage instrumented for anything you plan to remove
[ ] oasdiff breaking (or equivalent) is a required CI check
```
→ [Lesson 11](../02-rest-design/11-versioning-and-evolution.md)

### Pass 10 — Observability and hygiene (1 min)
```
[ ] X-Request-Id on every response, propagated outbound
[ ] Structured logs carrying request/trace/tenant IDs
[ ] Credentials redacted in logs, with a test asserting it
[ ] 4xx at warn/info, 5xx at error
[ ] RED metrics with BOUNDED labels (route patterns, never raw URLs)
[ ] Histograms/percentiles, never averages
[ ] Liveness does NOT check dependencies; readiness may
[ ] At least one business metric alerted on
[ ] helmet-equivalent headers; x-powered-by removed; trust proxy set to a hop count
```
→ [Lessons 20](../04-production/20-observability.md), [16](../03-security/16-owasp-and-hardening.md)

---

## 3. The severity scale — say this, not "I'd prefer"

A review is only useful if the reader knows what to do first. Grade every finding.

| Level | Meaning | Examples |
|---|---|---|
| **🔴 Blocker** | Ship this and you have an incident or a breach | Missing tenancy scope; mass assignment on a privileged field; no idempotency on a charge endpoint; secrets in logs; SSRF on a user-supplied URL; unbounded body parsing |
| **🟠 Must-fix before launch** | Permanent once clients exist | Breaking URL/naming inconsistency; offset pagination on a large collection; no error `code`s; no versioning; unbounded `limit`; missing `Retry-After` |
| **🟡 Should-fix** | Real cost, but fixable later without breaking clients | No ETag/caching; N+1; no request ID; averages instead of percentiles; missing rate-limit headers |
| **🔵 Nit** | Preference. **Say it's a nit** | `snake_case` vs `camelCase` (given consistency); `data` vs `items`; 400 vs 422 given a house convention |

**The discipline that makes you good at reviews:** never present a 🔵 with the same energy as a 🔴. A review that lists twelve nits and buries one missing authorization check has failed. Lead with blockers, state the consequence, and mark preferences as preferences.

### How to phrase a finding
```
❌ "You should use cursor pagination."
✅ "🟠 `GET /payments` uses OFFSET. At 4M rows, page 200k reads 4M rows (~20s) and
    concurrent inserts cause duplicate/skipped records — so an exporting client
    silently misses transactions. Suggest keyset pagination on (created_at, id).
    Lesson 08 has the SQL."
```
The pattern: **severity → what → the concrete consequence → a suggested fix.** The consequence is the part that gets it fixed; without it, you're just asserting taste.

---

## 4. The 12 findings you'll make most often

In roughly the order you'll encounter them in real code:

| # | Finding | Severity | Why it's this common |
|---|---|---|---|
| 1 | Missing tenancy scope in a query | 🔴 | Easiest bug to write, hardest to see |
| 2 | `Object.assign(entity, req.body)` / no `.strict()` | 🔴 | It looks concise |
| 3 | No idempotency on a side-effecting POST | 🔴 | Works fine until the first network blip |
| 4 | No timeout on an outbound call | 🔴 | Every HTTP client defaults to none |
| 5 | Unbounded `limit` / body size | 🟠 | Nobody tests the huge input |
| 6 | Offset pagination | 🟠 | It's the tutorial default |
| 7 | `200` with an error in the body | 🟠 | Feels friendly; breaks all tooling |
| 8 | Error messages with no stable `code` | 🟠 | Nobody notices until a client string-matches |
| 9 | Entities serialized directly (no output DTO) | 🟠 | Convenient; publishes every future column |
| 10 | No `Cache-Control` anywhere | 🟡 | Requires a deliberate decision nobody makes |
| 11 | Server-side N+1 | 🟡 | Invisible without query-count logging |
| 12 | Unknown query params silently ignored | 🟡 | The framework default |

**If you check only these twelve, you'll catch most of what matters.** That's a genuinely useful thing to be able to do in 5 minutes.

---

## 5. Reviewing a *design* (before code exists)

Different questions when there's nothing to run. Ask these ten, in order:

1. **"Write the client code."** Show me the five lines a consumer writes for the main use case. If it takes three round trips and a `setTimeout`, the design is wrong.
2. **"What's the largest collection here in two years?"** Decides pagination and indexing.
3. **"Which of these operations moves money or sends a message?"** Those need idempotency keys.
4. **"What happens if this is called twice?"** For every write.
5. **"What happens if two clients call it simultaneously?"** For every update.
6. **"Whose data is this, and where does the tenant come from?"** Must be "the credential."
7. **"What can the client set, and what does the server own?"** The mass-assignment boundary.
8. **"How long does the slowest call take?"** Over ~2s, you need 202 + a job resource.
9. **"What does the client do on each error?"** If two errors need different actions, they need different codes.
10. **"What's the first thing you'll want to change in six months, and can you?"** Tests evolvability before it's frozen.

Question 1 is the most valuable and the least used. **Writing the client first finds design flaws for free**, before anyone has implemented anything.

---

## 6. Reviewing your own work — the pre-merge gate

```
BEFORE I open the PR:
[ ] I ran the 10 passes on my own diff
[ ] Every new endpoint has an authorization test (the matrix meta-test passes)
[ ] Every new write has an idempotency or concurrency test
[ ] I tested the error paths, not just the happy path
[ ] The response validates against openapi.yaml, and oasdiff shows no breaking change
[ ] I looked at the actual SQL (EXPLAIN) for any new query
[ ] I broke it on purpose once, and the failure was legible
[ ] Logs from my new code contain no credentials
[ ] I can state what happens when each dependency of this code is down
```

That last item is the one that separates careful from fast. **For every new outbound call, know the answer to "what if this is slow, down, or returns garbage?"** — and have it in the code, not just in your head.

---

## 7. Interview traps

**Q1. "Review this API."** *(They show you a spec or a handler.)*
Don't free-associate. **Say your method out loud, then execute it:** *"I'll go through URLs, then methods and status codes, then authorization, then writes and concurrency, then collections, errors, reliability, caching, evolution, and observability. Authorization is where I'll spend most time, because that's where APIs actually get breached."* Then grade every finding. **Announcing an ordered method is itself most of the score** — it shows you've done this before.

**Q2. "What's the first thing you look for?"**
Authorization — specifically, *"if I change the ID in this URL to another tenant's, what happens?"* Because it's OWASP #1, it's invisible in review, and everything else is recoverable.

**Q3. "How do you give feedback someone will act on?"**
Severity, then the concrete consequence, then a suggested fix — and mark preferences as preferences. *"A review that buries a missing authorization check under twelve naming nits has failed, regardless of how correct the nits were."*

**Q4. "What would you change about an API you've worked on?"**
A "tell me about your judgement" question dressed as a technical one. Answer with a real trade-off you'd revisit and *why* — e.g. *"we returned entities directly to move fast, and within a year we'd published four internal fields we couldn't remove. I'd have written output DTOs on day one; the boilerplate cost was an hour and the removal cost was a deprecation cycle."* Specific, self-aware, and shows you learned the mechanism, not just the rule.

**Q5. "You have 15 minutes to review a 5,000-line API PR. What do you do?"**
Don't read linearly. Prioritise: (1) the route list and any new endpoints, (2) every query — check tenancy scoping, (3) every input schema — check strictness, (4) every outbound call — check timeouts, (5) the error paths, (6) the tests, looking specifically for authorization and concurrency coverage. Say explicitly that **the diff you skip is the diff you trust the tests to cover** — which is why the authorization matrix meta-test matters so much.

---

## 8. Build & break

### Build — review something real
Pick a public API you haven't studied (Twilio, SendGrid, Shopify, Razorpay, Plaid) and run all 10 passes against its docs. Produce a one-page review with graded findings.

You will find real issues in real, respected APIs. That's the point: **it proves the checklist works, and it calibrates you** — you'll see that even excellent APIs make trade-offs, and you'll start recognising which ones were deliberate.

### Build — review Ledger
Run the checklist on your own Ledger implementation. **Be honest.** Write the findings into `docs/review-findings.md` with severities, then fix every 🔴 and 🟠 before starting the project build log. This is the exercise that converts the track into working code.

### Build — the review as a repo artifact
Add `.github/pull_request_template.md`:
```markdown
## API changes in this PR
- [ ] New/changed endpoints: <list>
- [ ] Breaking change? (see docs/api-policy.md) — if yes, link the migration plan
- [ ] Authorization test added for every new endpoint
- [ ] Idempotency/concurrency test added for every new write
- [ ] Error paths tested
- [ ] openapi.yaml updated; oasdiff shows no unintended break
- [ ] EXPLAIN checked for new queries
- [ ] What happens when each new dependency is down: <answer>
```
A template converts a checklist you remember into a checklist the team executes. That's a real, cheap engineering-culture improvement, and describing it in an interview is a leadership signal.

### Explain out loud (2 minutes)
1. The 10 passes, in order, and why the order is that way.
2. The one question you ask of every handler.
3. The severity scale, and why grading matters more than finding.
4. The three questions you'd ask about a design before any code exists.

---

## Module 6 complete — checkpoint

- [ ] Contract-first vs code-first, and what actually prevents drift
- [ ] The seven artifacts you generate from one OpenAPI spec
- [ ] The two CI checks that stop drift and breakage
- [ ] What OpenAPI cannot express
- [ ] Why integration is the highest-value test layer for an API
- [ ] The four property tests, and the assertion that matters in each
- [ ] Provider vs consumer-driven contract testing
- [ ] What you look for in a load test
- [ ] The 10 review passes, in order
- [ ] The 12 most common findings

---

## What's next

Everything so far has been technique. Now you build the thing: **Ledger**, a complete, hosted, documented payments API that exercises every lesson in this track — and becomes the reason your interview answers start with *"here's how I handled that"*.

Next → **[Lesson 28: Ledger blueprint](../07-project/28-ledger-blueprint.md)**
