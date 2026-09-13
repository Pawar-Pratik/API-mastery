# Lesson 03 — HTTP in full: methods, status, semantics

> **Why this lesson exists:** HTTP is not a transport you tunnel JSON through — it's a **semantic protocol**, and it already answered most of the questions people re-litigate in API design meetings. Method choice determines whether a retry is safe. Status code choice determines whether a client can recover automatically. Get these wrong and no amount of clean URL design saves you. This is also the densest source of interview questions in the entire track.

**Time:** ~90 minutes · **Prereq:** Lesson 02

---

## 1. The idea in one sentence

> **The method says what you intend; the status code says what happened — and both are promises to intermediaries (browsers, proxies, CDNs, retry libraries) that will act on them whether or not you meant them.**

That last clause is the part people miss. When you pick `GET` you're not just naming an operation — you are telling every cache in the world "this is safe to store" and every retry library "this is safe to repeat." **If your `GET` has side effects, you haven't broken a convention; you've lied to infrastructure.**

---

## 2. The three properties that decide everything

Before any method table, learn these three words precisely. Nine out of ten API interviews ask about at least one, and candidates routinely conflate them.

### Safe
> The request is **read-only in intent**. It does not change resource state that the client is responsible for.

`GET`, `HEAD`, `OPTIONS`, `TRACE` are safe. Consequences of safety:
- Browsers may **prefetch** safe requests speculatively. A link prefetcher hitting `GET /orders/123/cancel` will cancel orders that nobody clicked. This has genuinely happened — Google Web Accelerator famously deleted data by prefetching `GET` delete links in 2005.
- Crawlers, link previewers (Slack unfurling a URL!), and antivirus scanners will call safe endpoints uninvited.

> Logging and analytics counters don't break safety — the *client-visible resource* is unchanged. Safety is about intent, not literal immutability.

### Idempotent
> Making the request **N times has the same effect on server state as making it once**.

| Method | Idempotent? | Why |
|---|---|---|
| `GET`, `HEAD`, `OPTIONS` | ✅ | Safe implies idempotent |
| `PUT` | ✅ | "Make the resource equal to this." Doing it twice: still equal to this |
| `DELETE` | ✅ | "Make it not exist." Doing it twice: still doesn't exist |
| `POST` | ❌ | "Process this." Twice = processed twice = two payments |
| `PATCH` | ❌ **by default** | `{"op":"increment","by":1}` twice adds 2. But a *replacement-style* patch (`{"status":"paid"}`) happens to be idempotent |

**Two clarifications that separate strong answers from weak ones:**

1. **Idempotent ≠ same response.** `DELETE /payments/pi_1` returns `204` the first time and `404` the second. The *state* is identical; the *response* differs. That is still idempotent. Candidates lose points insisting the response must match.
2. **Idempotency is a promise you must actually implement.** HTTP says `PUT` *should* be idempotent; it doesn't make it so. If your `PUT` handler appends to an audit array or increments a version counter in a way clients observe, you've broken the contract that retry libraries rely on.

**Why anyone cares:** every network client, load balancer, proxy and service mesh will **automatically retry** idempotent requests on timeout or 5xx, and will refuse to retry `POST`. That is why the *"the client retried my `POST` and the customer was charged twice"* problem exists at all, and why idempotency keys ([Lesson 18](../04-production/18-reliability-and-idempotency.md)) had to be invented.

### Cacheable
> The response may be stored and reused for a later equivalent request.

| Method | Cacheable | Reality |
|---|---|---|
| `GET`, `HEAD` | ✅ by default | Cached unless you say otherwise |
| `POST` | ⚠️ Technically yes, with explicit freshness headers | Effectively never in practice; most caches won't |
| `PUT`, `DELETE`, `PATCH` | ❌ | Never cached, and they *invalidate* cached entries for that URI |

### The matrix worth memorising

| Method | Safe | Idempotent | Cacheable | Body in request | Body in response |
|---|:---:|:---:|:---:|:---:|:---:|
| `GET` | ✅ | ✅ | ✅ | no* | yes |
| `HEAD` | ✅ | ✅ | ✅ | no | **no** |
| `OPTIONS` | ✅ | ✅ | ❌ | no | optional |
| `POST` | ❌ | ❌ | rarely | yes | yes |
| `PUT` | ❌ | ✅ | ❌ | yes | optional |
| `PATCH` | ❌ | ❌ | ❌ | yes | optional |
| `DELETE` | ❌ | ✅ | ❌ | rarely | optional |

\* A `GET` *may* carry a body per spec, but proxies and servers routinely drop it. **Never design a `GET` with a body.** (This is the real reason "complex search" endpoints end up as `POST /search` — covered in [Lesson 08](../02-rest-design/08-collections-and-pagination.md).)

> **Say this in an interview and you sound senior:** *"Safe, idempotent and cacheable are three different properties. Safe implies idempotent, and safe methods are cacheable by default. `PUT` and `DELETE` are idempotent but not safe. `POST` is none of them, which is exactly why retrying a `POST` needs an idempotency key."*

---

## 3. Every method, with the decisions that matter

### `GET` — retrieve a representation
```http
GET /v1/payments/pi_3Nx8 HTTP/1.1
```
Rules that bite:
- **No side effects.** Ever. Not "small ones."
- **No body.** Put filters in the query string; if they don't fit, see `POST /search`.
- **URL length is bounded in practice** — ~2,000 chars for broad browser/proxy safety (IE's old limit, still the safe number), 8KB typical server default. Long filter lists overflow this.
- Return `200` with a body, or `404`. Not `200 {"error": "not found"}` — see §6.

### `HEAD` — the headers only
Identical to `GET`, but the server **must not** send a body while still reporting the headers `GET` would have. Genuine uses: checking existence, size (`Content-Length`), or freshness (`ETag`, `Last-Modified`) before downloading; health checks; link validators.

Common bug: a framework computes the body then strips it, so `HEAD` costs exactly as much as `GET` and gives you no benefit. Worse: some hand-rolled handlers send a body on `HEAD`, which desynchronises HTTP/1.1 keep-alive connections and produces bizarre cross-request corruption.

### `POST` — process this data
The workhorse, and deliberately the most loosely defined method: *"perform resource-specific processing on the enclosed representation."* Legitimate uses:

1. **Create a subordinate resource** → `201 Created` + `Location` header
2. **Submit data for processing** → `200` with result
3. **Trigger an action that isn't a CRUD verb** → `POST /payments/pi_1/refund`
4. **Complex queries that don't fit in a URL** → `POST /search`
5. **Start a long-running job** → `202 Accepted`

**`POST` is not idempotent, so you must decide how retries behave.** There is no fourth option:
- Accept duplicates (fine for `POST /events` analytics).
- Require an `Idempotency-Key` header (correct for anything with money or notifications).
- Derive a natural uniqueness key and return `409 Conflict` on collision (e.g. `POST /users` with an email that exists).

### `PUT` — replace the resource at this URI

The most misunderstood method in HTTP. **`PUT` does not mean "update".** It means:

> *"Make the resource at this exact URI have this exact representation. Create it if it doesn't exist."*

Three consequences people get wrong:

1. **`PUT` is a full replacement.** Omitted fields must be **removed/reset**, not left alone. If your `PUT` handler only updates the fields present in the body, you've implemented `PATCH` and called it `PUT` — and clients that rely on `PUT` semantics to clear a field will be confused.
2. **The client chooses the URI.** `PUT /v1/customers/cus_alice` is valid even if nothing is there yet — that's an *upsert*, returning `201` if created and `200`/`204` if replaced. This is why `PUT` is right for client-generated IDs and idempotent provisioning (`PUT /webhooks/wh_abc123`), and why `POST /customers` (server assigns the ID) is right when you can't know the ID up front.
3. **`PUT` to a collection means "replace the whole collection"** — almost never what anyone wants. `PUT /payments` would mean *"delete all payments and make this array the new set."* Don't.

```http
PUT /v1/customers/cus_alice HTTP/1.1
Content-Type: application/json
If-Match: "a3f9c1"                ← optimistic concurrency, see Lesson 09

{ "name": "Alice", "email": "alice@x.dev", "phone": null }
```

### `PATCH` — apply a partial modification
The method whose *body format* you must choose, because HTTP doesn't define one. Three real options, covered fully in [Lesson 09](../02-rest-design/09-writes-patch-and-bulk.md):

| Format | Media type | Shape | Verdict |
|---|---|---|---|
| **JSON Merge Patch** (RFC 7386) | `application/merge-patch+json` | `{"name":"New"}`; `null` deletes a key | **The pragmatic default.** What almost everyone actually means by PATCH |
| **JSON Patch** (RFC 6902) | `application/json-patch+json` | `[{"op":"replace","path":"/name","value":"New"}]` | Powerful (array ops, `test` for concurrency), verbose, harder for clients |
| **Ad-hoc "just send the changed fields"** | `application/json` | Same as merge patch, undocumented | What 90% of APIs ship. Works, but ambiguous about `null` and nested objects |

The ambiguity that causes real bugs: in ad-hoc PATCH, does `{"metadata": {"a":1}}` **replace** the whole `metadata` object or **merge** into it? Merge Patch says merge (recursively). Most hand-rolled implementations replace. **Document which one you do** — this is a layer-2 semantic decision (Lesson 01) and it will otherwise be discovered by a customer losing data.

> **`PATCH` is not idempotent by default, but you can make it so.** Use `If-Match` with an ETag: the second identical request fails with `412 Precondition Failed` instead of applying twice. That's the answer to *"how do you make PATCH idempotent?"*, and very few candidates have it.

### `DELETE` — remove the resource
Return `204 No Content` (nothing useful to say) or `200` with a body (if you return the deleted object or a job handle).

The interview favourite: **`DELETE` on something already deleted — 404 or 204?**

Both are defensible; what matters is that you can argue it:
- **`404`** — literally accurate: there is no such resource now.
- **`204`** — honours idempotency in spirit: the desired end state holds, so the client's retry succeeded. **This is what I'd choose for an API with automatic retries**, because a retry after a lost `204` shouldn't surface as an error to the caller.

Say: *"I'd return 204 for both, because DELETE is idempotent and a retried delete shouldn't look like a failure. If the client genuinely needs to distinguish, that's what a preceding GET or an `If-Match` is for."* Then note the real-world split — Stripe returns `404`, Kubernetes returns `404`, many internal APIs return `204`. Knowing that it's contested is itself the senior signal.

Also: **soft delete is a semantic decision.** If `DELETE` merely sets `deleted_at`, then a subsequent `GET` returning `404` while the row still exists is fine — but the resource must disappear from list endpoints too, or clients will see ghosts.

### `OPTIONS` — capabilities
Mostly CORS preflight (Lesson 02). Should also advertise `Allow: GET, POST, PATCH` and must be answerable without authentication.

### The two you'll only meet in interviews
- **`TRACE`** — echoes the request; **disable it**. It enabled the Cross-Site Tracing attack for reading `HttpOnly` cookies.
- **`CONNECT`** — establishes a tunnel; used by forward proxies for HTTPS, not by your API.

---

## 4. Status codes: the ones that matter, and how to choose

There are ~60 codes. You need about 20 fluently. **The class tells the client what to do; the code tells them why.**

| Class | Meaning | Client's automatic behaviour |
|---|---|---|
| **1xx** | Informational | Rare (`100 Continue`, `101 Switching Protocols` for WebSocket) |
| **2xx** | Success | Proceed |
| **3xx** | Redirection | Follow (usually automatically) |
| **4xx** | **Client** error — *"you sent something wrong"* | **Do not retry unchanged.** Fix the request |
| **5xx** | **Server** error — *"my fault"* | **Retry is reasonable** (with backoff, if idempotent) |

> **The 4xx/5xx line is the single most consequential decision in your error handling**, because retry logic branches on it. Returning `500` for a validation failure means clients hammer you with a request that can never succeed. Returning `400` for a transient database outage means clients give up on a request that would have worked in 2 seconds. Both are real, common, and expensive.

### 2xx — success

| Code | Use it when | Notes |
|---|---|---|
| **200 OK** | Generic success with a body | The default for `GET`, and for `POST` that returns a result |
| **201 Created** | A resource was created | **Must** include `Location: /v1/payments/pi_3Nx8`. Body should be the created object (saves the client a round trip) |
| **202 Accepted** | Accepted for **async** processing; not done yet | Return a way to check: `Location: /v1/jobs/job_1` or a status field. The honest answer to "the operation takes 4 minutes" |
| **204 No Content** | Success, deliberately **no body** | `DELETE`, and `PUT`/`PATCH` when the client already knows the state. **Must not** have a body — some clients error if you send one |
| **206 Partial Content** | Range request | Video streaming, resumable downloads |
| **207 Multi-Status** | Per-item results in a bulk operation | WebDAV-origin; used by real bulk APIs ([Lesson 09](../02-rest-design/09-writes-patch-and-bulk.md)) |

### 3xx — redirection

| Code | Meaning | The trap |
|---|---|---|
| **301 Moved Permanently** | Permanent | **Browsers cache this aggressively and near-permanently.** A wrong 301 is very hard to undo. Never use it while experimenting |
| **302 Found** | Temporary, method may change | Historically ambiguous; clients rewrite `POST`→`GET` |
| **303 See Other** | "Go `GET` this instead" | The correct redirect after a `POST` (the POST-Redirect-GET pattern) |
| **307 Temporary Redirect** | Temporary, **method preserved** | Use this instead of 302 when the method matters |
| **308 Permanent Redirect** | Permanent, method preserved | The modern 301 |
| **304 Not Modified** | Your cached copy is still good | Not really a redirect. **The single biggest bandwidth win an API can offer** ([Lesson 17](../04-production/17-caching-and-performance.md)) |

### 4xx — the client's fault (this is where design skill shows)

| Code | Precise meaning | Choose it when |
|---|---|---|
| **400 Bad Request** | Malformed — the server **cannot** parse or the request is structurally invalid | Broken JSON, missing required field, wrong type, unparseable query param |
| **401 Unauthorized** | **Unauthenticated** (misnamed in the spec, forever) | No credentials, expired/invalid token. **Must** send `WWW-Authenticate`. Client action: log in / refresh |
| **403 Forbidden** | Authenticated, but **not allowed** | Valid token, insufficient permission. Client action: nothing — don't retry, don't re-auth |
| **404 Not Found** | No such resource | Also used deliberately to hide existence from unauthorised callers (see below) |
| **405 Method Not Allowed** | Wrong method for an existing URI | **Must** include `Allow: GET, POST` |
| **406 Not Acceptable** | Can't produce the `Accept`ed type | Client asked for XML, you only do JSON |
| **408 Request Timeout** | Client was too slow sending | Rare; usually the LB |
| **409 Conflict** | The request conflicts with **current state** | Duplicate email on create; illegal state transition (`refund` a payment that's already refunded); optimistic-lock version mismatch |
| **410 Gone** | Existed, permanently removed | Better than 404 for hard-deleted resources and retired API versions — it tells clients to stop asking |
| **412 Precondition Failed** | `If-Match`/`If-Unmodified-Since` failed | Optimistic concurrency: someone else changed it first |
| **413 Content Too Large** | Body exceeds your limit | Pair with a documented max size |
| **415 Unsupported Media Type** | Body's `Content-Type` isn't supported | Client sent XML or `text/plain` to a JSON-only endpoint. **Very commonly returned as 400 by mistake** |
| **422 Unprocessable Content** | **Syntactically valid, semantically wrong** | Well-formed JSON, correct types, but `amount: -500`, or `end_date` before `start_date`, or a currency you don't support |
| **428 Precondition Required** | You require `If-Match` and it's absent | Forces safe concurrency on critical updates |
| **429 Too Many Requests** | Rate limited | **Must** include `Retry-After`. Client action: back off |
| **451** | Legal reasons | Real, and occasionally used |

**The 400 vs 422 question comes up constantly.** The honest state of the world:
- **The clean distinction:** 400 = *I can't understand you*; 422 = *I understood you and you're wrong*.
- **Reality:** many major APIs (including Stripe) use `400` for everything client-side, because the distinction buys clients little and 422 was historically a WebDAV code. GitHub uses `422` for validation errors extensively.
- **What to say:** *"I'd pick one and apply it consistently, and I'd care far more about the machine-readable error body than the code — because clients branch on `error.code`, not on 400-vs-422. If the team has no convention, I'd use 400 for parse/shape failures and 422 for business-rule failures, and document it."* That answer beats dogma either way.

**The 403 vs 404 question is a security question.** If a merchant requests another merchant's payment:
- `403` tells them *"that exists and isn't yours"* — leaking existence, and letting an attacker enumerate valid IDs.
- `404` tells them nothing.

**Rule: use 404 when the existence of the resource is itself sensitive; 403 when the caller legitimately knows it exists but lacks permission for this action.** GitHub does exactly this — private repos 404 for non-members. This is an excellent thing to volunteer in an interview because it shows you think about information leakage, not just correctness.

### 5xx — your fault

| Code | Meaning | Cause you'll actually see |
|---|---|---|
| **500 Internal Server Error** | Unhandled exception | Your bug. **Never leak a stack trace** — it's a real disclosure vulnerability |
| **501 Not Implemented** | Method not supported at all | Rare |
| **502 Bad Gateway** | Proxy got an invalid/no response from upstream | App crashed, wasn't listening, or returned garbage. Classic LB↔app idle-timeout mismatch (Lesson 02) |
| **503 Service Unavailable** | Temporarily down/overloaded — **on purpose** | Maintenance, or **deliberate load shedding**. Include `Retry-After`. The *correct* code when your queue is full |
| **504 Gateway Timeout** | Upstream took too long | Your DB or a downstream is slow. Means "the gateway gave up" — the work may still be running |
| **507 / 508** | Storage / loop | Rare |

> **The load-shedding insight worth stating in interviews:** when you're overloaded, returning `503` with `Retry-After` **fast** is better than serving everyone slowly. A queue that grows without bound converts a capacity problem into a total outage — every request times out and every retry adds load. Shedding early is how you stay partially available. This is the reasoning behind circuit breakers ([Lesson 18](../04-production/18-reliability-and-idempotency.md)).

---

## 5. The decision tables (print these)

**Choosing a method**

```
Is it read-only?                         → GET (HEAD if you only need headers)
Creating, server assigns the ID?         → POST /collection        → 201 + Location
Creating/replacing, client knows the ID? → PUT /collection/{id}    → 201 or 200/204
Full replacement of an existing thing?   → PUT
Partial change?                          → PATCH (+ If-Match to make it idempotent)
Removing?                                → DELETE                  → 204
An action that isn't CRUD?               → POST /resource/{id}/action
A query too complex for a URL?           → POST /resource/search   (and document why)
Takes longer than a request should wait? → POST → 202 + job resource, or webhook
```

**Choosing a status code for a failure**

```
Could I even parse it?              no  → 400
Right Content-Type?                 no  → 415
Are they authenticated?             no  → 401 + WWW-Authenticate
Are they allowed?                   no  → 403 (or 404 if existence is sensitive)
Does the URI exist?                 no  → 404 (410 if it's permanently gone)
Right method for this URI?          no  → 405 + Allow
Values sane / business rules ok?    no  → 422 (or 400, consistently)
Conflicts with current state?       yes → 409
Precondition (If-Match) failed?     yes → 412   (missing but required → 428)
Over their rate limit?              yes → 429 + Retry-After
Body too big?                       yes → 413
My fault, unexpected?               yes → 500
My fault, deliberate/overloaded?    yes → 503 + Retry-After
Downstream too slow?                yes → 504
```

---

## 6. Production rules

| Rule | Why |
|---|---|
| **Never return 200 with an error in the body** | Every client, proxy, retry library and dashboard branches on status. `200 {"success":false}` makes your error rate invisible in monitoring and forces every client to parse bodies to detect failure. This is the most common API design sin in the wild |
| **Never 500 a validation error** | Clients will retry forever a request that can never succeed |
| **Never 400 a transient failure** | Clients will give up on something that would work in 2s |
| **`201` must carry `Location`** | Otherwise the client can't address what it just made |
| **`405` must carry `Allow`; `401` must carry `WWW-Authenticate`; `429`/`503` must carry `Retry-After`** | These are how automatic clients recover without a human reading docs |
| **`204` must have no body** | Some HTTP clients throw on a body with 204 |
| **Don't invent codes (e.g. `499`, `450`)** | Intermediaries treat unknown 4xx as 400 and unknown 5xx as 500; you gain nothing and break tooling. Put your specificity in the error body's `code` field |
| **Make `GET` genuinely side-effect free** | Prefetchers, crawlers, Slack link unfurling and antivirus scanners will call it |
| **Keep `PUT` a true full replacement, or don't call it `PUT`** | Half-`PUT` breaks the one promise clients rely on |
| **Log the status code *and* an app-level error code together** | `429` alone doesn't tell you which limit; `403` alone doesn't tell you which rule |

> **Spring equivalent:** `ResponseEntity.created(uri)`, `@ResponseStatus`, and `ProblemDetail` in Spring 6 / Boot 3 — see `spring boot/05-web/16-api-design-dto-validation-errors.md`. Same semantics, different syntax.

---

## 7. Interview traps

**Q1. "Difference between `PUT` and `PATCH`?"**
Weak: *"PUT replaces, PATCH updates."* Strong:
> *"`PUT` replaces the resource at that URI entirely — omitted fields must be reset, and it can create the resource, so it's an upsert with a client-chosen ID. `PATCH` applies a partial modification, and HTTP doesn't define the body format, so you pick one — JSON Merge Patch is the pragmatic default. `PUT` is idempotent; `PATCH` isn't necessarily, though you can make it idempotent with `If-Match`."*

**Q2. "Difference between `POST` and `PUT`?"**
The pivot is **who chooses the URI**, not "create vs update." `POST /payments` — server assigns the ID, not idempotent. `PUT /payments/pi_123` — client names it, idempotent, upsert.

**Q3. "Is `DELETE` idempotent even though the second call returns 404?"**
Yes. Idempotency is about **effect on server state**, not response equality. Have the reasoning for your 204-vs-404 choice ready — they *will* follow up.

**Q4. "Why can't the client just retry a `POST`?"**
Because `POST` is "process this," and processing twice means two payments, two emails, two shipments. The fix is to make the operation idempotent at the application layer with an `Idempotency-Key`, which lets the server recognise and de-duplicate the retry. This is the highest-yield API answer in existence — you should be able to design the storage for it on a whiteboard ([Lesson 18](../04-production/18-reliability-and-idempotency.md)).

**Q5. "401 vs 403?"**
401 = I don't know who you are (retry with credentials). 403 = I know who you are and the answer is no (retrying is pointless). Bonus: *"401 must include `WWW-Authenticate`, and the names are historically backwards — 401 is really 'unauthenticated'."*

**Q6. "Client sends a valid JSON body with `amount: -100`. Status?"**
`422` (or `400` if that's your house convention) — and say *why*: it parsed fine, so it's semantic, not syntactic. Then add the part that matters: the response body needs a machine-readable `code` and a `field` pointer so the client can highlight the input. **Never** `500`.

**Q7. "What's `202 Accepted` for, and how does the client learn the outcome?"**
Work that outlives a request. Three options, and you should name the trade-offs: (a) return `202` + a job resource the client polls, (b) `202` + a webhook callback, (c) `202` + an SSE/WebSocket stream. Polling is simplest and works through firewalls; webhooks are efficient but require the client to run an endpoint and verify signatures. ([Lesson 23](../05-beyond-rest/23-realtime-and-webhooks.md).)

**Q8. "Is `POST` cacheable?"**
Technically yes with explicit `Cache-Control`/`Expires`, and a `POST` response can even be cached against the request URI — but in practice essentially nothing does it, and you shouldn't rely on it. The useful part of the answer: *"`POST`, `PUT`, `PATCH` and `DELETE` **invalidate** cached entries for that URI, which is how HTTP caches stay coherent."*

**Q9. "Return 500 or 503 when your database is down?"**
`503` with `Retry-After`, because it's transient and you want clients to back off and retry rather than treat it as a permanent bug. `500` implies "unexpected defect." And if you're deliberately shedding load, `503` is the honest signal.

**Q10. "What status for 'this email is already registered'?"**
`409 Conflict` — it conflicts with current state. (`422` is defensible if you treat it as a validation rule.) The trap is `400`, which tells the client nothing actionable. Follow-up they'll ask: *"doesn't that leak which emails are registered?"* — yes, and for a signup endpoint that's a real enumeration concern; the mitigation is to send a "check your email" response either way and rate-limit the endpoint. Volunteering that is a strong signal.

**Q11. "Your `GET /reports` takes 90 seconds. What's wrong and how do you fix it?"**
It's not a `GET` problem, it's an interaction-pattern problem: `POST /reports` → `202` + `Location: /reports/rep_1` → client polls or gets a webhook → `GET /reports/rep_1/download` when ready. Also mention that a 90s request will be killed by an LB, retried by clients, and multiply the load.

---

## 8. Build & break

### Build — a status-code-correct resource
Write `scratch/payments-route.ts`. Focus is entirely on methods and statuses.

```ts
import { Router } from "express";
const r = Router();

// GET — safe, idempotent, cacheable
r.get("/:id", async (req, res) => {
  const p = await repo.find(req.params.id, req.merchantId);
  if (!p) return res.status(404).json(problem("payment_not_found", "No such payment"));
  res.set("ETag", etagOf(p));                       // enables 304 → Lesson 17
  if (req.headers["if-none-match"] === etagOf(p)) return res.status(304).end();
  res.json(toDto(p));
});

// POST — create; not idempotent, so demand a key for money
r.post("/", async (req, res) => {
  const key = req.header("Idempotency-Key");
  if (!key) return res.status(400).json(problem("idempotency_key_required",
      "Provide an Idempotency-Key header for all payment creation requests"));

  const parsed = CreatePayment.safeParse(req.body);
  if (!parsed.success) return res.status(422).json(validationProblem(parsed.error));

  const { payment, replayed } = await service.create(parsed.data, key, req.merchantId);
  res.status(replayed ? 200 : 201)
     .set("Location", `/v1/payments/${payment.id}`)
     .json(toDto(payment));
});

// PATCH — partial, made idempotent by requiring If-Match
r.patch("/:id", async (req, res) => {
  const ifMatch = req.header("If-Match");
  if (!ifMatch) return res.status(428).json(problem("precondition_required",
      "Send If-Match with the ETag you last read"));

  const result = await service.patchMetadata(req.params.id, req.body, ifMatch, req.merchantId);
  if (result.kind === "not_found")   return res.status(404).json(problem("payment_not_found", "..."));
  if (result.kind === "stale")       return res.status(412).json(problem("stale_etag", "Re-read and retry"));
  if (result.kind === "bad_state")   return res.status(409).json(problem("invalid_state",
      `Cannot modify a payment in state ${result.state}`));
  res.json(toDto(result.payment));
});

// The action that isn't CRUD → POST on a sub-path
r.post("/:id/refund", async (req, res) => {
  const out = await service.refund(req.params.id, req.body, req.header("Idempotency-Key"), req.merchantId);
  if (out.kind === "already_refunded") return res.status(409).json(problem("already_refunded", "..."));
  res.status(202).set("Location", `/v1/refunds/${out.refundId}`).json(toDto(out));
});

// DELETE — idempotent: 204 whether or not it existed
r.delete("/:id", async (req, res) => {
  await service.cancel(req.params.id, req.merchantId);   // no-op if already gone
  res.status(204).end();
});

// Wrong method on a real URI
r.all("/:id", (_req, res) =>
  res.status(405).set("Allow", "GET, PATCH, DELETE").end());

export default r;
```

Now justify, in writing, every status in that file. If you can't justify one, that's a `QUESTIONS.md` entry.

### Break — six experiments with `httpbin` / `httpstat.us`
```bash
# 1. Watch a 301 get cached by your browser (then try to un-cache it). Do it in a throwaway app, never prod.
curl -i https://httpstat.us/301

# 2. HEAD really has no body — but the headers match GET
curl -I  https://api.github.com/users/torvalds
curl -sI https://api.github.com/users/torvalds | grep -i content-length

# 3. 304 in action: grab the ETag, send it back, observe zero bytes of body
ETAG=$(curl -sI https://api.github.com/users/torvalds | tr -d '\r' | grep -i '^etag:' | cut -d' ' -f2)
curl -i -H "If-None-Match: $ETAG" https://api.github.com/users/torvalds

# 4. Wrong method → does the API send Allow?
curl -i -X DELETE https://api.github.com/users/torvalds

# 5. 429 with Retry-After (hammer an unauthenticated GitHub endpoint ~60×)
for i in $(seq 1 70); do curl -s -o NUL -w "%{http_code} " https://api.github.com/users/torvalds; done

# 6. Prove `fetch` doesn't throw on 500
node -e "fetch('https://httpstat.us/500').then(r=>console.log('no throw, status', r.status))"
```

### Break your own code — three deliberate lies
1. Make a `GET` endpoint that increments a counter. Load the URL in Slack or paste it in a chat with link previews. Watch the counter jump without anyone clicking. That's why safety is a *promise*, not a preference.
2. Make `PUT` behave like `PATCH`. Then write a client that clears a field by omitting it. Watch the field survive. That's a broken contract, silently.
3. Return `200 {"error":"insufficient_funds"}`. Now point a retry library at it and try to build a dashboard of your error rate. Feel the pain — then fix it to `422`.

### Explain out loud (90 seconds)
1. Safe vs idempotent vs cacheable, with a method that is one but not another.
2. `PUT` vs `POST` vs `PATCH`, decided by URI ownership and totality.
3. Why the 4xx/5xx boundary changes client behaviour.
4. Your 404-vs-403 rule, and the security reason behind it.

---

## What's next

You know the verbs and the verdicts. Next: everything *around* the body — the headers that negotiate content, prove identity, control caches and frame the message — plus a precise definition of "payload", which you asked about specifically and which almost nobody can state exactly.

Next → **[Lesson 04: Headers & payloads — the anatomy of a message](04-headers-and-payloads.md)**
