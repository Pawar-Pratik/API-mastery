# Lesson 11 — Versioning, evolution & unknown fields

> **Why this lesson exists:** you asked it directly — *"what if the API gets an extra parameter beyond the designed parameters? Should it work or not?"* That question has **three defensible answers**, and which one is right depends on something most engineers have never articulated. This lesson answers it properly, and then covers the larger problem it belongs to: **how an API changes without breaking the people who depend on it** — the layer-4 concern from Lesson 01, and the one that separates people who've maintained an API from people who've only launched one.

**Time:** ~85 minutes · **Prereq:** Lessons 01, 09, 10

---

## 1. The idea in one sentence

> **You cannot change a published API; you can only add to it, or publish a second one — so the entire discipline is about maximising what counts as "adding" and minimising how often you must "publish a second one."**

---

## 2. What is a breaking change? (be precise — this is the foundation)

A change is **breaking** if a client that worked before could stop working, *without changing its own code*. Everything else is safe. Here's the complete table; memorise the left column.

### Breaking

| Change | Why it breaks |
|---|---|
| **Removing** a field from a response | Client reads `res.customer.email` → `undefined` → crash |
| **Renaming** anything (field, endpoint, param, enum value) | It's a remove + an add |
| **Changing a field's type** | `"amount": 4999` → `"amount": "49.99"`; arithmetic breaks silently |
| **Adding a required request field** | Every existing request now 422s |
| **Adding a new validation constraint** | Previously-valid requests now fail |
| **Removing an endpoint or a method** | 404 / 405 |
| **Changing a status code** for the same condition | Client's `if (res.status === 200)` breaks |
| **Changing an error `code`** | Client's error branch breaks |
| **Changing default behaviour** | e.g. default page size 100 → 20, or default sort order flips |
| **Making a field nullable** that never was | `res.name.toUpperCase()` on null |
| **Narrowing an enum** (removing a value the client sends) | Requests rejected |
| **Changing pagination semantics** | Iteration loops break or silently skip |
| **Tightening rate limits** | Working integrations start 429ing |
| **Changing the meaning of a field without changing its shape** | **The most dangerous of all** — no error, just wrong behaviour |

### Non-breaking (*usually* — see §4)

| Change | Caveat |
|---|---|
| **Adding an optional request field** | Safe |
| **Adding a field to a response** | Safe **iff** clients are tolerant readers |
| **Adding a new endpoint** | Safe |
| **Adding a new optional query parameter** | Safe |
| **Adding an enum value to a *response*** | Breaks clients with exhaustive `switch` — see below |
| **Relaxing a validation rule** | Safe |
| **Adding a new error code** | Safe **iff** clients have a default branch |
| **Performance improvements** | Safe unless someone depended on the timing (Hyrum's Law) |

Two entries deserve expansion because they're where the real subtleties live:

**Adding an enum value is a *response*-side breaking change in disguise.** A client with `switch (payment.status) { case "succeeded": … case "failed": … }` and no `default` will silently do nothing — or throw — when you add `"disputed"`. This is why mature APIs **document the enum as open** ("new values may be added; handle unknown values gracefully") and why TypeScript clients should model it as `"succeeded" | "failed" | (string & {})` rather than a closed union. Google's AIPs mandate an `UNSPECIFIED = 0` enum member for exactly this reason.

**"Changing the meaning without changing the shape" is the worst kind.** If `amount` meant "gross" and now means "net of fees", every client compiles, every test passes, and every financial report is wrong. **There is no technical protection against this** — only discipline: never repurpose a field. Add a new one, deprecate the old one.

---

## 3. Your question: should an API accept an extra parameter?

Here it is properly. There are three positions, and **the right answer differs for requests and responses, and differs again for bodies and query parameters.**

### The principle behind all of it: the Tolerant Reader / Postel's Law

> *"Be conservative in what you send, liberal in what you accept."*

For **responses**, this is settled and non-negotiable:

> **A client MUST ignore fields it does not recognise.**

Why: it's the only thing that makes server evolution possible. If clients broke on new fields, you could never add a field without a version bump, and you'd be shipping `/v47/payments`. Every mature API documents this explicitly, and **strict client-side response validation is an anti-pattern** — this is exactly why you should parse a response by *picking the fields you need* rather than asserting the whole shape:

```ts
// ❌ Breaks the day the server adds a field. You've made yourself fragile on purpose.
const Payment = z.object({ id: z.string(), amount_minor: z.number() }).strict();

// ✅ Tolerant reader: validate what you use, ignore the rest.
const Payment = z.object({ id: z.string(), amount_minor: z.number() });  // strips unknowns by default
// (and if you must keep extras, .passthrough())
```

For **requests**, you have a genuine choice:

### Position A — Reject unknown fields (`400`/`422`). **My default for a body.**

```http
POST /v1/payments
{ "amount_minor": 4999, "currncy": "usd" }        ← typo

→ 422 { "code": "unknown_field",
        "detail": "Unknown field 'currncy'. Did you mean 'currency'?",
        "errors": [{ "field": "currncy", "code": "unknown_field" }] }
```

**Arguments for rejecting:**
1. **Typos become errors instead of silent misbehaviour.** `currncy` ignored means the payment is created in your *default* currency. That's a financial bug delivered as a success. Rejecting turns a silent wrong answer into a loud, immediately-fixed error.
2. **Mass assignment defence in depth.** If your validator ignores unknown fields *and* somewhere downstream a lazy `Object.assign` exists, `{"merchant_id": "...", "status": "succeeded", "balance": 999}` is a live vulnerability. `.strict()` closes the whole class ([Lesson 09](09-writes-patch-and-bulk.md)).
3. **It keeps the contract honest.** If unknown fields are ignored, clients start sending fields hoping they do something, and support gets *"why doesn't `discount` work?"* tickets forever.
4. **You can always relax later.** Going from strict → lenient is non-breaking. Lenient → strict is a breaking change. **Start strict.**

### Position B — Ignore unknown fields silently

**Arguments for ignoring:**
1. It's the tolerant-reader principle applied symmetrically.
2. It lets clients send one superset object to multiple API versions or multiple endpoints.
3. **The round-trip case, which is the strongest argument:** a client does `GET /customers/cus_1`, edits one field, and `PUT`s the whole object back. If you added a read-only field (`created_at`, `livemode`) since they wrote their code, a strict server rejects their own data. **This is a real and common client pattern**, and it's why some APIs must be lenient.

### Position C — Ignore, but tell them

```json
{ "id": "pi_3Nx8", "amount_minor": 4999,
  "warnings": [ { "code": "unknown_field", "field": "currncy",
                  "detail": "Ignored. Did you mean 'currency'?" } ] }
```
Best of both in principle; rarely implemented because clients don't read warnings. Worth knowing as an option — and genuinely useful if you also surface it in a developer dashboard, which is what Stripe effectively does.

### The decision rule

| Situation | Do this | Why |
|---|---|---|
| **Request body** on a create/action endpoint | **Reject** (`.strict()`) | Typos are expensive; mass-assignment defence |
| **Request body** on a `PUT` that clients round-trip | **Ignore** read-only fields you own; reject genuinely unknown ones | Allow their own data back |
| **Query parameters** (filters, sort) | **Reject, always** | A typo'd filter returns the *wrong data set* silently (Lesson 08) — the worst failure mode in the API |
| **Response fields**, as a client | **Always ignore** unknowns | The only thing that makes server evolution possible |
| **Webhook payloads**, as a receiver | **Always ignore** unknowns | Same, and you don't control the sender's release schedule |

> **The full interview answer:**
> *"For responses it's not a choice — clients must ignore unknown fields, or the server can never add one. For requests I default to rejecting unknown fields, because a typo'd `currncy` silently creating a payment in the wrong currency is far worse than a 422, and strictness closes off mass assignment. The important asymmetry is query parameters: an ignored unknown filter returns the wrong result set while looking successful, so those I reject unconditionally. The one case for leniency is a `PUT` where clients round-trip the object they read — there I'd ignore read-only fields I own rather than reject their own data."*

That answer covers all three positions, states a default, and names the exception. It's complete.

---

## 4. Versioning strategies

Once a change genuinely is breaking, you need a version. Five options.

### Option 1 — URI path: `/v1/payments` · **the pragmatic winner**

```
GET /v1/payments
GET /v2/payments
```

| Pros | Cons |
|---|---|
| Immediately visible; `curl`-able; obvious in logs, dashboards and support tickets | Purists: a resource's identity shouldn't change because its representation did |
| Trivial to route (gateway, nginx, load balancer) | Encourages whole-API version bumps rather than granular change |
| Browser-cacheable and CDN-friendly as distinct URLs | Duplicate code paths if handled naively |
| Both versions can run side by side, on different deploys | |

**Used by:** basically everyone — Twitter/X, Stripe (`/v1/` — frozen since 2011, see below), Twilio, Slack.

### Option 2 — Custom header

```http
GET /payments
Ledger-Version: 2026-03-14
```

| Pros | Cons |
|---|---|
| URLs stay clean and permanent | Invisible: you can't paste a URL and get a reproducible result |
| Enables **granular, per-account** versioning | Caching requires `Vary`, and CDNs handle it badly |
| Easy to default (omit → oldest or newest) | Harder to debug, easy to forget in `curl` |

**Used by:** Stripe (`Stripe-Version`), Shopify, Azure (`api-version` query param, same idea).

### Option 3 — Media type (`Accept`) — "the correct one"

```http
GET /payments
Accept: application/vnd.ledger.v2+json
```
Formally the most RESTful — you're negotiating a *representation*, which is exactly what versioning is. **Used by:** GitHub (`application/vnd.github.v3+json`).
**Reality:** verbose, easy to get wrong, poor tooling support, confuses people. GitHub itself moved toward header/date-based versioning for its newer surfaces.

### Option 4 — Query parameter

`GET /payments?version=2`. Easy to add, but it pollutes every URL, mixes protocol with data, and is easy to forget. Azure does it; most don't.

### Option 5 — Date-based version pinning (Stripe's actual model) · **the best one, and the most expensive**

This is worth understanding in detail because it's the reference implementation and a great interview talking point.

- The URL is frozen at `/v1/` forever.
- Every account is **pinned** to the API version current when it signed up (`2019-08-14`, say).
- A caller can override per request with `Stripe-Version: 2026-03-14`.
- Internally, the server has a chain of **version changes** — small, composable transformations. A request from an old version is transformed forward to today's internal shape; the response is transformed back down through every intervening change.

| Pros | Cons |
|---|---|
| **Clients never break**, ever. Code from 2015 still runs | Genuinely expensive to build and maintain |
| Upgrading is opt-in and testable per request | Every breaking change needs a forward + backward transformer, forever |
| Changes are granular, not "v1 → v2 big bang" | Requires exceptional internal discipline and test coverage |

Stripe has said this is one of their highest-leverage engineering investments — it's a large part of why developers trust them. **Don't build it for an internal API.** Do reference it when asked "how would you version an API with 5,000 integrators?", because it's the answer at that scale.

### The recommendation

```
Internal API, coordinated deploys       → no version. Evolve additively; break with a migration plan
Public API, small/medium               → /v1 in the path + strong additive discipline
Public API, many integrators           → /v1 frozen + date-based version header (Stripe model)
Anything                               → make 95% of changes non-breaking, so this matters less
```

**And the meta-point, which is the actual senior insight:** *"versioning is a failure mode, not a feature. Every version you ship is code you maintain forever. The goal is to design so that most changes are additive — which means tolerant readers, open enums, no field repurposing, and optional-by-default request fields. A team that needs `/v4` in three years has an evolution-discipline problem, not a versioning problem."*

---

## 5. Deprecation: how to remove something without breaking trust

You will eventually need to remove things. Do it in-band and on a schedule.

### The HTTP mechanism (RFC 8594 + the Deprecation header draft)

```http
HTTP/1.1 200 OK
Deprecation: Wed, 01 Apr 2026 00:00:00 GMT
Sunset: Tue, 01 Sep 2026 00:00:00 GMT
Link: <https://docs.ledger.dev/migrate/v1-to-v2>; rel="deprecation"; type="text/html"
Warning: 299 - "This endpoint is deprecated; use POST /v2/payments"
```

- **`Deprecation`** — when it became (or becomes) deprecated
- **`Sunset`** — when it will stop working. **This date is a promise; honour it or announce a change**
- **`Link rel="deprecation"`** — where the migration guide is

### The process that actually works

| Phase | Duration | Action |
|---|---|---|
| **1. Announce** | — | Changelog, email to integrators, docs banner, `Deprecation` header live |
| **2. Instrument** | ongoing | **Log every call to the deprecated thing, per client.** You cannot remove what you can't measure |
| **3. Nudge** | months | Dashboard warnings, targeted emails to the specific accounts still calling it |
| **4. Brownout** | 1–2 events | Return `410 Gone` for a scheduled 30–60 minutes. Loudly announced in advance. **This is the single most effective technique in existence** — it converts "I'll do it later" into a real ticket, safely |
| **5. Sunset** | at the date | `410 Gone` with a migration link in the body |

**Timelines by audience:** internal with known consumers, weeks. Partner, a quarter with direct comms. Public, **6–12 months minimum** — and for anything payment- or auth-related, longer.

> **The most important step is #2.** "We think nobody uses it" is how outages happen. Instrument per-client usage of every deprecated field and endpoint, and you can email the exact eleven accounts still calling it instead of guessing. Volunteering this in an interview signals real operational experience.

---

## 6. Evolution patterns (how to avoid needing a version)

### Expand / contract (parallel change) — the core technique

Never change in place. Three deploys:

```
1. EXPAND   — add the new alongside the old. Both work. Write to both, read from old
2. MIGRATE  — move clients over; read from new, keep writing both. Instrument old usage
3. CONTRACT — remove the old, once usage is zero and the sunset date has passed
```

Example — renaming `amount` to `amount_minor`:
```json
// Phase 1: both present, identical value. Docs mark `amount` deprecated.
{ "amount": 4999, "amount_minor": 4999 }

// Phase 2: writes accept either; if both are sent and disagree → 422 (never guess)
// Phase 3: `amount` removed at the sunset date.
```
Cost: a period of duplication. Benefit: zero breakage. **This is the same expand/contract discipline as a zero-downtime database migration**, which is exactly why you already know it from `spring boot/06-data/24-performance-n1-and-migrations.md`.

### Additive-only design habits

| Habit | Prevents |
|---|---|
| **Every new request field is optional with a documented default** | Forced client updates |
| **Enums are documented as open**; clients must handle unknown values | Enum-addition breakage |
| **Return an object, never a bare array** (Lesson 05) | Nowhere to add metadata |
| **Wrap collections in an envelope from day one** | Retrofitting pagination as a breaking change |
| **Never repurpose a field** | Silent semantic breakage — the worst kind |
| **New behaviour behind an opt-in flag or a new endpoint** | Changing defaults |
| **Nullable-from-the-start for anything that might be absent** | The later "make it nullable" break |
| **Prefer adding an endpoint over overloading one** | Parameter soup that can never be simplified |

### Feature flags vs versions
For behavioural changes, an opt-in flag is often better than a version: `POST /v1/payments` with `Ledger-Beta: new-fx-rounding`. Clients opt in, you measure, then flip the default in the next version. This is how you ship a risky change without a big-bang migration.

---

## 7. Production rules

| Rule | Why |
|---|---|
| **Version from day one (`/v1`), even if you never ship `/v2`** | Adding a version prefix later is itself a breaking change |
| **Publish a written definition of "breaking change" for your API** | Otherwise every argument is about opinions |
| **Reject unknown fields in request bodies; reject unknown query params always** | Silent typos are worse than errors |
| **Be a tolerant reader as a client; never `.strict()` a response schema** | Or you break every time the server improves |
| **Document enums as open, and handle unknown values in clients** | Enum additions are the most common accidental break |
| **Never repurpose a field's meaning** | No technical protection exists; only discipline |
| **Instrument per-client usage of everything you plan to remove** | You cannot deprecate what you can't measure |
| **Ship `Deprecation` + `Sunset` + a migration `Link` in-band** | Not everyone reads changelogs; everyone sees headers eventually |
| **Run at least one announced brownout before a sunset** | The only technique that reliably produces action |
| **Expand → migrate → contract. Never change in place** | Zero-breakage renames |
| **Keep a public, dated changelog** | It's the artifact integrators actually trust |
| **Contract-test against the previous version in CI** | So a breaking change fails the build, not a customer ([Lesson 26](../06-craft/26-testing-apis.md)) |

---

## 8. Interview traps

**Q1. "Client sends an extra field you didn't design for. Should it work?"**
Your question — deliver the §3 answer: **responses must tolerate, request bodies I'd reject by default, query params reject always, with the round-trip `PUT` case as the exception.** Naming the asymmetry between bodies and query params is the part that impresses.

**Q2. "How would you version an API with 5,000 integrators?"**
Frozen `/v1` path + date-based version pinning per account, with an internal chain of forward/backward transformers (the Stripe model). Then be honest about the cost, and add: *"and I'd invest most of the effort upstream — making changes additive — because every version is permanent maintenance."*

**Q3. "Is adding a field to a response a breaking change?"**
By the spec, no. In reality: only if clients are tolerant readers, and Hyrum's Law says some client is doing strict schema validation, snapshot-testing responses, or iterating `Object.keys()`. So: *"non-breaking by contract; I'd still check usage and announce it, and I'd never add a field that changes the meaning of an existing one."*

**Q4. "Is adding an enum value breaking?"**
**For responses, effectively yes** — clients with exhaustive `switch` and no default will break. Mitigation: document enums as open from day one, model them as open unions in typed clients, and treat the addition as announceable. For *request* enums, widening is safe.

**Q5. "URL versioning or header versioning?"**
Give the trade-off, then commit: *"URL for discoverability, routing and support — you can paste a URL into a ticket and reproduce it. Header for granularity and clean URLs. I'd ship `/v1` in the path, and if I later needed per-account pinning I'd add a version header on top of the frozen path, which is what Stripe does."*

**Q6. "How do you remove a field that 200 clients might use?"**
The five-phase process, and lead with **instrumentation** — you don't guess who uses it, you measure per client. Then expand/contract, a brownout, and a sunset date you honour.

**Q7. "What's the most dangerous kind of breaking change?"**
Semantic: same shape, new meaning. `amount` gross → net, or a timestamp switching from seconds to milliseconds, or a boolean's polarity flipping. Every client compiles and every one is wrong. **The only defence is never doing it** — add a new field instead.

**Q8. "Do internal APIs need versioning?"**
*"Not if you can genuinely coordinate deploys and you know every consumer. That holds at 5 services and fails at 50 — nobody knows who calls what, so you end up treating internal APIs like public ones. I'd skip formal versioning early but keep the additive discipline from day one, because the discipline is what makes versioning unnecessary."*

**Q9. "Client says your change broke them, but it was documented as non-breaking. What do you do?"**
Operationally: roll back or ship a compatibility shim *first*, investigate after — the customer's outage outranks being right. Then find out what they depended on (usually strict validation, key ordering, or a field's absence), fix the doc to say so explicitly, and add a contract test. **Never** lead with "you were doing it wrong." This question is testing judgement, not knowledge.

---

## 9. Build & break

### Build — the compatibility policy document
Write `docs/api-policy.md` for Ledger. It's short, and it's the artifact that makes every future argument fast:

```markdown
## What we consider a breaking change
[the §2 "breaking" list]

## What we may do at any time, without notice
- Add new endpoints, optional request fields, and response fields
- Add new values to any enum (clients MUST handle unknown values)
- Add new error codes (clients MUST have a default branch)
- Change `detail` text in error responses (`code` is the contract; text is not)
- Change the ordering of fields in a JSON object
- Improve performance or change response latency

## What clients MUST do to stay compatible
1. Ignore unknown response fields
2. Handle unknown enum values with a default branch
3. Branch on `code`, never on `detail` text
4. Treat all IDs as opaque strings; never parse them
5. Follow returned pagination cursors; never construct them

## Deprecation policy
- 6 months minimum notice for any removal
- `Deprecation` and `Sunset` headers on every deprecated endpoint
- One announced 30-minute brownout at least 30 days before sunset
```

That "what clients MUST do" section is the single highest-leverage paragraph in an API's docs, and almost nobody writes it.

### Build — a strict-request / tolerant-response pair
```ts
// SERVER: strict on the way in. Typos are errors.
const CreatePayment = z.object({
  amount_minor: z.number().int().min(50),
  currency:     z.enum(["usd","eur","gbp","inr"]),
  customer:     z.string().startsWith("cus_").optional(),
  description:  z.string().max(500).optional(),
  metadata:     z.record(z.string().max(500)).optional(),
}).strict();      // ← unknown key = 422 unknown_field

// CLIENT: tolerant on the way in. New server fields must not break us.
const PaymentView = z.object({
  id:           z.string(),
  amount_minor: z.number(),
  currency:     z.string(),
  // Open enum: known values are typed, unknown ones still parse.
  status: z.union([
    z.enum(["pending","requires_capture","succeeded","failed","refunded"]),
    z.string(),                      // ← forward compatibility, deliberately
  ]),
});

function label(status: string) {
  switch (status) {
    case "succeeded": return "Paid";
    case "failed":    return "Failed";
    case "refunded":  return "Refunded";
    default:          return "Processing";   // ← the branch that saves you
  }
}
```

### Build — expand/contract a rename, for real
In a scratch project, rename `amount` → `amount_minor` across three commits, and write a test for each phase:
1. **Expand:** response contains both; write accepts either; sending both with different values → `422`.
2. **Migrate:** add a per-client usage counter for `amount`; add `Deprecation`/`Sunset` headers.
3. **Contract:** remove `amount`; old clients get `422 unknown_field` on write and simply don't see it on read.
Then write the changelog entry you'd publish for each phase. **Writing the changelog is the exercise** — it forces you to state the client impact in plain language.

### Break — four experiments
1. **Strict response validation.** Point a `.strict()` response schema at an API, then have the server add one field. Watch the client break on a non-breaking change. That's why tolerant reading is a rule.
2. **Exhaustive switch, no default.** Add a new status server-side and watch the client render nothing, silently. Add the `default` branch.
3. **Silently ignore an unknown query param.** `GET /payments?statuss=succeeded` returns everything. Now imagine a reconciliation job. Fix it to 400 and feel the difference.
4. **Repurpose a field.** Change `amount` from gross to net in a scratch service and run your existing test suite. Watch it pass. That's the horror — no test protects you, because the shape didn't change.

### Explain out loud (2 minutes)
1. Define a breaking change, and name the three most dangerous ones.
2. Your unknown-fields policy, with the request/response and body/query asymmetries.
3. Two versioning strategies with trade-offs, and your default.
4. How you'd remove a field 200 clients might be using.

---

## Module 2 complete — checkpoint

Cold, no notes:

- [ ] The six REST constraints, and what statelessness buys you
- [ ] Where your API sits on Richardson, and why Level 2 is the destination
- [ ] HATEOAS: what it is, why it's rare, and the two patterns that *are* universal
- [ ] Turn a requirements paragraph into a resource list and URL set
- [ ] The nesting rule, and nest-to-create / flat-to-read / query-to-filter
- [ ] Why sequential IDs are a security problem
- [ ] Three ways to model an action, and how to choose
- [ ] Why offset pagination fails **two** ways, with numbers you measured yourself
- [ ] Write the keyset SQL, including the tiebreaker and row-value comparison
- [ ] What you'd tell a PM who demands an exact total count
- [ ] `PUT` vs `PATCH` omitted-field semantics, and which patch format you'd pick
- [ ] The lost-update race and three fixes, ranked — with the version check in the `WHERE`
- [ ] Bulk partial failure: `207`, per-item index, per-item idempotency, per-item metering
- [ ] The four audiences of an error and the field serving each
- [ ] Your 403-vs-404 rule, and the enumeration reason
- [ ] The unknown-field answer, with all four asymmetries
- [ ] How you'd deprecate and remove a field safely

That's a full API design interview. Any unticked box is a re-read, now.

---

## What's next

Design is done. Module 3 is **security**, which is where APIs actually get breached — and where the failures are authorization, not authentication. It starts with the full landscape: API keys, Basic, sessions, JWT, mTLS and HMAC signing, and how to choose between them for a given client type.

Next → **[Lesson 12: The authentication landscape](../03-security/12-authentication-landscape.md)**
