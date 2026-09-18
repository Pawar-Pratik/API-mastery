# Lesson 31 — API design round playbook

> **Why this lesson exists:** the API design round is 45 minutes, an open prompt (*"design the API for a ride-hailing app"*), and a blank whiteboard. Candidates who know everything in Modules 1–6 still fail it — because they start listing endpoints in minute two, never ask about scale, and run out of time before reaching the parts that demonstrate seniority. This is a **repeatable framework with a clock**, plus three fully worked examples.

**Time:** ~75 minutes to learn, then practise · **Prereq:** Modules 1–6

---

## 1. The idea in one sentence

> **A design round is not testing whether you can name endpoints — it's testing whether you can extract requirements, make explicit trade-offs, and volunteer the hard parts before being asked.**

The single most common failure is **starting with endpoints**. The second is **never mentioning failure modes**. Both are timing problems, and the framework fixes both.

---

## 2. The framework: 45 minutes, six phases

```
 0–5   min  ① CLARIFY      — ask, don't assume. Write the answers down
 5–10  min  ② MODEL        — nouns, lifecycles, state machines
10–20  min  ③ SURFACE      — resources, URLs, methods, one payload in full
20–28  min  ④ HARD PARTS   — idempotency, concurrency, pagination, auth ← the differentiator
28–36  min  ⑤ OPERATE      — errors, limits, caching, failure modes, observability
36–42  min  ⑥ EVOLVE       — versioning, what you'd do differently, what you'd build next
42–45  min     Questions for them
```

**Announce the plan in minute one.** Literally say: *"Let me spend five minutes on requirements, then the domain model, then the API surface, then I want to make sure we get to idempotency, concurrency and failure modes because that's where the interesting decisions are. Stop me if you'd rather go deeper anywhere."*

This does three things: it shows you have a method, it tells them you know where the depth is, and it gets you permission to control the clock. Interviewers almost always say yes and then let you run.

---

## 3. Phase ① — Clarify (5 min, and never skip it)

**Ask these eight. Write the answers on the board where you can see them** — you'll refer back, and it looks like requirements gathering rather than stalling.

| # | Question | Why it changes the design |
|---|---|---|
| 1 | **Who are the clients?** Browser, mobile, third-party servers, internal? | Decides style (REST/gRPC/GraphQL) and auth ([L24](../05-beyond-rest/24-choosing-an-api-style.md)) |
| 2 | **Public or internal?** How many integrators? | Decides versioning strictness and doc investment |
| 3 | **Read/write ratio and scale?** Requests/sec, rows per tenant in 2 years | Decides pagination, caching, indexing ([L08](../02-rest-design/08-collections-and-pagination.md)) |
| 4 | **Which operations have side effects that must not repeat?** | Decides where idempotency keys go ([L18](../04-production/18-reliability-and-idempotency.md)) |
| 5 | **Is anything slower than ~2 seconds?** | Decides 202 + job resource vs synchronous ([L03](../01-foundations/03-http-methods-and-status.md)) |
| 6 | **Multi-tenant? Who owns what?** | Decides the whole authorization model ([L15](../03-security/15-authorization-and-multitenancy.md)) |
| 7 | **Does anyone need to be *told* when something happens?** | Decides webhooks/SSE/WebSocket ([L23](../05-beyond-rest/23-realtime-and-webhooks.md)) |
| 8 | **What's explicitly out of scope?** | Protects your 45 minutes |

**Then state your assumptions out loud** for anything they didn't answer: *"I'll assume mobile plus third-party server clients, roughly 10:1 read/write, a few million rows per tenant, and that payments must not double-charge. Shout if any of that's wrong."*

> **Why this phase wins points:** a real API design starts with requirements, and interviewers are watching for whether you'll invent them. Candidates who jump to endpoints have skipped the actual job. Two of the eight questions (#4 and #6) also let you plant the flag for later depth.

---

## 4. Phase ② — Model (5 min)

On the board:
1. **The noun list**, from how the domain speaks.
2. **The relationships** (1–many, many–many).
3. **The state machine** for the central entity — draw it.

**The state machine is the highest-value five minutes in the whole round.** It shows you think about legal transitions, not just CRUD, and it hands you the 409-vs-404 distinction, the concurrency discussion, and the event list for free.

```
ride:  requested → matched → driver_arriving → in_progress → completed
              ↘ cancelled_by_rider     ↘ cancelled_by_driver      ↘ no_show
```

Then say the sentence that pays off later: *"I'd enforce every transition in the `WHERE` clause of the update, so an illegal transition is a zero-row update returning 409 rather than a race in application code."* ([L09](../02-rest-design/09-writes-patch-and-bulk.md))

---

## 5. Phase ③ — Surface (10 min)

Write the URL list. Then **one payload in full** — request and response — and stop. Don't write all of them; nobody needs your fourth response body, and it burns clock you need for phase ④.

```
GET    /v1/rides                      list + filter + cursor paginate
POST   /v1/rides                      request a ride   (Idempotency-Key)  → 201
GET    /v1/rides/{id}
POST   /v1/rides/{id}/cancel          → 200 | 409
GET    /v1/rides/{id}/events          audit trail
GET    /v1/drivers/{id}/location      → or SSE stream
POST   /v1/rides/{id}/rating          → 201  (one per ride, so a singleton sub-resource)
GET    /v1/me
```

Narrate the two or three decisions that were actually decisions:
- *"`/cancel` as an action sub-path rather than `PATCH {"status":"cancelled"}`, because cancellation has side effects, carries a reason, and rider-cancel and driver-cancel need different permissions."*
- *"Rating is `POST /rides/{id}/rating` — one per ride, created once, separately permissioned."*
- *"IDs are opaque and prefixed so they're not enumerable and they're identifiable in support tickets."*

Then one payload, showing money/units/time discipline:
```jsonc
// POST /v1/rides   Idempotency-Key: <uuid>
{ "pickup":  { "lat": 19.0760, "lng": 72.8777 },
  "dropoff": { "lat": 19.1136, "lng": 72.8697 },
  "product": "sedan",
  "payment_method": "pm_9s2k" }

// → 201 Created   Location: /v1/rides/rid_01HQ8ZK
{ "object": "ride", "id": "rid_01HQ8ZK", "status": "requested",
  "fare_estimate_minor": 24500, "currency": "inr",     // minor units + currency
  "eta_seconds": 240,                                   // unit in the NAME
  "driver": null,
  "created_at": "2026-03-14T09:30:00Z" }                // ISO 8601, UTC, *_at
```

---

## 6. Phase ④ — Hard parts (8 min) — **this is where the round is won**

Volunteer all five. **Do not wait to be asked.** If you're running out of time, sacrifice phase ③, never this.

**1. Idempotency.** *"`POST /rides` needs an `Idempotency-Key` — a rider double-tapping or a mobile client retrying on a flaky network must not create two rides. The key is scoped by tenant and endpoint, stored with a hash of the request body, claimed atomically with `INSERT … ON CONFLICT`, 24-hour TTL, and a 5xx releases the key so the client can retry."*

**2. Concurrency.** *"Two riders requesting the same driver, or a rider cancelling while the driver accepts. Driver assignment is a conditional update — `WHERE driver_id = $1 AND status = 'available'` — so exactly one request wins and the other gets 409. For rider-editable records I'd use `ETag` + `If-Match` and return 412 on a stale write rather than silently overwriting."*

**3. Pagination.** *"Ride history is cursor-paginated. Offset breaks two ways at scale: it's O(offset) so deep pages get slow, and concurrent inserts cause duplicate and silently skipped rows — which matters for anyone exporting their history. Cursor on `(created_at, id)` with the ID as a tiebreaker, `has_more` instead of a total count."*

**4. Authorization.** *"Multi-tenant with three actor types — rider, driver, support. The tenancy scope comes from the credential only, never from a request field. `GET /rides/{id}` must 404 for a rider who isn't on that ride, not 403, so ride IDs aren't enumerable. And I'd have an actor×endpoint matrix test that fails CI if any route lacks explicit authorization coverage."*

**5. Real-time.** *"Driver location to the rider's phone is one-way and high-frequency — SSE, or a WebSocket if the client also streams. Notifying a third-party partner is a webhook with HMAC signing, jittered backoff and a dead-letter path, plus a pollable event log so they can self-recover after an outage."*

> **Each of those is 60–90 seconds.** Together they're the difference between "knows REST" and "has run this in production." Rehearse them until they're fluent — the fluency itself is signal.

---

## 7. Phase ⑤ — Operate (8 min)

Cover four, briefly:

**Errors:** *"RFC 9457 `problem+json` with a stable `code`, per-field errors for validation, and `request_id` in the body and header. One code per distinct client action — `driver_unavailable` and `payment_declined` need different handling, so different codes."*

**Limits:** *"Token bucket in Redis via a Lua script for atomicity, keyed on the user, layered with per-endpoint and global limits. `RateLimit-*` on every response so clients self-throttle. `POST /rides` gets a tighter limit than reads, because it triggers matching work."*

**Caching:** *"`private, no-cache` + `ETag` on ride reads — 304s make polling cheap. Reference data like vehicle products is `public, max-age`. **Fare quotes I would not cache**, because a stale price is a customer dispute."*

**Failure modes — the part to volunteer:** *"If the matching service is slow, I want a timeout, a bulkhead so it can't consume all workers, a circuit breaker so I fail fast instead of parking a thousand requests, and load shedding with 503 + `Retry-After` rather than an unbounded queue. And retries need full jitter, because synchronised retries after a restart are a self-inflicted DDoS."*

**Observability, in one sentence:** *"`X-Request-Id` on every response, structured logs keyed by it, RED metrics with bounded labels, traces across services — and I'd alert on rides-created-per-minute, because a bad deploy that silently rejects valid requests won't show up as 5xx."*

---

## 8. Phase ⑥ — Evolve (6 min)

**Versioning:** *"`/v1` in the path for visibility and routing. Most changes should be additive, so I'd document that clients must ignore unknown fields and handle unknown enum values — otherwise every new ride status is a breaking change."*

**What you'd do differently / next:** have two ready.
- *"I'd add surge pricing as a separate quote resource rather than a field, so a quote has an identity, an expiry and an audit trail."*
- *"The thing I'd watch is the driver-location endpoint — if riders poll it, that's the traffic that will dominate the system, and it's the first thing I'd move to a push model with a dedicated store."*

**Naming what you'd watch is a senior signal**, because it shows you're thinking about the system after launch, not just at design time.

---

## 9. Three worked examples

### Example A — "Design a file upload API"

**The trap:** describing a `multipart/form-data` endpoint that streams to disk. It works, and it's the junior answer.

**The senior answer:**
```
POST /v1/uploads                        → 201
  { "filename": "invoice.pdf", "content_type": "application/pdf", "size_bytes": 2400000 }
  ← { "id": "upl_1", "upload_url": "https://s3...?X-Amz-Signature=...",
      "expires_at": "...", "method": "PUT" }

Client PUTs the bytes DIRECTLY to storage (never through your API)

POST /v1/uploads/upl_1/complete         → 200  { "status": "processing" }
GET  /v1/uploads/upl_1                  → status: processing | ready | failed
                                          (or a webhook / SSE on completion)
```

Why, and say all of it:
- **Pre-signed URLs** remove upload bandwidth, memory pressure and request timeouts from your app tier entirely.
- **Multipart upload** to storage gives resumability for large files.
- **You validate metadata before issuing the URL** — size, content type, quota — so you never accept bytes you'd reject.
- **Never trust the declared `content_type` or `filename`**: sniff magic bytes after upload, generate your own storage key (path traversal), and serve user content from a **separate origin** with `nosniff` so a polyglot HTML file can't execute on your domain.
- **Virus/content scanning** happens in the `processing` state, which is why `complete` returns 202-ish semantics rather than `ready`.
- **Idempotency:** the `complete` call needs a key, or a retry double-processes.
- **Quota:** per-tenant storage limits, checked at URL issue time.

### Example B — "Design a notification service"

**The trap:** `POST /notifications` and you're done.

**The senior answer** — the interesting parts are all about *not* sending:
```
POST /v1/notifications          (Idempotency-Key)     → 202 + /v1/notifications/{id}
  { "recipient": "usr_1", "template": "payment_receipt",
    "channels": ["email","push"],       // preference-ordered
    "data": { "amount_minor": 4999 },
    "dedupe_key": "receipt:pi_3Nx8",    // ← the important field
    "priority": "normal" }

GET  /v1/notifications/{id}     → per-channel delivery status
POST /v1/preferences            → user opt-outs, quiet hours, channel prefs
GET  /v1/notifications?recipient=usr_1&status=failed
```

The five things to volunteer:
1. **202, always.** Sending is inherently async — you're calling SES, FCM, Twilio.
2. **`dedupe_key`** on top of the idempotency key: idempotency protects against *your* retries; dedupe protects against **two different upstream services** both deciding to send a receipt for the same payment. Different problem, different field. **This distinction is the strongest single point in the answer.**
3. **Preferences and quiet hours are part of the API contract**, and legally required in many jurisdictions — so a notification can legitimately be `suppressed`, which is a status, not a failure.
4. **Per-channel status**, because email can succeed while push fails; the response model must be per-channel, not one status.
5. **Rate limits per *recipient*, not just per caller** — otherwise a buggy service sends a user 400 emails, and that's a real incident. Plus a bulkhead per provider so a Twilio outage doesn't stall email.

### Example C — "Design an API for a food delivery app"

The prompt most likely to be given, because it has three actors and a genuine state machine.

**Clarify:** three clients (customer app, restaurant tablet, driver app) plus partner integrations. Reads dominate for browsing; writes are order placement. Orders must not duplicate. Live tracking required.

**Model:** `restaurant`, `menu`, `item`, `cart`/`draft_order`, `order`, `delivery`, `driver`, `rating`, `payment`.

**The state machine** (draw it — it's the centre of the answer):
```
placed → accepted → preparing → ready_for_pickup → picked_up → delivered
    ↘ rejected_by_restaurant   ↘ cancelled_by_customer (only before preparing)
                                ↘ cancelled_by_platform
```

**Surface, with the decisions narrated:**
```
GET  /v1/restaurants?lat=&lng=&radius=&cuisine=&open_now=   ← cacheable, public
GET  /v1/restaurants/{id}/menu                              ← heavily cached, ETag
POST /v1/orders                    (Idempotency-Key)   → 201
POST /v1/orders/{id}/accept        (restaurant)        → 200 | 409
POST /v1/orders/{id}/cancel        (customer|platform) → 200 | 409
GET  /v1/orders/{id}
GET  /v1/orders/{id}/tracking      → SSE stream of driver position + status
GET  /v1/orders?status=&created[gte]=                  ← cursor paginated
```

**Hard parts, in the order you'd raise them:**
- **Order placement is the money path** → idempotency key mandatory; the customer double-tapping "Place order" on a slow network must not create two orders.
- **Price integrity:** the client must **not** send prices. Send item IDs and quantities; the server computes the total from current menu prices. Otherwise a client can order a ₹2000 meal for ₹1. *"Never let the client tell you what something costs"* is a great line.
- **Menu changes mid-order:** an item goes out of stock between cart and checkout → `409` with a machine-readable list of unavailable items so the app can update the cart. This is the kind of specific error design that stands out.
- **Concurrency:** two drivers accepting the same delivery → conditional update on `status='ready_for_pickup' AND driver_id IS NULL`; one wins, one gets 409.
- **Live tracking:** SSE for the customer (one-way, auto-reconnect, resume via `Last-Event-ID`); WebSocket only if the driver app needs low-latency bidirectional.
- **Caching:** restaurant lists and menus are the read-heavy public path → CDN-cacheable with `ETag`, which is where most of your traffic goes. Orders are `private, no-cache`.
- **Authorization:** three actor types with different permissions on the *same* order — the customer can cancel before `preparing`, the restaurant can accept/reject, the driver can only transition pickup states. **This is the richest authorization model of the three examples**, so spend time here.

---

## 10. What loses points

| Mistake | Fix |
|---|---|
| Starting with endpoints in minute two | Clarify first, always. Announce the plan |
| Not asking about scale | *"How many rows per tenant in two years?"* changes pagination and indexing |
| Never mentioning idempotency | Volunteer it on the first write endpoint you draw |
| Never mentioning failure | *"What happens when this dependency is slow?"* — ask it of yourself, out loud |
| Only the happy path | Every endpoint you narrate: what are its error cases? |
| Silent trade-offs | Say *"I'm choosing X over Y because Z"* — the reasoning is what's scored |
| Endpoint soup | 40 endpoints with no pattern. Show the pattern, not the inventory |
| Vague auth ("we'll use JWT") | Name the client type, the mechanism, and the storage decision |
| Running out of time before the hard parts | The clock is your responsibility. Sacrifice phase ③ |
| Arguing with the interviewer's constraint | *"That's a good constraint — here's what changes if we accept it"* |
| Client-supplied prices, tenant IDs, or statuses | The server owns money, tenancy and the state machine |

---

## 11. Phrases that signal seniority

Steal these. They're compact, specific, and each one implies a body of experience.

- *"Let me write the client code first and see if this design is pleasant to use."*
- *"That operation can be retried, so it needs an idempotency key — here's how I'd store it."*
- *"I'd put the state guard in the `WHERE` clause, so an illegal transition is a zero-row update, not a race."*
- *"I'd return 404 rather than 403 there, so IDs aren't enumerable."*
- *"Tenancy comes from the credential; I'd never accept it in a request field."*
- *"That's a non-breaking change by the spec, but I'd check consumer behaviour before calling it safe."*
- *"I wouldn't cache that — a stale value costs more than the round trip saves."*
- *"I wouldn't guarantee ordering, because guaranteeing it means serial delivery, which means head-of-line blocking."*
- *"Exactly-once doesn't exist over a network. At-least-once plus idempotent processing does."*
- *"I'd alert on the business metric, not just the error rate — a silent failure won't show up as a 5xx."*
- *"The thing I'd watch after launch is X, because that's the traffic that will dominate."*
- *"Here's what would change my mind about this decision."*

That last one is the strongest. **Naming your own falsification condition** is the clearest possible signal that the decision is reasoned rather than remembered.

---

## 12. Practice protocol

Design rounds are a performance skill; reading this lesson is not practice.

**Do this: 8 prompts, 45 minutes each, out loud, on paper, timed.**

1. Ride-hailing (three actors, live tracking)
2. Food delivery (three actors, rich state machine, price integrity)
3. Payments (idempotency, webhooks, ledger) — you've built it, so aim for fluency
4. File upload / document management (pre-signed URLs, scanning)
5. Notifications (dedupe vs idempotency, preferences, per-channel status)
6. A calendar/booking API (double-booking is the concurrency question)
7. A social feed (pagination with a mutable, ranked timeline — genuinely hard)
8. An IoT telemetry ingest API (write-heavy, batching, time-series, backpressure)

**After each one, score yourself:**
```
[ ] Did I clarify before designing?
[ ] Did I draw a state machine?
[ ] Did I volunteer idempotency unprompted?
[ ] Did I volunteer a concurrency scenario?
[ ] Did I cover pagination with a reason?
[ ] Did I name the authorization model and the 404-vs-403 rule?
[ ] Did I describe at least three failure modes?
[ ] Did I state a trade-off with "I'm choosing X over Y because Z"?
[ ] Did I name what I'd do differently / what I'd watch?
[ ] Did I finish on time?
```
**Under 8/10 is a fail.** Redo that prompt in a week — not the next day, so you're recalling rather than reciting.

Prompts 6, 7 and 8 are the ones that will expose gaps, because they break the payments-shaped pattern: booking is purely a concurrency problem, a ranked feed breaks cursor pagination's assumptions, and telemetry ingest inverts the read/write ratio and makes batching and backpressure the whole design.

---

## What's next

The design round is one format. The other is the scenario question — *"it's 3am and payments are failing, what do you do?"* — which tests operational judgement rather than design. Next: 40 of them, with the reasoning.

Next → **[Lesson 32: Scenario gauntlet](32-scenario-gauntlet.md)**
