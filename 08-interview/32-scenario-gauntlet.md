# Lesson 32 — Scenario gauntlet: "what would you do if…"

> **Why this lesson exists:** design rounds test what you'd build; scenario questions test what you'd *do* — and they're the closest an interview gets to simulating the job. They're also where experience is impossible to fake, because the right answer has a **shape**: contain, diagnose, fix, verify, prevent. Learn the shape and 40 examples, and you'll handle the 41st.

**Time:** ~90 minutes · **Prereq:** Modules 1–6

---

## 1. The answer shape

Every incident answer follows the same five beats. **Say them in this order** — the ordering is itself the signal, because juniors debug first and seniors contain first.

```
① CONTAIN    Stop the bleeding. Before you understand it.
② DIAGNOSE   Narrow it: who, where, when, which. Use telemetry, not intuition.
③ FIX        The smallest change that resolves it.
④ VERIFY     Prove it's fixed. Name the metric you'd watch.
⑤ PREVENT    The systemic fix + the regression test + the alert that would have caught it.
```

Two rules that make answers land:
- **Containment before root cause.** *"I'd disable the endpoint / revoke the key / roll back first, then investigate."* Understanding a fire is not the same as putting it out.
- **Always finish with ⑤.** An answer that stops at "and then I'd fix it" is half an answer. The prevention step is what separates someone who's been on call from someone imagining it.

And one honest addition that always plays well: **"and I'd communicate."* Status page, the affected customers, and your team. Interviewers at senior level are listening for whether you know an incident is partly a communication event.

---

## 2. Duplicates & idempotency (5)

**1. "A customer says they were charged twice for one order."**
① Contain: refund the duplicate immediately — the customer's money comes first, before you know why. ② Diagnose: find both charges by customer and timestamp; check whether they share an `Idempotency-Key` (a bug in your storage) or have different keys (the client generated a new key per attempt — a client-side bug). Check the request logs for a retry pattern. ③ Fix: depends which. ④ Verify: query for other pairs with the same signature in the last 30 days — **assume it's not the only one**. ⑤ Prevent: require the key rather than accepting requests without it; document that the key must be generated once per logical operation and reused on retry; add the concurrent-duplicate test that fires 10 identical requests and asserts one row.

> The two details that make this answer strong: refunding **before** diagnosing, and **searching for other victims**. Both are what you'd actually do.

**2. "Your idempotency check uses SELECT-then-INSERT. What's wrong?"**
It's a race: two concurrent requests both see "no existing key" and both proceed. Fix: `INSERT … ON CONFLICT DO NOTHING RETURNING key` — one atomic statement tells you whether you won. Then the test that proves it: 10 concurrent requests, assert exactly one 201.

**3. "A client complains that retrying gets 409 instead of the original response."**
They're retrying while the first request is still in flight — which is correct behaviour, but their retry timeout is shorter than your processing time. Fix on their side: back off and retry with the same key. Fix on yours: return `Retry-After` on the 409, and consider blocking briefly on the in-progress lock rather than 409ing instantly. ⑤ Document your p99 for that endpoint so clients can set sane timeouts.

**4. "Two different services both send the user a receipt email."**
Not an idempotency problem — a **deduplication** problem. Idempotency protects against *one* caller's retries; this is two callers each doing the right thing once. Fix: a `dedupe_key` derived from the business event (`receipt:pi_3Nx8`) checked at the notification service, plus clear ownership of who sends what. ⑤ Rate limit per *recipient*, so any future bug is capped at annoying rather than catastrophic.

**5. "Your idempotency keys table is 400GB."**
No TTL sweep. ① Contain: it's a capacity problem, so check disk headroom first. ③ Fix: delete rows older than 24h, in **batches** (a single `DELETE` of 400GB will lock and blow your WAL), then add the index on `created_at` if missing. ⑤ A scheduled sweeper, an alert on table size, and partitioning by day so cleanup becomes `DROP PARTITION` instead of `DELETE`.

---

## 3. Performance & latency (7)

**6. "p99 went from 200ms to 4s overnight. p50 unchanged."**
② Diagnose in order: is it one client or all (per-client p99)? One endpoint or all? Did it start at a deploy, a cron boundary, or a traffic change? Then open a slow trace and see where the time went. Flat p50 with a blown p99 means *some* requests hit something the rest don't — a specific tenant with much more data, a cache miss path, a lock, a cold shard, or a retry. ⑤ Alert on p99 per endpoint, not aggregate; an aggregate p99 hides one pathological caller.

**7. "The API got slower after you added a feature, but the new endpoint is fast."**
Look for shared-resource contention: the new endpoint holds DB connections longer (pool starvation), or does CPU-heavy work that blocks the Node event loop, or added a hot cache key that evicts others, or added an N+1 to a *shared* code path. Measure connection-pool wait time and event-loop lag — both are metrics you should already have.

**8. "One customer's requests are 10× slower than everyone else's."**
Almost always data volume: they have 4M rows where others have 4k, so a query that's index-assisted for others is a scan for them. Check their query plans specifically. Could also be their query shape (filters you didn't index) or an unbounded `limit`. ⑤ Test with a realistically-large tenant in CI, not an empty one — *"our staging data was 1,000 rows so we never saw it"* is the actual root cause of most of these.

**9. "Latency is fine in your region and terrible in Europe."**
RTT is distance-bound and you can't beat it. Split the latency (`curl -w`) to confirm it's network, not server. Fixes: a regional deployment or read replica, a CDN for cacheable reads, HTTP/2 or 3 with TLS 1.3 to cut handshake round trips, and fewer round trips per screen. **Say explicitly that reducing round trips beats making the server faster** when RTT dominates.

**10. "The API is fast but the app feels slow."**
The app is making too many *sequential* requests — a waterfall. Count requests per screen and look at the dependency chain. Fixes: `?expand=` to collapse a client-side N+1, a batch or composite endpoint, or parallelising independent calls. This is exactly the pain GraphQL addresses, and saying so shows you know *why* it exists.

**11. "CPU is at 30% but requests are queueing."**
Not CPU-bound. In Node: the event loop is blocked by synchronous work (big `JSON.parse`/`stringify`, sync crypto, regex backtracking) or the libuv threadpool (4 by default) is saturated. In a thread-per-request stack: all threads are parked on a slow downstream. Either way the fix is finding what's blocking, not adding CPU. **Check event-loop lag.**

**12. "Adding a Redis cache made things slower."**
Plausible outcomes to check: the cached value is bigger than the query result so serialization dominates; the hit rate is near zero (wrong key granularity or too-short TTL) so you added a hop for nothing; or you're making one Redis call *per item* in a loop instead of `MGET`. ⑤ Always measure hit rate before and after — a cache without a hit-rate metric is a guess.

---

## 4. Failures & outages (8)

**13. "Payments are failing. It's 3am. Go."**
① Contain: check if it's total or partial. Is it one processor, one region, one endpoint? If a deploy went out in the last hour, **roll back first** — it's the fastest hypothesis test available. ② Diagnose: error codes distribution (are they 5xx from us or upstream 502s?), the processor's status page, your circuit-breaker state, DB health. ③ Fix. ④ Verify with `payments_created_total` returning to its normal curve, not just 5xx dropping. ⑤ Post-mortem with a regression test and an alert; if there wasn't a rollback path, that's finding #1.

**14. "Intermittent 502s, roughly 1 in 200 requests, no pattern in the app logs."**
The "no app logs" clue is the answer: the request never reached your handler. Classic cause is an **idle-timeout mismatch** — the LB's idle timeout is longer than the app's keep-alive, so the app closes a connection the LB still thinks is good. Fix: make the app's keep-alive longer than the LB's idle timeout. Also check for OOM-killed workers and missing connection draining on deploy.

**15. "Your third-party processor starts taking 30 seconds."**
Narrate the cascade, in order: requests pile up → pool saturates → **unrelated endpoints** fail → clients retry, doubling load → health checks fail → the orchestrator restarts pods, dropping in-flight work → cold caches slow recovery. Fixes mapped: timeout, bulkhead, circuit breaker, load shedding, jittered retries, and **health checks that don't depend on the downstream**.

**16. "One tenant's traffic is degrading service for everyone."**
① Contain: rate limit that tenant specifically, now. ② Diagnose: is it volume, or one expensive query shape? ⑤ Prevent: per-tenant rate limits *and* per-tenant concurrency caps (rate limits alone don't stop queue starvation), fair queueing across tenants instead of FIFO, and **shuffle sharding** to bound the blast radius of any single tenant.

**17. "A deploy broke a client, but it passed all your tests."**
The tests didn't encode the client's expectation. Find what they depended on: usually strict response validation, key ordering, a field's absence, or an enum they switched on exhaustively. ① Roll back or ship a compat shim first. ⑤ Prevent: `oasdiff breaking` in CI, consumer-driven contract tests for known clients, a documented compatibility policy stating what may change, and per-client usage instrumentation.

**18. "Redis is down. What happens to your API?"**
Depends on what Redis does for you, and you should enumerate: rate limiting (fail open, with an alert — but fail *closed* on auth endpoints), caching (fall through to the DB, and watch for a stampede as everything misses at once), sessions (**this one is an outage** — which is an argument for a fallback or for JWTs on that path), and idempotency if you stored keys there (which is why they belong in Postgres). ⑤ Know and document the answer per dependency *before* it happens.

**19. "Your database failed over. What did clients see?"**
In-flight queries error (so 500s or 503s for the failover window, typically 30–120s), connection pools must reconnect, and any non-idempotent write in flight is **ambiguous** — which is exactly the Lesson 18 problem, and exactly why idempotency keys let clients retry safely. ⑤ Connection-pool retry config, a circuit breaker so you fail fast during failover rather than parking requests, and 503 + `Retry-After` rather than 500 so clients back off correctly.

**20. "Half your instances are serving errors, half are fine."**
That asymmetry is the diagnosis: it's a **partial deploy** (mixed versions), a per-instance resource problem (one host's disk full, one wedged process), or a config difference between them. ① Contain: drain the bad instances from the LB immediately. ② Then compare: version, config, and host metrics between a good and a bad instance. ⑤ Readiness probes that actually catch the condition, and deploys that halt on error-rate regression.

---

## 5. Security incidents (6)

**21. "An API key was committed to a public GitHub repo."**
① **Revoke immediately** — don't assess first. ③ Issue a replacement and get it to the customer. ② **Audit everything that key did**, which is only possible if you log `key_id` per request. ④ Notify the customer; loop in legal/compliance if customer data was accessed. ⑤ Prevent: secret scanning in CI, GitHub push protection, prefixed keys so scanners recognise them, shorter-lived credentials, and per-key anomaly alerting. **The step candidates forget is the audit.**

**22. "You notice one API key enumerating sequential IDs."**
② That's reconnaissance or an active BOLA attempt. Check whether any of those requests **succeeded** — if a cross-tenant read returned 200, you have a breach, not an attempt. ① Contain: rate limit or suspend the key; block the pattern at the WAF. ⑤ Prevent: the structural fixes (scoped repos, RLS, 404-not-403), non-sequential IDs, and an alert on a high 404 rate per key — which is the signature of enumeration.

**23. "A pentest reports that you can read other users' data by changing an ID."**
That's BOLA, OWASP #1, and it's a 🔴. ① Contain: patch that endpoint now. ② **Then audit every other endpoint that takes an ID** — if one is missing the check, others are. ⑤ Prevent structurally: tenant as a required repository argument, per-request scoped repos, RLS as a backstop, and the actor×endpoint matrix test that fails CI for any uncovered route. *"I'd remove the ability to write the bug, not just fix this instance."*

**24. "Someone is using your payments endpoint to test stolen cards."**
Business-flow abuse — nothing is technically broken. ① Contain: block the offending merchant/card fingerprints; tighten attempt limits. ⑤ Prevent: per-merchant and per-card-fingerprint attempt limits, a rising failure-ratio circuit breaker per merchant, platform-wide fingerprint blocking after N failures, velocity checks, and bot detection at the sensitive step. Note the real cost: gateway fees and your processor risk score.

**25. "Your webhook endpoint registration is being used to scan internal services."**
SSRF. ① Contain: stop outbound requests to non-public IPs at the **network layer** — that's the real control, not application validation. ⑤ Prevent: protocol/port allowlist, resolve-then-validate-the-IP, connect to the validated IP with the `Host` header, `redirect: manual` with per-hop re-validation, egress restrictions, IMDSv2. And **re-validate on every delivery**, not just registration — DNS changes.

**26. "You find `Authorization` headers in your logs."**
① Contain: this is a credential exposure — rotate anything that appeared, and restrict log access. ② Determine the window and who could read them. ⑤ Prevent: redaction in the logger config **plus a test asserting a sentinel credential never appears in captured logs**, because this regresses silently the moment someone logs a new object.

---

## 6. Client & contract problems (7)

**27. "A client is polling your API 100×/second."**
② Find out why: they're probably waiting for a state change (a payment to settle, a job to finish). ① Contain: rate limit them, with clear `Retry-After`. ③ Fix the underlying need: give them webhooks or SSE, and `ETag`/304 so polls are cheap. ⑤ **Publish rate limits and return `RateLimit-*` headers on every response** so clients can self-throttle. *"A client polling that hard is usually a design gap on my side, not malice."*

**28. "A client says your API returns 500 for their request but works for others."**
Ask for the `request_id` — which requires that you returned it. Then their exact payload: it'll be an input you don't handle (unicode, a huge field, an unusual number, a null where you assumed a value). A 500 means **unhandled input**, so it's your bug regardless of how odd the input is. ⑤ Schemathesis-style property fuzzing against your spec finds this class before customers do.

**29. "A client is sending you fields you don't recognise."**
Depends where: an unknown **body** field should 422 (a typo'd `currncy` silently charging in a default currency is worse than an error), and an unknown **query param** must 400 (it changes which data you return). Then check whether they're guessing at a feature they need — an undocumented field being sent hopefully is a product signal.

**30. "You need to remove a field 200 clients might use."**
Lead with **measurement**: per-client usage instrumentation, so you email the exact eleven accounts still reading it rather than guessing. Then expand/contract, `Deprecation`+`Sunset`+migration `Link` headers, a scheduled announced **brownout** (the only technique that reliably produces action), then removal at the promised date.

**31. "A partner integration breaks every time you deploy."**
They're depending on something undocumented — response ordering, timing, a field's absence, an error message string. ② Find out exactly what. ⑤ Then two changes: document the guarantee (or explicitly document that it *isn't* one), and add a contract test so their expectation is encoded in **your** CI rather than discovered in production. And a "what clients MUST do" section in your docs.

**32. "A client's clock is wrong and your HMAC verification rejects them."**
Diagnose: their timestamp is outside your ±5-minute window. Don't widen the window much — it's your replay protection. Better: return a **specific** error code (`signature_timestamp_out_of_range`) with your server time in the response, so they can detect the skew themselves. ⑤ Document the tolerance and the error code. *"A precise error turns a two-day support thread into a one-minute fix."*

**33. "A client says 'your API is confusing.'"**
Treat it as a real bug report, and get specifics: which endpoint, what did they expect. Recurring root causes are inconsistency (mixed casing, mixed conventions), missing semantics in docs (units, nullability, inclusivity), and unhelpful errors. ⑤ The fix is usually docs and error messages rather than the API, and the test is: hand a new developer a key and the docs and see if they succeed in 5 minutes unaided.

---

## 7. Data & consistency (7)

**34. "A client creates a resource, immediately GETs it, and gets 404."**
Read-after-write inconsistency — you're reading from a replica that hasn't caught up. Fixes, best first: **read your own writes** by routing a read that follows a write to the primary (or by using a session/consistency token); return the created object in the `201` so the client needn't re-read at all; or document the eventual consistency, which pushes retry logic onto every client. ⑤ **Returning the full object on create is the cheapest fix and removes the whole class.**

**35. "Balances don't match the sum of transactions."**
A reconciliation failure — the most serious class of bug in a financial system. ① Contain: freeze payouts if the discrepancy is material. ② Find the first divergent transaction by replaying entries chronologically. Common causes: a mutable `balance` column updated outside a transaction, a rounding error (floats!), a missing reversing entry on a refund, or a partial failure that wrote one side of a double entry. ⑤ Prevent: append-only entries with balance as a `SUM`, a `CHECK` constraint per transaction group, and a **continuous reconciliation job asserting `SUM(credits) − SUM(debits) = 0`** that alerts rather than waiting for someone to notice.

**36. "Two clients PUT the same resource and one update vanished."**
Lost update. Fix: `ETag` on read, `If-Match` on write, 412 on mismatch — **with the version predicate inside the `UPDATE … WHERE`**, not a separate read-then-compare, which is the same race one layer down. ⑤ And where possible design it away with append-only or field-level updates that commute.

**37. "Webhooks are arriving out of order and a client's state machine is corrupted."**
Expected behaviour, not a bug — retries reorder by construction. Fix on your side: ship a monotonic `sequence` and `created_at`, and **document that events may arrive out of order**. Fix on theirs: treat a webhook as a *signal to read current state*, not as a state transition. Say why you won't guarantee order: serial delivery per endpoint means one stuck event becomes a multi-day backlog.

**38. "A client processed the same webhook 5 times."**
At-least-once delivery means duplicates are guaranteed, so this is expected too. Their receiver must dedupe on the event ID. Check whether they're **acking slowly** — if they process synchronously for 30 seconds, you time out and retry while they're still working, and now they process it twice legitimately. ⑤ Document "ack fast, process async" and "dedupe on event ID" in your webhook guide; both are on you to communicate.

**39. "A migration locked the payments table for 8 minutes in production."**
① Contain: kill it if it's still running (and know whether that's safe for the specific statement). ⑤ Prevent, and this is the substantive answer: `ALTER TABLE … ADD COLUMN` with a non-volatile default is fast on modern Postgres but adding a `NOT NULL` without a default rewrites the table; index creation must use `CREATE INDEX CONCURRENTLY`; backfills must be **batched**; and any migration must set a `lock_timeout` so it fails fast rather than queueing every subsequent query behind it. Expand/contract for column renames — never a single-step rename.

**40. "You need to delete one tenant's data for GDPR."**
Test whether the system was designed for it. You need: a complete inventory of every table holding tenant data (which is why `tenant_id` everywhere matters), a deletion order that respects foreign keys, a decision on **immutable ledger entries** (financial records usually have a statutory retention period that *conflicts* with erasure — so you anonymise the personal fields and retain the financial ones, and you must be able to explain that), plus backups, logs, caches, search indexes, analytics warehouses and third-party processors. ⑤ Design the deletion path up front; retrofitting it is a project, and the ledger/erasure tension is the part interviewers are listening for.

---

## 8. The five meta-answers

Questions that aren't about any specific incident.

**"How do you decide what to work on during an incident?"**
Impact first: how many customers, is money affected, is data being lost or exposed. Then containment before diagnosis. Then explicitly split roles if there's more than one person — someone communicating, someone investigating. **The instinct to say "I'd stop the bleeding before I understood it" is the whole answer.**

**"When do you page someone?"**
When it's user-affecting and actionable. Error rate, latency percentiles, error-budget burn rate, and a business metric. Not CPU, not disk, not a pod restart. **Every alert needs an action and a runbook link, and an alert nobody can act on should be deleted** — alert fatigue is worse than no alerts.

**"How do you write a post-mortem?"**
Blameless, timeline-first, with contributing factors rather than a single "root cause" (complex failures rarely have one). Every action item gets an owner and a date. And the test of a good post-mortem: **would the fix have prevented this, and would the new alert have caught it earlier?** If neither, you haven't finished.

**"What's the worst production incident you've caused?"**
Have a real one ready. The structure that works: what happened, what you did, **what you learned mechanically** (not "I'll be more careful" — a specific mechanism you now use), and what you changed so it can't recur. Owning a mistake with a mechanism-level lesson is a strong signal; claiming you've never caused one is not.

**"How do you prevent this class of bug generally?"**
The best question they can ask, and the answer is always the same shape: **make it structurally impossible rather than remembered.** Tenancy as a required argument. A permission declaration that fails at startup. `oasdiff` as a build gate. An authz matrix test that fails on uncovered routes. A reconciliation job that alerts. *"I'd rather delete the ability to make the mistake than train people not to."*

---

## 9. Practice protocol

Cover the answers. For each of the 40, say your five beats out loud in under 90 seconds. Score:

```
[ ] Did I contain before diagnosing?
[ ] Did I name the specific telemetry I'd look at?
[ ] Did I distinguish "one client / one endpoint / one instance" from "all"?
[ ] Did I give a systemic prevention, not just a fix?
[ ] Did I name the regression test and the alert?
[ ] Did I mention communication where it mattered?
```

**The ones worth over-rehearsing**, because they're asked most and reward depth: #1 (double charge), #13 (payments down at 3am), #15 (slow dependency cascade), #21 (leaked key), #23 (BOLA finding), #34 (read-after-write), #35 (balances don't reconcile).

---

## The API track is complete

You've covered 32 lessons: foundations, REST design, security, production, beyond-REST, craft, a full project, and interview performance.

**Before moving to TypeScript, do these four things:**

1. **Retake the 20-question self-test** from the [API README](../README.md), cold and timed. Compare with the score you noted before Lesson 01.
2. **Empty your `QUESTIONS.md`.** Every unexplained behaviour you wrote down is a debt; pay it now while the context is fresh.
3. **Build at least Phase 4 of Ledger** ([Lesson 29](../07-project/29-ledger-build-log.md)). If you read all 32 lessons and built nothing, you have recognition, not skill — and interviewers can tell the difference within two follow-up questions.
4. **Do three timed design rounds** from [Lesson 31 §12](31-design-round-playbook.md), out loud. Score under 8/10 and redo them in a week.

Then → **[TypeScript: Lesson 01](../../TypeScript/README.md)**

> The TypeScript track assumes everything you now know about contracts, because **types are contracts enforced at compile time** — the same discipline, at a smaller scale and with a compiler to check it.
