# Lesson 07 — Resource modelling & URL standards

> **Why this lesson exists:** you asked *"what are the standards used to write an endpoint?"* — this is that lesson, and it's the most visible skill you have. A reviewer forms an opinion of your engineering from your URL list in about ten seconds, before reading a single line of logic. The good news: the rules are finite, defensible, and once internalised you'll never guess again.

**Time:** ~80 minutes · **Prereq:** Lesson 06

---

## 1. The idea in one sentence

> **A good URL set is one a developer can predict: after learning three endpoints they can correctly guess the fourth — and the way you achieve that is by modelling your domain as nouns with identity, then letting HTTP methods supply all the verbs.**

The test is *predictability*, not beauty. `/v1/customers/{id}/payment_methods` is not pretty; it's guessable, which is worth far more.

---

## 2. Finding your resources (the part before any URL)

Resource modelling is domain modelling. Do it before you type a slash.

### Step 1 — List the nouns your users talk about
Listen to how the *domain* speaks, not how your database is shaped:

> *"A **merchant** creates a **payment** from a **customer** using a **payment method**. If something goes wrong they issue a **refund**, and the customer might raise a **dispute**. Money accumulates in a **balance** and gets sent out as a **payout**. Every change writes an **event**, and we tell the merchant via a **webhook endpoint** with a **delivery** per attempt."*

That paragraph contains your entire resource list: `merchants`, `payments`, `customers`, `payment_methods`, `refunds`, `disputes`, `balance`, `payouts`, `events`, `webhook_endpoints`, `deliveries`.

### Step 2 — Ask three questions per noun

| Question | If yes | If no |
|---|---|---|
| **Does it have identity?** Can I point at *this one*? | It's a resource, gets a URI | It's a field or a value object |
| **Does it have a lifecycle?** Is it created, changed, ended? | It's a resource | Probably a computed view |
| **Would someone want to list, filter or link to them?** | It's a resource | It's a sub-structure of its parent |

Applied: an `address` on a customer usually has no independent identity → it's a **field**. But a `payment_method` is created independently, reused across payments, and deleted separately → it's a **resource**.

### Step 3 — Resources are NOT your database tables

The mistake that separates junior from senior design. Three ways they legitimately diverge:

- **One resource, many tables.** A `payment` resource may join `payments`, `payment_attempts`, `card_details` and `fx_rates`. The client sees one coherent thing.
- **One table, many resources.** A `ledger_entries` table might surface as `/balance`, `/payouts` and `/transactions` — three different views for three different questions.
- **A resource with no table at all.** `/v1/balance` is `SUM(entries)`; `/v1/reports/revenue` is a computation. Both are perfectly good resources.

> **The framing to use in a review:** *"resources are the vocabulary of your API's users; tables are the vocabulary of your storage. If they happen to match, that's a coincidence, not a goal."* This is also exactly the DTO argument you already know from `spring boot/05-web/16` — same principle, applied at the URL layer instead of the serialization layer.

---

## 3. The URL rules — the complete, defensible set

### Rule 1 — Nouns in paths, verbs as HTTP methods

```
❌ /getPayments          /createPayment       /deletePaymentById
✅ GET /payments         POST /payments       DELETE /payments/{id}
```

Why it's not just style: the verb-in-URL form means intermediaries can't distinguish reads from writes, so you lose caching, read-vs-write rate limiting, and meaningful metrics — and you'll accumulate `/getPaymentV2`, `/getPaymentByEmail`, `/getPaymentFull` forever with no pattern to guess from.

### Rule 2 — Plural collection names, consistently

```
✅ /payments          /payments/pi_3Nx8
❌ /payment           /payment/pi_3Nx8            ← "is this the collection or one item?"
❌ /payments  and  /customer                       ← the worst option: inconsistent
```

The mental model that makes it obvious: `/payments` is a **collection** (a container), `/payments/{id}` is an **item inside it**. A folder is plural; a file in it is one thing. Filesystems have taught every developer this already.

Irregular plurals: use the correct English plural (`/people`, not `/persons`), and if a word is awkward (`/statuses`, `/taxonomies`) just pick one and be consistent. **Consistency beats correctness here.**

The one legitimate singular: a **singleton resource** where only one exists per context — `/v1/balance`, `/v1/me`, `/v1/settings`, `/v1/account`. There's no collection to list, so plural would be a lie.

### Rule 3 — Lowercase, hyphens between words in paths

```
✅ /payment-methods       ← hyphens (kebab-case)
⚠️ /payment_methods       ← underscores; also common, notably Stripe
❌ /paymentMethods        ← camelCase in a URL
❌ /PaymentMethods        ← PascalCase
```

Reasons: **paths are case-sensitive** (Lesson 02), so mixed case guarantees 404s from typos; hyphens are what search engines and RFC 3986 conventions prefer; and underscores can be visually hidden by link underlining.

**Real-world honesty:** Stripe uses `snake_case` in paths *and* JSON, and it's a fine API. Google's AIPs mandate `camelCase` JSON with `kebab-case` paths. **Pick one convention for paths and one for JSON fields, document both, never mix.** In a review, "inconsistent casing" is the finding; "you chose hyphens" is not.

### Rule 4 — Hierarchy expresses containment, not every relationship

```
✅ /customers/cus_9s2k/payment-methods       ← payment methods BELONG to that customer
✅ /payments/pi_3Nx8/refunds                 ← refunds only exist for a payment
❌ /customers/cus_9s2k/payments/pi_3Nx8/refunds/re_9k2/disputes/dp_1
```

**Nesting rule: at most one level of nesting, two in exceptional cases.** Deeper is a smell, for concrete reasons:
- The URL becomes unwieldy and error-prone.
- You must validate the *whole* chain (does this refund actually belong to this payment which belongs to this customer?) — and if you forget, you've built an authorization bug.
- The child usually has a globally unique ID anyway, making the parents pure decoration.

**The standard pattern**, and the one Stripe uses:

```
POST /customers/cus_9s2k/payment-methods    ← nested for CREATE (the parent is context)
GET  /payment-methods/pm_1                   ← flat for READ (the ID is sufficient)
GET  /payment-methods?customer=cus_9s2k      ← query param for FILTERED LIST
```

This gives you the discoverability of nesting where it helps (creation, listing in context) and the simplicity of flat URLs where nesting adds nothing. **Being able to explain this pattern is a strong design-review signal.**

> When you *do* nest, still validate the relationship. `GET /payments/pi_1/refunds/re_9` where `re_9` belongs to `pi_2` must be a `404`, not a successful read of `re_9`. Skipping that check is a real BOLA variant ([Lesson 15](../03-security/15-authorization-and-multitenancy.md)).

### Rule 5 — Path for identity, query string for everything else

| Belongs in the **path** | Belongs in the **query string** |
|---|---|
| What resource this is | Filtering: `?status=succeeded` |
| Hierarchy/ownership | Sorting: `?sort=-created_at` |
| Anything required to identify the thing | Pagination: `?limit=20&cursor=...` |
| | Field selection: `?fields=id,amount` |
| | Expansion: `?expand=customer` |
| | Anything **optional** |

The rule that decides it: **if removing it changes *which* resource you're talking about, it's a path segment. If it changes *how much* or *which subset*, it's a query parameter.**

```
✅ GET /payments?status=succeeded&created_after=2026-01-01&limit=20
❌ GET /payments/status/succeeded/limit/20        ← invents a fake hierarchy
❌ GET /payments/succeeded                        ← now "succeeded" looks like an ID
```

That last one is the subtle failure: `/payments/succeeded` is indistinguishable from `/payments/{id}`, so your router has to special-case a value — and the day someone gets a payment ID that collides, you have a bug.

### Rule 6 — Actions that aren't CRUD: three legitimate options

Real domains have operations that aren't create/read/update/delete. Refund, capture, cancel, publish, retry, send, archive, merge. You have three options, in order of preference:

**Option A — Model the action's *result* as a resource.** Best when it produces a lasting record.
```http
POST /payments/pi_1/refunds       → 201 + /refunds/re_9k2
POST /payments/pi_1/captures      → 201
POST /webhook-endpoints/we_1/deliveries/wd_5/retries  → 202
```
Why it's best: refunds/captures/retries *are* things — they have IDs, timestamps, amounts, statuses, and you'll want to list and audit them. You've discovered a resource you'd otherwise have missed. **Always check for this option first.**

**Option B — A verb sub-path on the resource.** Best for a pure state transition with no lasting artifact.
```http
POST /payments/pi_1/cancel        → 200
POST /users/u_1/deactivate        → 200
POST /invoices/inv_1/send         → 202
```
Not strictly RESTful, universally accepted, and used by Stripe, GitHub and Kubernetes. Rules: always `POST` (it's not safe or idempotent), verb goes **last**, and namespace it under the resource so it's still noun-scoped.

**Option C — A state field via `PATCH`.** Best when it's genuinely just a field, with no side effects and no separate permission.
```http
PATCH /payments/pi_1  {"status": "cancelled"}
```
The problem: it invites illegal transitions (`{"status": "succeeded"}` — can a client just *declare* success?), can't carry action parameters (a cancellation reason), can't be permissioned separately, and hides side effects. **Use it only for genuinely inert fields**, and validate transitions server-side regardless.

> **The decision rule to memorise:** *"Does the action produce something worth keeping? → make that a resource. Is it a state transition with side effects or its own permission? → verb sub-path. Is it truly just setting a field? → PATCH."*

### Rule 7 — Sensible, opaque, prefixed IDs

```
✅ /payments/pi_3Nx8kQ2mR       ← prefixed, opaque, unguessable
✅ /payments/9c8b7a6d-...        ← UUID; fine, less readable
❌ /payments/1041                ← sequential integer
```

Why **not** sequential integers, in order of severity:
1. **Enumeration.** `/payments/1042` is a guess away. Combined with any authorization gap you have a data breach — and you've made the gap trivially discoverable. This is the most common real-world API breach pattern.
2. **Business-intelligence leak.** A competitor signs up twice a week and diffs your IDs to learn your exact growth rate. Companies have leaked their transaction volume this way.
3. **They can't be generated client-side or offline**, and they collide across shards.

Why **prefixed** (`pi_`, `cus_`, `re_` — Stripe's convention, now widely copied):
- A support ticket saying `pi_3Nx8` is instantly identifiable; `9c8b7a6d` could be anything.
- You can validate the *type* at the boundary and reject `POST /refunds {"payment": "cus_9s2k"}` before it becomes a confusing 404.
- Logs, error messages and Slack conversations become self-describing.
- In TypeScript you can brand them so the compiler catches type mix-ups ([TS Lesson 09](../../TypeScript/README.md)).

Consider **UUIDv7** or ULID if you need sortable-by-time IDs with good index locality (UUIDv4's randomness hurts B-tree inserts at volume). Present them prefixed and treat them as opaque strings to clients.

> **Treat IDs as opaque, always.** Never document their structure, never let clients parse them, never encode meaning a client might depend on. The moment a client parses your ID you can't change the scheme (Hyrum's Law again).

### Rule 8 — Version at the start of the path

```
✅ /v1/payments
❌ /payments/v1
❌ /api/payments?version=1
```
Rationale in [Lesson 11](11-versioning-and-evolution.md); for now: leftmost so routing, gateways and docs can split on it cleanly.

### Rule 9 — No file extensions, no trailing slashes, no verbs anywhere

```
❌ /payments.json      ← that's what Accept is for
❌ /payments/          ← pick one; /payments/ and /payments become two cache entries
❌ /api/v1/getAllPaymentsList   ← every rule broken at once
```
Enforce it: redirect `301`/`308` from the trailing-slash form to the canonical one, or reject it. Just don't silently serve both — you'll double your cache footprint and split your metrics.

### Rule 10 — Keep URLs under ~2,000 characters

Browsers/proxies historically cap around 2KB; servers default to ~8KB headers. Long filter lists overflow this. When they do, that's the signal to move to `POST /payments/search` with a body — [Lesson 08](08-collections-and-pagination.md) covers it, including how to keep it honest.

---

## 4. Reserved characters and encoding (the bugs nobody warns you about)

```
GET /v1/customers?email=alice%2Btest%40x.dev&q=a%26b
```

| Character | Raw meaning in a URL | Must be encoded as |
|---|---|---|
| `+` | **A space** in a query string (legacy form encoding) | `%2B` |
| `&` | Parameter separator | `%26` |
| `=` | Key/value separator | `%3D` |
| `/` | Path separator | `%2F` |
| `?` | Starts the query | `%3F` |
| `#` | Starts the fragment — **everything after is never sent** | `%23` |
| `%` | Escape character itself | `%25` |
| space | — | `%20` (or `+` in a query) |

**The two bugs you will personally hit:**

1. **`alice+test@x.dev`.** Email plus-addressing is extremely common. Unencoded, your server receives `alice test@x.dev` and the lookup fails — a *"my email doesn't work on your site"* bug that is invisible in testing because your test emails have no `+`. Fix: `encodeURIComponent` on the client, and never hand-build query strings.
2. **IDs or slugs containing `/` or `#`.** A path segment with an unencoded `/` silently becomes two segments and routes somewhere else entirely; a `#` truncates the request. This is why you don't put user-controlled text in path segments — use an opaque ID and pass the text in the body or an encoded query param.

Use the platform, not string concatenation:
```ts
// ❌ every bug above, waiting to happen
const url = `/v1/customers?email=${email}&q=${q}`;

// ✅
const url = `/v1/customers?${new URLSearchParams({ email, q })}`;
// and for path segments:
const url2 = `/v1/customers/${encodeURIComponent(id)}`;
```

> **Security note:** URL decoding happens at multiple layers (CDN, proxy, framework), and **double encoding** (`%252F` → `%2F` → `/`) is a classic path-traversal and WAF-bypass technique. Never decode manually, never decode twice, and validate the *decoded* value against an allowlist.

---

## 5. Advanced patterns you'll need

### Field selection (sparse fieldsets)
```
GET /payments/pi_1?fields=id,amount_minor,status
```
Reduces payload; complicates caching (each field combination is a distinct cache entry — set `Vary` appropriately or accept the fragmentation). Use it when clients genuinely differ in needs. When they differ *a lot*, that's the argument for GraphQL.

### Expansion (avoiding N+1 for the client)
```
GET /payments/pi_1?expand=customer,payment_method
```
```json
{ "id": "pi_1", "customer": { "id": "cus_9s2k", "email": "..." } }
```
Without expansion, a payments table showing customer emails needs 1 + N requests — the client-side N+1 problem, and one of the strongest arguments GraphQL has. Stripe's `expand` is the reference implementation: **default to the ID string, expand to the object on request.** Cap expansion depth (Stripe allows 4 levels) or you've built an accidental DoS.

> Design detail that matters: when not expanded, is the field `"customer": "cus_9s2k"` (a string) or `"customer": {"id": "cus_9s2k"}` (an object)? Stripe chose the string, which means the field's *type changes* based on a query param — awkward for typed clients. The alternative, always an object with just `id` populated, is friendlier to TypeScript. Either is defensible; **decide deliberately and document it**, because clients will build types around it.

### Singletons and `/me`
```
GET   /v1/balance          ← one per merchant, no collection
GET   /v1/me               ← the authenticated principal
PATCH /v1/me
```
`/me` is excellent: it removes the need for the client to know its own ID, and it makes authorization trivial (there is no other user to leak). Prefer it over `/users/{myOwnId}`.

### Sub-resources that are really relationships
```
PUT    /v1/payments/pi_1/tags/urgent      ← idempotent: add to a set
DELETE /v1/payments/pi_1/tags/urgent      ← idempotent: remove from a set
GET    /v1/payments/pi_1/tags
```
Membership as a resource. `PUT` twice is harmless, `DELETE` twice is harmless — genuinely idempotent, unlike `POST /tags {"tag":"urgent"}`. This is an elegant pattern worth having in your pocket.

### Batch and bulk
```
POST /v1/payments/batch          → 207 Multi-Status, per-item results
```
Non-RESTful, sometimes unavoidable. [Lesson 09](09-writes-patch-and-bulk.md) handles the error semantics, which are the hard part.

### Search
```
GET  /v1/payments?q=alice                       ← simple search: query param
POST /v1/payments/search  { ...complex... }     ← when filters exceed a URL
```
Document the `POST` as the deliberate exception it is, and note the cost (not cacheable).

---

## 6. The reference URL set for Ledger

This is the target. Note that it's guessable: learn three rows and you can predict the rest.

```
# Payments
GET    /v1/payments                       list + filter + paginate
POST   /v1/payments                       create            → 201 + Location
GET    /v1/payments/{id}                  read
PATCH  /v1/payments/{id}                  partial update (metadata, description)
POST   /v1/payments/{id}/capture          action: state transition
POST   /v1/payments/{id}/cancel           action: state transition
POST   /v1/payments/{id}/refunds          action-as-resource → 201 + /v1/refunds/{id}
GET    /v1/payments/{id}/refunds          refunds of this payment
GET    /v1/payments/{id}/events           audit trail for this payment

# Refunds (flat read, nested create)
GET    /v1/refunds                        ?payment=pi_1&status=succeeded
GET    /v1/refunds/{id}

# Customers and their owned things
GET    /v1/customers
POST   /v1/customers
GET    /v1/customers/{id}
PATCH  /v1/customers/{id}
DELETE /v1/customers/{id}
POST   /v1/customers/{id}/payment-methods    create in context
GET    /v1/payment-methods?customer=cus_1    filtered list, flat
GET    /v1/payment-methods/{id}
DELETE /v1/payment-methods/{id}

# Singletons
GET    /v1/balance
GET    /v1/me
PATCH  /v1/me

# Money out
GET    /v1/payouts
POST   /v1/payouts
GET    /v1/payouts/{id}

# Webhooks: endpoints own deliveries; deliveries own attempts
GET    /v1/webhook-endpoints
POST   /v1/webhook-endpoints
GET    /v1/webhook-endpoints/{id}
PATCH  /v1/webhook-endpoints/{id}
DELETE /v1/webhook-endpoints/{id}
GET    /v1/webhook-endpoints/{id}/deliveries
POST   /v1/webhook-endpoints/{id}/deliveries/{did}/retry

# Platform
GET    /v1/events                         the event log (?type=payment.succeeded)
GET    /v1/events/{id}
POST   /v1/api-keys
DELETE /v1/api-keys/{id}
GET    /healthz                            unversioned, unauthenticated, for probes
GET    /v1/openapi.json                    the spec itself
```

Two deliberate details: `/healthz` is **outside** `/v1` because infrastructure probes shouldn't be versioned with your business API, and the spec is served from the API so tooling can always fetch the live contract.

---

## 7. Production rules

| Rule | Why |
|---|---|
| **Plural collections, one convention, no exceptions** | Predictability is the entire product of URL design |
| **Max one level of nesting** | Deeper means chain validation you'll forget, which is an authz bug |
| **Nest to create, flat to read, query to filter** | Best of both; the industry-standard compromise |
| **Opaque, prefixed, non-sequential IDs** | Blocks enumeration and BI leakage; makes support tickets legible |
| **Verbs only as the last segment, only on `POST`** | Keeps actions noun-scoped and honest about non-idempotency |
| **Never accept a client-supplied field that changes ownership** | `POST /payments {"merchant_id": "..."}` is a tenancy bypass. Ownership comes from the token, never the body |
| **Validate the whole parent chain on nested reads** | Or `/payments/pi_1/refunds/re_9` leaks another payment's refund |
| **Encode with `URLSearchParams`/`encodeURIComponent`, never string concat** | The `+`-in-email bug is real and embarrassing |
| **Choose one of trailing-slash / no-trailing-slash and redirect the other** | Two URLs for one resource splits caches and metrics |
| **`/healthz` unversioned and unauthenticated; no business logic in it** | Probes must work when your DB is down — but see the nuance in [Lesson 20](../04-production/20-observability.md) |
| **Never put PII or secrets in a path or query** | URLs are logged at every hop (Lesson 02) |

> **Spring equivalent:** `@RequestMapping("/v1/payments")` on the class, `@GetMapping("/{id}")` on the method, `@PathVariable` vs `@RequestParam` mapping exactly onto Rule 5. Your `spring boot/05-web/15-controllers-and-mapping.md` is the mechanics; this lesson is the design.

---

## 8. Interview traps

**Q1. "Design the URLs for a blog with posts, comments and authors."**
They're watching for: plural nouns, nesting discipline, and how you handle the comment-under-post relationship.
```
GET  /posts                          POST /posts
GET  /posts/{id}                     PATCH /posts/{id}     DELETE /posts/{id}
POST /posts/{id}/comments            ← nested create
GET  /posts/{id}/comments            ← contextual list
GET  /comments/{id}                  ← flat read
GET  /comments?author={id}           ← filter, not /authors/{id}/comments
POST /posts/{id}/publish             ← action: state transition with side effects
GET  /authors/{id}                   GET /authors/{id}/posts
```
Then volunteer the reasoning: *"I nest for creation because the parent is required context, but read flat because a comment ID is globally unique — that keeps me from validating a chain on every read, which is where authorization bugs hide."*

**Q2. "Should it be `/users/123/orders` or `/orders?user=123`?"**
Both, for different jobs: nested to **create** (the parent scopes it), flat with a filter to **list** (composable with other filters — `?user=123&status=paid&created_after=...` — which nesting can't do). Say that and you've answered better than the question expected.

**Q3. "How do you model 'send an invoice' in REST?"**
Walk the three options from Rule 6 and pick with a reason: `POST /invoices/{id}/send` if sending is a transition, or `POST /invoices/{id}/deliveries` if you need a record of each send attempt (which for email you almost certainly do — bounces, retries, timestamps). **Choosing the resource version and explaining why is the strong answer.**

**Q4. "Why not sequential integer IDs?"**
Enumeration (→ real breaches), business-intelligence leakage, and no offline/sharded generation. Then the follow-up: *"UUIDv4 hurts index locality at scale, so I'd use UUIDv7 or ULID for time-sortable IDs with good insert behaviour, presented with a type prefix."*

**Q5. "`PUT /payments/pi_1/status` — good or bad?"**
Bad, and the reason is the lesson: it exposes a *field* as a resource, which invites clients to declare illegal states, can't carry a reason, and can't be permissioned separately. Use an action sub-path (`/cancel`, `/capture`) so the server owns the state machine.

**Q6. "Path param or query param for filtering?"**
Query. Path segments identify; query parameters filter. And name the concrete failure: `/payments/succeeded` collides with `/payments/{id}`.

**Q7. "How would you handle 40 optional filters that blow past URL length?"**
`POST /payments/search` with a JSON body, documented as a deliberate exception, and mention what you lose (HTTP caching, and it's technically a non-safe method for a read operation). Bonus: mention saved-search resources (`POST /searches` → `GET /searches/{id}/results`) as the fully RESTful alternative that also gives you shareable, cacheable, re-runnable queries.

**Q8. "Client sends `POST /payments` with `merchant_id` in the body. What do you do?"**
**Ignore it and derive the merchant from the authenticated token** — or reject the request. Accepting it is a cross-tenant write: merchant A creates a payment attributed to merchant B. This is a mass-assignment vulnerability and it's the answer they're fishing for ([Lesson 15](../03-security/15-authorization-and-multitenancy.md)).

**Q9. "Is `/v1` in the path RESTful?"**
Strictly, purists say no — a resource's identity shouldn't change because your representation format changed; content negotiation via `Accept` is the "correct" mechanism. Pragmatically everyone uses the path because it's visible, `curl`-able, cacheable, routable, and obvious in logs. Give both halves and state your choice ([Lesson 11](11-versioning-and-evolution.md)).

---

## 9. Build & break

### Build — model a domain from a paragraph
Do this before looking at any answer. Here's the requirement:

> *"Restaurants list menus. A menu has sections, each with dishes. Customers place orders containing dishes with quantities and notes. An order moves through placed → accepted → preparing → ready → delivered, or gets cancelled. Customers can rate a delivered order. Restaurants need daily sales reports."*

Produce: the resource list, the full URL set with methods, and — for each of the three hardest decisions — one sentence of justification. Specifically decide:
1. Are `sections` a resource, or a field of `menu`?
2. How do order **state transitions** work — `PATCH` with a status, or action sub-paths?
3. Is `rating` a resource, a sub-resource of order, or a field on order?

Then check yourself against these positions: (1) sections have no independent lifecycle → nested field of the menu, unless they're reordered/toggled independently; (2) action sub-paths (`/orders/{id}/accept`, `/cancel`) because each has distinct permissions — restaurant vs customer — and side effects; (3) `POST /orders/{id}/rating` as a singleton sub-resource, since one rating per order, created once, independently permissioned.

If you disagreed with any of those but can defend your version, **you're doing this right** — that's what a design review sounds like.

### Build — a router that enforces the rules
```ts
import { Router } from "express";

const v1 = Router();

// Collections and items
v1.get   ("/payments",              listPayments);      // ?status=&limit=&cursor=
v1.post  ("/payments",              createPayment);     // 201 + Location
v1.get   ("/payments/:id",          getPayment);
v1.patch ("/payments/:id",          patchPayment);

// Actions: POST, verb last, noun-scoped
v1.post  ("/payments/:id/capture",  capturePayment);
v1.post  ("/payments/:id/cancel",   cancelPayment);

// Action-as-resource
v1.post  ("/payments/:id/refunds",  createRefund);      // 201 + /v1/refunds/:rid
v1.get   ("/payments/:id/refunds",  listRefundsOfPayment);

// Nested create, flat read, filtered list
v1.post  ("/customers/:id/payment-methods", attachPaymentMethod);
v1.get   ("/payment-methods",       listPaymentMethods); // ?customer=cus_1
v1.get   ("/payment-methods/:id",   getPaymentMethod);

// Singletons
v1.get   ("/balance",               getBalance);
v1.get   ("/me",                    getMe);

// Explicit 405 with Allow, so clients learn instead of guessing
v1.all("/payments/:id", (_req, res) =>
  res.status(405).set("Allow", "GET, PATCH").end());

export default v1;
```

Now add the guard that makes nesting safe:
```ts
/** Nested reads must validate the whole chain, or they leak. */
async function listRefundsOfPayment(req, res) {
  const payment = await repo.findPayment(req.params.id, req.merchantId);  // ← tenancy check
  if (!payment) return res.status(404).json(problem("payment_not_found", "..."));
  const refunds = await repo.refundsOf(payment.id);   // scoped to a verified parent
  res.json({ data: refunds.map(toDto) });
}
```

### Break — three URL bugs to experience
1. **The `+` bug.** Build `GET /customers?email=` and query `alice+test@x.dev` by string-concatenating the URL. Watch the lookup fail. Fix with `URLSearchParams`, and note that your existing tests never caught it.
2. **The path-collision bug.** Add `GET /payments/succeeded` alongside `GET /payments/:id`. Register them in the wrong order and watch one shadow the other. That's why filters aren't path segments.
3. **The nesting-leak bug.** Implement `GET /payments/:pid/refunds/:rid` that looks up `rid` **without** checking it belongs to `pid`. Then fetch another payment's refund through the wrong parent and watch it succeed. Fix it. That's BOLA in miniature, and you just wrote it on purpose so you'll never write it by accident.

### Explain out loud (90 seconds)
1. How you find resources from a requirements paragraph, and why they aren't tables.
2. The nesting rule, and the nest-to-create/flat-to-read pattern.
3. Three ways to model an action, and how you choose.
4. Why sequential IDs are a security issue.

---

## What's next

You can design the URLs. Now the endpoint that's hardest to get right and most likely to be tested: **the list endpoint** — pagination that survives 4 million rows and concurrent writes, filtering that doesn't become SQL injection, and sorting that doesn't table-scan.

Next → **[Lesson 08: Collections — pagination, filtering, sorting, search](08-collections-and-pagination.md)**
