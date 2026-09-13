# Lesson 01 — What an API really is

> **Why this lesson exists:** almost everyone can *use* an API before they can *define* one. That gap is exactly what interviewers probe, because designing an API is a completely different skill from calling one. This lesson replaces the fuzzy "it's how two apps talk" idea with a precise one: **an API is a contract, and a contract's whole purpose is to let two sides change independently.** Every rule in the next 31 lessons is derived from that sentence.

**Time:** ~50 minutes · **Prereq:** none

---

## 1. The idea in one sentence

> **An API is a promise about behaviour that lets one piece of software depend on another without knowing how it works — and, crucially, without breaking when it changes.**

Read that again and notice what's *not* in it: HTTP isn't in it. JSON isn't in it. REST isn't in it. Those are implementation details of *one kind* of API. The essence is **promise** + **hiding** + **independent change**.

---

## 2. Why APIs exist: the coupling problem

Forget technology for a moment. Here's the actual problem.

You have code that needs a customer's outstanding balance. Without an API, the honest version of that code looks like this:

```ts
// No API. Direct coupling.
const rows = await db.query(
  `SELECT SUM(amount_minor) FROM invoice_lines l
     JOIN invoices i ON i.id = l.invoice_id
    WHERE i.customer_id = $1 AND i.status IN ('open','past_due')
      AND l.voided_at IS NULL`,
  [customerId]
);
const balance = rows[0].sum ?? 0;
```

This works. It also means:

1. **You now know the schema.** Tables, columns, join keys, the `voided_at` soft-delete convention, the exact set of statuses that count as "owed".
2. **The billing team can no longer change their schema.** Rename `amount_minor`, split `invoices` into two tables, add a `credit_notes` table that must now be subtracted — and your code silently returns a *wrong number*. Not a crash. A wrong number, in a financial report.
3. **Business logic is duplicated.** Does a `past_due` invoice in a grace period count? Does a partially-paid invoice count for the remainder? The billing team knows. Your query has guessed. Now there are two definitions of "balance" in the company, and they disagree.
4. **There's no place to put a rule.** When Finance says "balances must exclude disputed invoices," someone has to find all 14 queries across 6 services.

Now the same need, through an API:

```ts
const { balanceMinor, currency } = await billing.getCustomerBalance(customerId);
```

What changed is not the syntax. What changed is **who owns the knowledge**. The billing team owns "what balance means" and can restructure their entire database on a Tuesday; you own "I need a balance." The only thing you both agreed on is the promise:

> *Given a customer ID, you get their current outstanding balance in minor units plus a currency, or a NOT_FOUND error.*

That promise is the API. Everything else is negotiable.

**This is the whole point, and it's worth being able to say out loud in an interview:** an API is not a way to *transfer data*. It's a way to *transfer data without transferring knowledge*. The technical name for the knowledge you're refusing to share is **implementation detail**, and the discipline of refusing is **encapsulation** — the same idea as a `private` field in Java, scaled up to a company.

### The corollary that trips people up

If an API's job is to hide implementation, then **anything you expose becomes permanent**. Publish `customer_id` as an integer and you can never move to UUIDs. Return a database enum and you can never rename the enum. Include a `debug` field "just for now" and someone's production code will depend on it within a month.

> **Hyrum's Law** — worth memorising, and interviewers love it:
> *"With a sufficient number of users of an API, it does not matter what you promise in the contract: all observable behaviours of your system will be depended on by somebody."*
>
> Consequence: the *ordering* of your JSON array, the *precision* of your timestamps, the *casing* of your error codes, and even your *response latency* are part of your API whether you intended them to be. Design accordingly — and when someone asks *"can we just add this field quickly?"*, you now know why the answer involves a deprecation policy.

---

## 3. The analogy — and where the usual one lies to you

You've heard: *"An API is like a waiter in a restaurant. You order from the menu, the waiter goes to the kitchen, and food comes back."*

It's fine as far as it goes, but it teaches the wrong lesson, because it makes you think **the API is the waiter** — the messenger. It isn't. The waiter is the *transport* (that's HTTP). **The API is the menu.**

The menu is where the real engineering lives, and the analogy becomes genuinely useful once you push it:

| Restaurant | API | Why it matters |
|---|---|---|
| The **menu** lists what you may order | The API **surface**: endpoints, methods, fields | You cannot order what isn't on the menu. Undocumented behaviour isn't a feature |
| Each item has a **fixed name and description** | Endpoint + schema | "Pasta" must mean the same thing every time, or nobody can order confidently |
| **You don't specify the recipe** | Implementation is hidden | The kitchen changed supplier, technique, staff. You still got pasta |
| **"No substitutions"** | Validation and constraints | Constraints aren't rudeness; they're what makes the promise keepable |
| **Prices and portion sizes** | Rate limits, quotas, pricing | Every real API is metered somehow. Free ones are metered in requests/minute |
| **"Allergen information available"** | Documentation and error contracts | The interesting part is what happens when something goes wrong |
| **The kitchen can be closed** | 503, maintenance, degradation | The promise includes *availability*, not just shape |
| **A new menu is printed each season** | Versioning | And the old menu's regulars will complain. Loudly |
| **Regulars order "the usual"** | Hyrum's Law | You built an undocumented feature by accident and now it's load-bearing |

The lesson: when you design an API, **you are writing a menu, not building a kitchen.** Juniors design the kitchen and let the menu fall out of it (that's how you end up with `GET /getUserDataByIdV2`). Seniors design the menu first and then make the kitchen satisfy it. That is literally what "contract-first" means ([Lesson 25](../06-craft/25-openapi-contract-first.md)).

### A second analogy, for the operational half

An API is also like a **power socket**. The contract is: 230V, 50Hz, this physical shape, this much current available. You plug in anything you want. The power station can switch from coal to solar and your laptop doesn't care.

But notice what the socket contract includes beyond *shape*:
- **Voltage tolerance** (data validity)
- **Maximum current** (rate limits)
- **What happens on overload** — a breaker trips, it doesn't burn your house down (graceful failure, 429/503 rather than a hang)
- **A standard** everyone agreed to, even though it's arbitrary (why REST conventions matter more than REST correctness)

Keep both analogies. The menu explains the *shape* of an API; the socket explains its *guarantees*. Interviewers who ask "what makes a good API?" are usually satisfied by the menu answer and impressed by the socket one.

---

## 4. The anatomy of a contract: the four layers

This is the mental model I want you to carry for the rest of your career. When you "design an API," you're designing four layers, and **most bad APIs are bad because someone only designed the first one.**

### Layer 1 — Syntax (the shape)
*What can I call, and what do the bytes look like?*

```
POST /v1/payment_intents
Content-Type: application/json

{ "amount": 4999, "currency": "usd", "customer": "cus_9s2k" }
→ 201 Created
{ "id": "pi_3Nx8", "status": "requires_payment_method", "amount": 4999, ... }
```

Endpoints, methods, field names, types, required vs optional, status codes. This is the part tools can check for you (OpenAPI, Protobuf, GraphQL SDL). It's also the *easiest* layer, which is why so many people stop here.

### Layer 2 — Semantics (the meaning)
*What does it actually mean, and what happened as a result?*

Syntax says `amount: 4999`. Semantics answers:
- 4999 **what**? Cents. (Not dollars. Never floats — [Lesson 05](05-data-formats.md).)
- Is `amount` the amount *charged*, or *authorised*, or *requested before fees*?
- After a 201, has money moved? (No — a payment intent is a *plan*, not a payment. That distinction is a semantic design decision worth millions to Stripe.)
- What does `status: "requires_payment_method"` permit next? Which transitions are legal?
- Is `customer` required for a `pi` to be refundable later?

**No schema language can express any of this.** This layer lives in your documentation, your naming, and your error messages — which is exactly why naming is a real engineering activity and not bikeshedding.

> The single most common cause of integration bugs isn't syntax errors. It's two teams agreeing on the *shape* of a field and disagreeing on its *meaning*. `expires_at` — is that inclusive? UTC? Does an expired object still appear in list results? Every one of those has burned a production system somewhere.

### Layer 3 — Operational guarantees (the SLA)
*What can I rely on when reality intrudes?*

| Guarantee | The question a client actually needs answered |
|---|---|
| **Availability** | 99.9%? Which nine? Measured how? |
| **Latency** | p50 / p99 — and I care about p99, because that's what my users feel |
| **Throughput / limits** | How many requests before you 429 me? Per key or per IP? |
| **Consistency** | I just created it — will it appear in the very next `GET`? (Read-after-write, or eventually consistent?) |
| **Ordering** | Will webhooks arrive in order? (Almost always: **no**) |
| **Delivery** | At-least-once, at-most-once, exactly-once? (Exactly-once doesn't exist over a network — [Lesson 18](../04-production/18-reliability-and-idempotency.md)) |
| **Durability** | Once you 201, is it definitely persisted, or queued? |
| **Idempotency** | Can I safely retry? |

This layer is where senior engineers live and juniors don't look. If you can only remember one thing from this lesson to say in an interview, make it this: **"the response schema is the easy half of an API contract; the guarantees are the half that decides whether clients can be written correctly."**

Concretely: an API that returns a `200` but is eventually consistent forces *every single client* to write retry-and-poll logic. That's not a documentation footnote — that's an architectural decision you've imposed on hundreds of other engineers.

### Layer 4 — Lifecycle (the promise over time)
*How will this change, and how much warning do I get?*

Versioning strategy, deprecation policy and timeline, changelog, what counts as a breaking change, how long old versions live. [Lesson 11](../02-rest-design/11-versioning-and-evolution.md) is entirely this.

Real examples worth knowing, because they're the industry reference points:
- **Stripe** pins each account to the API version that was current when it signed up, and keeps *every* version working — for over a decade. They transform old-version requests/responses through a chain of "version changes". Extremely expensive, extremely beloved.
- **GitHub** uses dated media types (`application/vnd.github.v3+json`) and, for GraphQL, a formal preview + deprecation schedule.
- **AWS** effectively never breaks an API. Ever. Look at the shape of `ec2:DescribeInstances` and you're looking at 2006 design decisions preserved on purpose.
- **Google** publishes an internal-turned-public [API Improvement Proposals](https://google.aip.dev) set — worth skimming later; the "which company's style should I copy?" question has a real answer, and it's usually Google's AIPs for gRPC-shaped work and Stripe's for public REST.

> **The four-layer summary you should be able to recite:** syntax, semantics, guarantees, lifecycle. Juniors ship layer 1. Mid-levels document layer 2. Seniors negotiate layer 3. Staff engineers own layer 4.

---

## 5. The five axes — the real answer to "types of API"

Search "types of API" and you'll be told there are four: public, partner, private, composite. That's **one axis**, and it's the least technically interesting one. Worse, it leaves you unable to answer *"is REST a type of API?"* coherently.

Here's the map. An API sits at **one point on every axis simultaneously.**

### Axis 1 — Audience (who may call it)

| Type | Meaning | Real example | What changes about your design |
|---|---|---|---|
| **Public / open** | Anyone can sign up | Stripe, Twilio, OpenWeather | Docs are a product. Versioning is forever. Rate limits and abuse prevention are mandatory. Every error message is read by strangers |
| **Partner** | Contracted third parties | Uber↔Spotify, bank↔fintech, airline GDS feeds | mTLS or signed requests, IP allowlists, per-partner SLAs and quotas, negotiated change windows |
| **Private / internal** | Your own services | The 20 services behind your app | You can coordinate deploys, so you may break things — *if* you can prove nobody depends on it. Usually gRPC, usually less documented (often a mistake) |
| **Composite / aggregate** | One call fans out to many | A BFF (backend-for-frontend), GraphQL gateway | Exists purely to reduce client round-trips. Its contract belongs to the *client*, not the services |

> **Interview nuance:** "internal so we can move fast" is true only if you *know* your consumers. At 200 services nobody does, which is exactly why big companies end up treating internal APIs like public ones. Saying this shows you've thought past the slogan.

### Axis 2 — Architectural style (the shape of the contract)

This is what people usually mean when they say "type of API," and the entirety of Module 5 is dedicated to it. The preview:

| Style | One-line identity | Where it dominates today |
|---|---|---|
| **REST** | Resources addressed by URL, manipulated with HTTP methods | ~80% of public web APIs. The default, and the correct default |
| **GraphQL** | One endpoint, client declares the exact shape it wants | Client-heavy products with many screens: GitHub, Shopify, Meta |
| **gRPC** | Typed RPC over HTTP/2 with Protobuf | Internal service-to-service at scale: Google, Netflix, Uber, most k8s ecosystems |
| **SOAP** | XML envelopes + WSDL + WS-* standards | Legacy enterprise, banking, telco, government. Not dead — *load-bearing* |
| **WebSocket** | Persistent bidirectional connection | Chat, collaborative editing, trading, games |
| **SSE** | Server-pushed event stream over plain HTTP | Notifications, live dashboards, LLM token streaming |
| **Webhooks** | *They* call *you* when something happens | Every payments/SaaS platform. "Reverse API" |
| **tRPC** | Typed RPC where server types leak into the client on purpose | TypeScript monorepos, same-team full-stack |
| **OData / JSON:API** | Standardised REST conventions incl. query language | Microsoft/SAP ecosystems; JSON:API in Ruby/Ember-adjacent shops |

**These are not mutually exclusive.** Stripe is REST + webhooks. GitHub is REST + GraphQL + webhooks. A typical modern product is: REST or GraphQL at the edge, gRPC internally, webhooks outbound, SSE or WebSocket for live updates. Saying *"we'd use REST at the edge and gRPC internally, for these reasons"* is a much stronger interview answer than picking one.

### Axis 3 — Protocol / transport

`HTTP/1.1` · `HTTP/2` · `HTTP/3 (QUIC)` · raw `TCP` · `AMQP` (RabbitMQ) · `MQTT` (IoT) · `Kafka protocol`.

Why you care: the transport decides your *performance ceiling*. HTTP/1.1 gives you ~6 parallel connections per origin and head-of-line blocking; HTTP/2 gives multiplexing (and makes gRPC streaming possible); HTTP/3 removes TCP head-of-line blocking on lossy mobile networks. [Lesson 02](02-journey-of-a-request.md) covers this with numbers.

### Axis 4 — Interaction pattern

| Pattern | Shape | Example |
|---|---|---|
| **Request–response** | Ask, wait, get answer | `GET /payments/pi_123` |
| **Request–accept–poll** | Ask, get `202 Accepted`, poll for completion | Video transcode, report generation |
| **Request–accept–callback** | Ask, get 202, they webhook you later | Payment settlement, KYC checks |
| **Server push (one-way)** | Subscribe, receive | SSE dashboards |
| **Bidirectional stream** | Both sides send anytime | WebSocket chat, gRPC streaming |
| **Fire-and-forget** | Send, don't wait | Analytics beacons, log shipping |

This axis is the one most often forgotten, and it's where the interesting design questions live. *"The operation takes 4 minutes. Design the API."* — if you reach for request–response, you've failed the question ([Lesson 23](../05-beyond-rest/23-realtime-and-webhooks.md)).

### Axis 5 — Layer (what kind of boundary it crosses)

| Kind | Example | Note |
|---|---|---|
| **Remote / web API** | Stripe REST API | What this course is about |
| **Library / SDK API** | `Array.prototype.map`, React's hooks, `stripe-node` | Same contract discipline, no network. **React's props are an API** — remember that in the React track |
| **OS / system API** | `open()`, Win32, POSIX | Where the term originated |
| **Database API** | JDBC, the Postgres wire protocol | A protocol *and* a library API |
| **Hardware / driver API** | GPIO, OpenGL | Same idea, different stakes |

> **Why this matters for you specifically:** everything you learn about API design applies directly to designing React components and TypeScript modules. A component's props are its syntax; what the component does with them is its semantics; whether it re-renders predictably is its guarantee; and a prop rename is a breaking change. The tracks are the same skill at three scales.

---

## 6. Reading the map: five real systems

Locating real APIs on the five axes is the fastest way to make this concrete. Do this exercise with any API you meet from now on.

| System | Audience | Style | Transport | Pattern | Why they chose it |
|---|---|---|---|---|---|
| **Stripe API** | Public | REST + webhooks | HTTP/1.1 & 2 | Req–resp + callback | Payments involve slow external networks (banks). Webhooks are unavoidable, so the API is honest about async |
| **GitHub** | Public | REST **and** GraphQL | HTTP/2 | Req–resp | REST for simple/scriptable use, GraphQL because their object graph is deep and clients over-fetched badly |
| **Google internal** | Private | gRPC | HTTP/2 | All four | Millions of internal calls/sec; JSON parsing and header overhead are a real bill |
| **Slack** | Public + partner | REST + Events API (webhooks) + WebSocket (Socket Mode) | HTTP + WS | All | Bots need push. Firewalled customers can't receive webhooks, so Socket Mode inverts the direction |
| **AWS S3** | Public | REST-ish (HTTP with signed requests) | HTTP/1.1 | Req–resp | Designed 2006, must never break, so it's deliberately primitive and extremely stable |

Notice the pattern: **nobody chose a style because it was fashionable.** Each choice is a consequence of a constraint — external slowness, graph depth, request volume, firewalls, longevity. That is exactly the reasoning an interviewer wants to hear, and it's why Module 5 ends with *"choosing an API style"* rather than starting with it.

---

## 7. What makes an API *good* — the checklist you'll be judged against

Not a style guide. These are the properties that determine whether other engineers succeed or suffer.

| Property | Test for it | Failure looks like |
|---|---|---|
| **Predictable** | Can a developer guess the next endpoint after learning three? | `/users/:id`, `/getOrder?id=`, `/v2/Product/fetch` in one API |
| **Consistent** | Same concept, same name, same casing, same shape, everywhere | `user_id` here, `userId` there, `UserID` in errors |
| **Minimal** | Can you remove anything without losing a use case? | 14 optional query params, 11 of which nobody uses but you can't delete |
| **Hard to misuse** | Is the wrong call impossible, or merely discouraged? | `DELETE /users` with no ID deletes everyone |
| **Honest about failure** | Do errors say what to *do*, not just what broke? | `500 {"error":"error"}` |
| **Observable** | Can a client correlate their request with your logs? | No request ID in the response, so support tickets are unanswerable |
| **Evolvable** | Can you add a field without a version bump? | Clients that 400 on unknown fields ([Lesson 11](../02-rest-design/11-versioning-and-evolution.md)) |
| **Documented at layer 2** | Does the doc explain meaning, not just types? | "amount: number" with no unit |

**The single best design heuristic** — and a great thing to say out loud in an interview:

> *"Write the client code first."*

Before you design endpoints, write the five lines a consumer would want to write. If those five lines need three round-trips, a client-side join, and a `setTimeout` to wait for consistency, your design is wrong — and you found out before building it. This is why Stripe's API feels good: they wrote the `stripe.charges.create(...)` call, then made a server that deserved it.

---

## 8. Interview traps

> Answer out loud before reading. Recognition is not recall.

**Q1. "What is an API?"**
The junior answer is *"a way for two applications to communicate."* The strong answer:
> *"A contract that lets one component depend on another's behaviour without depending on its implementation — so both can change independently. Concretely it's four layers: syntax, semantics, operational guarantees, and lifecycle. Most API problems in production are semantic or operational, not syntactic."*

**Q2. "How many types of APIs are there?"**
Trap: they want to see if you'll recite a blog list. Strong answer: *"Depends on the axis — audience, architectural style, transport, interaction pattern, or layer. They're independent: Stripe is public + REST + HTTP + request-response-plus-callback, all at once. Which axis are you interested in?"* Asking that clarifying question is itself a positive signal.

**Q3. "Is REST an API?"**
No. REST is an **architectural style** — a set of constraints. An API *built in* the REST style is a REST API. (Bonus: REST doesn't even require HTTP, though in practice it's always HTTP.)

**Q4. "What's the difference between an API and a web service?"**
A web service is a *subset*: an API exposed over a network, historically implying SOAP/XML. Every web service is an API; not every API is a web service (`Math.max` is an API). Interviewers ask this to hear whether you know APIs exist outside HTTP.

**Q5. "Why not just let services share a database?"**
The strongest one-liner: *"Because a shared database makes your schema your API — the widest possible contract, with no owner, no versioning, and no way to enforce invariants."* Then name the four costs from §2: schema freeze, duplicated logic, no invariant enforcement, and a wrong-answers-not-crashes failure mode. Mention the legitimate exception too — a **read-only replica for analytics**, deliberately accepted as coupled, which shows you're not dogmatic.

**Q6. "What's an API-first company?"**
One where the API is designed and reviewed *before* implementation, and the company's own product consumes the same API it publishes (no privileged internal backdoor). Twilio and Stripe are the canonical examples; Amazon's 2002 Bezos mandate ("all teams will expose data through service interfaces… no other form of interprocess communication allowed") is the origin story every interviewer recognises, and it's why AWS exists at all.

**Q7. "You need to add a field to a response. Breaking change?"**
*Adding* an optional field is normally non-breaking — **but** only if clients are tolerant readers, and Hyrum's Law says someone may be doing strict-schema validation, snapshot testing responses, or reading `Object.keys()`. Correct senior answer: *"Non-breaking by the spec, but I'd check consumer behaviour before calling it safe, and I'd never add a field that changes the meaning of an existing one."* Full treatment in [Lesson 11](../02-rest-design/11-versioning-and-evolution.md).

**Q8. "What's the hardest part of API design?"**
There's a real answer and it's not "authentication": **you can't take anything back.** Naming, semantics and defaults are permanent the moment a third party ships code against them. That's why contract-first review exists and why *"we'll clean it up later"* is not available to you.

---

## 9. Build & break

### Build: read a real contract like an engineer
No code yet. Do this — it takes 20 minutes and pays off for the rest of the track.

1. Open Stripe's API reference for `POST /v1/payment_intents` (`docs.stripe.com/api/payment_intents/create`).
2. Write down in your notes:
   - Three **layer-1 (syntax)** facts: required fields and their types.
   - Three **layer-2 (semantics)** facts the schema alone would not tell you (units? does a 201 mean money moved? which statuses can follow which?).
   - Two **layer-3 (guarantee)** facts (idempotency? rate limits? is a created object immediately readable?).
   - One **layer-4 (lifecycle)** fact (how versioning works there).
3. Then answer: *why is `amount` an integer?* Write the reason down. If you're not certain, that's a `QUESTIONS.md` entry — and [Lesson 05](05-data-formats.md) answers it.

### Break: feel Hyrum's Law
```bash
curl -s https://api.github.com/users/torvalds
```
Look at the response. Now:
1. Pick three fields you'd consider "internal noise" (`gravatar_id`, `node_id`, the `*_url` templates).
2. Imagine deleting one. Who breaks? You can't know — and neither can GitHub. That's why `gravatar_id` is still there, empty, years after Gravatar stopped mattering.
3. Write one sentence in your notes: *"The cost of adding a field is paid forever."* You'll re-read it during Lesson 11.

### Explain out loud (60 seconds, no notes)
1. What an API is, in contract terms.
2. The four layers, and which one causes most production bugs.
3. Why "types of API" is a bad question, and the better framing.

If you stumbled on any of the three, re-read that section now — not tomorrow.

---

## What's next

You now know what an API *is*. Next you'll learn what actually happens when one is called — every stage from `fetch()` to bytes on the wire and back, because you cannot design good APIs while the network is a black box, and every latency question in every interview is really a question about this pipeline.

Next → **[Lesson 02: The journey of one request — DNS → TCP → TLS → HTTP](02-journey-of-a-request.md)**
