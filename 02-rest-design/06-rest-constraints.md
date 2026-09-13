# Lesson 06 — REST, actually: constraints & maturity

> **Why this lesson exists:** "REST" is the most-claimed and least-understood term in web engineering. Nearly every "REST API" in production violates at least one REST constraint, and that's often *fine* — but you need to know **which** constraint you're breaking and **what you're giving up**, because that's exactly what a senior interviewer probes. This lesson also settles HATEOAS: what it is, why almost nobody does it, and how to answer when asked.

**Time:** ~70 minutes · **Prereq:** Module 1

---

## 1. The idea in one sentence

> **REST is not a protocol, a format, or a URL style — it is a set of six architectural constraints that trade away efficiency and convenience in exchange for scalability, evolvability and independent deployment of client and server.**

The word that matters is **trade**. Every REST constraint costs you something. Fielding's 2000 dissertation was explicit about this: constraints are chosen for the properties they *induce*, and you may knowingly relax them.

---

## 2. Where REST came from, and why that matters

Roy Fielding co-authored HTTP/1.1 and then, in his 2000 doctoral dissertation, described the architectural style the Web had accidentally discovered. He named it **RE**presentational **S**tate **T**ransfer.

**The problem he was solving was not "how do I design a nice API."** It was: *how do you build a distributed hypermedia system that scales to millions of independently-operated servers and billions of clients, where no two components can ever be deployed together?*

That's the Web. And the answer — the constraints below — is why a browser written in 2026 can render a page served in 1996, and why you can deploy your API without asking permission from anyone who uses it.

Understanding this origin fixes the most common confusion in one move:

> **REST was designed for the *open web*, where you have zero control over your clients. Most "REST APIs" today are internal, with a handful of known clients that deploy alongside the server. When you're in that situation, some REST constraints are paying for a benefit you don't need.** That's the real reason gRPC and tRPC exist and thrive — and being able to say this is a genuine senior signal.

Note also: **REST does not require HTTP.** It's an architectural style; HTTP is one implementation. In practice REST-over-HTTP is the only thing anyone builds, but the distinction is a real interview question.

---

## 3. The six constraints

Learn these as **constraint → what it buys → what it costs → how APIs violate it.** That triple is the answer to every REST question you'll ever be asked.

### Constraint 1 — Client–Server

> Separate the user interface from the data storage. They communicate only through the interface.

| Buys you | Costs you |
|---|---|
| Independent evolution and deployment | Network round trips |
| The UI can be rewritten (web → mobile → CLI) with no server change | Latency you can't remove |
| Different scaling profiles for each side | Two codebases, two deploy pipelines |

**How it's violated:** server-rendered templates that couple presentation to data (`GET /users` returning HTML with the user table baked in), or an API that returns display-formatted strings (`"total": "$49.99"` instead of `4999` + `"usd"`) — you've moved a presentation decision to the server, and now every client is stuck with your locale.

### Constraint 2 — Statelessness

> Each request contains all the information needed to understand it. **The server stores no client context between requests.**

This is the most important constraint and the most misunderstood one.

**What it does NOT mean:** "the server has no state." Of course it has state — that's what the database is for. It means **no *client session* state on the server**. The server remembers your data; it does not remember *you* between calls.

```
❌ Stateful:   POST /login          → server stores "session 7 = user 42" in memory
               GET  /cart          → server looks up session 7 to know who's asking

✅ Stateless:  GET  /cart
               Authorization: Bearer <token containing/pointing to user 42>
               → every request re-establishes who is asking, from the request itself
```

| Buys you | Costs you |
|---|---|
| **Horizontal scaling** — any server can handle any request, so you can add boxes behind a round-robin LB | Every request re-sends auth (bigger requests) |
| **Resilience** — a server crashing loses nothing; no session to migrate | Re-authentication/re-authorization work per request |
| **No sticky sessions** — no load-balancer affinity to configure or debug | The client must manage more state |
| **Simple deploys** — restart any instance whenever you like | Some interactions (multi-step wizards) get awkward |

**This is the constraint that makes cloud-native architecture possible.** Autoscaling, rolling deploys, blue/green, spot instances, serverless — all of it requires that killing an instance mid-flight loses nothing. Say that in an interview and you've explained *why* anyone cares about statelessness rather than reciting the definition.

**How it's violated, in order of how often you'll see it:**
1. **In-memory sessions.** Works on one box; breaks the moment you scale to two. The classic symptom: *"users get randomly logged out"* — because half the requests land on the box without their session.
2. **Sticky sessions as a fix.** Now the LB is stateful, one box's death logs out all its users, and rolling deploys become disruptive. It's a workaround, not a solution.
3. **Server-side pagination cursors held in memory.** The next page only works if you hit the same box.
4. **Multi-step flows with server-held partial state** ("step 2 of 4"). The fix is to return the accumulated state to the client, or persist it as a real resource (a `draft_order`) with its own URI — which is often the *better* design anyway.

> **The honest nuance interviewers reward:** a shared session store (Redis) is *technically* still server-side session state, but it satisfies the property REST cares about — any instance can serve any request. Purists will argue; the useful framing is: *"statelessness is a property of the individual server, and what it buys is that requests are not tied to a particular instance. Redis-backed sessions preserve that property while keeping revocation easy, which is why plenty of serious systems prefer them to JWTs."* That connects to [Lesson 13](../03-security/13-sessions-and-jwt.md) and shows you're not dogmatic.

### Constraint 3 — Cacheability

> Responses must label themselves as cacheable or not.

| Buys you | Costs you |
|---|---|
| Eliminated round trips (the fastest request is the one never sent) | Staleness — the hardest correctness problem in the field |
| CDN offload; massive cost reduction | Invalidation complexity |
| `304 Not Modified` = correctness *and* bandwidth | Risk of leaking one user's data to another via a shared cache |

**How it's violated:** the overwhelmingly common violation is **saying nothing**. An API with no `Cache-Control` and no `ETag` on any endpoint has opted out of the single largest performance lever HTTP offers, and the entire browser/CDN/proxy layer is forced to guess. Full treatment in [Lesson 17](../04-production/17-caching-and-performance.md).

### Constraint 4 — Uniform interface

> The interface between components must be uniform. This constraint has **four sub-constraints**, and they're the ones that actually generate "API design rules."

#### 4a. Identification of resources
Everything interesting has a URI. `/v1/payments/pi_3Nx8` identifies one payment, forever.

#### 4b. Manipulation through representations
You never touch the resource; you exchange **representations** of it. The same payment can be `application/json`, `text/csv`, or a PDF receipt. To change it, you send a representation of the desired state.

> The name **RE**presentational **S**tate **T**ransfer is literally this: you transfer representations of state. Being able to unpack the acronym meaningfully is a nice small win in an interview.

#### 4c. Self-descriptive messages
Every message contains everything needed to process it: method, `Content-Type`, cache directives, auth. **No out-of-band context.** This is what lets a proxy that knows nothing about your business cache, route, compress and rate-limit your traffic correctly.

**How it's violated:** `POST /api` with `{"action": "getPayment", "id": "pi_1"}`. The message is now opaque — no intermediary can tell it's a read, so nothing can cache it, no gateway can rate-limit reads differently from writes, and every request looks identical in your metrics. **This is RPC-over-HTTP, and it's the single most common "REST" violation.** (It's also exactly what GraphQL does, which is why GraphQL caching is hard — [Lesson 21](../05-beyond-rest/21-graphql.md).)

#### 4d. HATEOAS — Hypermedia as the Engine of Application State

> The client should discover what it can do next from **links in the response**, not from hardcoded URL knowledge.

```json
{
  "id": "pi_3Nx8",
  "status": "requires_capture",
  "amount_minor": 4999,
  "_links": {
    "self":    { "href": "/v1/payments/pi_3Nx8" },
    "capture": { "href": "/v1/payments/pi_3Nx8/capture", "method": "POST" },
    "cancel":  { "href": "/v1/payments/pi_3Nx8/cancel",  "method": "POST" }
  }
}
```

The idea is genuinely elegant. The client doesn't need to know that capture is only legal from `requires_capture` — **the absence of the link tells it.** State machine logic lives in one place (the server), and URLs become an implementation detail the server can change freely.

**Why almost nobody does it:**

1. **Clients don't actually work that way.** Real client code is `if (payment.status === "requires_capture") showCaptureButton()`. Nobody writes a generic hypermedia agent, so the indirection buys nothing while costing payload size and complexity.
2. **It doesn't remove coupling, it moves it.** The client still has to know what `"capture"` *means* as a link relation. You've swapped URL coupling for rel-name coupling — a real improvement, but a smaller one than advertised.
3. **No agreed format.** HAL, JSON:API, Siren, JSON-LD, Collection+JSON all compete. Choosing means choosing a niche.
4. **Tooling doesn't support it.** OpenAPI generators, SDKs and mocks are all built around fixed endpoints.
5. **The economics differ from the Web's.** HATEOAS pays off when you have unknown clients you can never coordinate with (browsers + humans reading links). API clients are *programs written against documentation* — a fundamentally different situation.

**The interview answer** (this question is asked a lot, and dogma in either direction is a red flag):

> *"HATEOAS is the constraint that makes something Level 3 REST, and it's the one nearly everyone skips. I've never seen a client that genuinely discovers its way through an API at runtime — real clients hardcode paths and branch on status. But the pragmatic subset is genuinely valuable and I do use it: pagination links (`next`/`prev` cursors, so the client never constructs a cursor URL), and `Location` on `201`. Those two are hypermedia that clients actually follow. Full HAL/Siren I'd only reach for if I had truly uncoordinated clients — and if I needed that much discoverability I'd probably question whether REST was the right style at all."*

Note the two genuinely successful hypermedia cases, worth naming: **pagination cursors** (Stripe, GitHub, Slack all return opaque `next` URLs/cursors so they can change the underlying scheme freely — this is HATEOAS earning its keep) and **the web itself**.

### Constraint 5 — Layered system

> The client cannot tell whether it's talking to the origin server or an intermediary.

| Buys you | Costs you |
|---|---|
| Load balancers, CDNs, API gateways, WAFs, service meshes, caches — all transparently insertable | Latency per hop |
| Security boundaries (terminate TLS at the edge, auth at the gateway) | Harder debugging (which hop broke?) |
| Legacy systems wrapped behind a modern facade | Header/IP trust problems (`X-Forwarded-For`) |

This constraint is why the entire cloud infrastructure industry can exist. **Every piece of infrastructure between your client and your app is only possible because of this constraint plus self-descriptive messages.**

**How it's violated:** relying on the client's direct IP, requiring a direct TCP connection, or embedding absolute URLs with your internal hostname in responses (so the response breaks behind a proxy).

### Constraint 6 — Code on demand *(optional)*

> The server can ship executable code to extend the client.

That's JavaScript in a web page. Explicitly **optional** in the dissertation, and essentially never relevant to API design. Know it exists so you can name all six.

---

## 4. The Richardson Maturity Model

Leonard Richardson's four levels are the standard vocabulary for "how RESTful is this?" Interviewers use it, so know it — and know that Level 2 is the industry's actual destination.

### Level 0 — The swamp of POX
One endpoint, one method, action in the body.
```http
POST /api
{"action": "getPayment", "id": "pi_1"}
```
HTTP is a tunnel. SOAP lives here. So do most `/graphql` endpoints, technically.

### Level 1 — Resources
Multiple URIs, still one method.
```http
POST /payments/pi_1   {"action": "get"}
POST /payments/pi_1   {"action": "refund"}
```
Progress: things have identity. But methods and status codes are still unused.

### Level 2 — HTTP verbs and status codes
```http
GET    /payments/pi_1        → 200
POST   /payments             → 201 + Location
PATCH  /payments/pi_1        → 200
DELETE /payments/pi_1        → 204
```
Methods carry meaning, status codes carry outcomes, caching works, intermediaries understand your traffic.

> **This is where Stripe, GitHub, Twilio, AWS and essentially every API you admire actually sit.** When someone says "REST API," this is what they mean. Level 2 done *well* — consistent naming, correct status codes, real error contracts, sane pagination, honest cache headers — is the practical maximum, and it is genuinely hard to achieve. Most APIs that claim Level 2 are Level 2 with three Level 1 endpoints hiding in them.

### Level 3 — Hypermedia controls
Level 2 + links driving state transitions. Fielding's position is that **only Level 3 is REST at all**; he wrote a well-known 2008 post titled "REST APIs must be hypertext-driven" precisely because of this.

> **How to handle "is your API RESTful?"**: *"It's Richardson Level 2, which is what the industry means by REST. It's not Level 3 — no hypermedia controls beyond pagination links and `Location` headers — so strictly by Fielding's definition it isn't REST. I'd rather be consistent and well-documented at Level 2 than half-implement HATEOAS."* That answer demonstrates you know the theory *and* have judgement about it. Both halves matter.

---

## 5. When to knowingly break a constraint

This is the section that makes you useful in a design review, rather than a rule-quoter.

| Constraint | Break it when | Accept this cost |
|---|---|---|
| **Statelessness** | WebSocket connections (inherently stateful); server-side session for revocation-critical apps (banking) | Sticky routing or a shared store; harder horizontal scaling |
| **Uniform interface** (4c/4d) | An endpoint whose operation genuinely isn't CRUD: `POST /payments/pi_1/capture` | Purists object; intermediaries can't cache it (they couldn't anyway — it's a write) |
| **Uniform interface** (self-descriptive) | GraphQL's single endpoint, when client-driven queries genuinely solve your over-fetching problem | You lose HTTP caching, per-endpoint metrics and per-endpoint rate limits, and must rebuild all three yourself |
| **Cacheability** | Data that is per-user and volatile | Nothing — just say `Cache-Control: private, no-store` explicitly rather than silently |
| **Client–server** | Server-driven UI (Airbnb's, Shopify's) where you must change the app without an App Store release | Presentation coupling; the client becomes a renderer |
| **Resource orientation** | Batch/bulk endpoints (`POST /payments/bulk`), search (`POST /search`) | Non-uniform, non-cacheable; needed anyway ([Lesson 09](09-writes-patch-and-bulk.md)) |

**The rule:** break a constraint deliberately, document why, and know the cost. *"We use `POST /reports/search` because filters exceed URL length limits; the cost is that it isn't cacheable, which is fine because report queries are unique per user anyway"* — that's a senior sentence. *"We just use POST for everything"* is not.

---

## 6. REST vs RPC — the framing that actually clarifies

Almost every API argument is really this one.

| | REST | RPC |
|---|---|---|
| **You think in** | **Nouns** — resources with state | **Verbs** — procedures with parameters |
| Example | `POST /payments` | `createPayment(amount, currency)` |
| Adding capability | New resource or sub-resource | New method |
| Discoverability | Uniform: know one endpoint, guess the rest | Per-method: read the docs |
| Caching | Free via HTTP | Build it yourself |
| Best when | Many clients, CRUD-shaped domain, public API | Action-shaped domain, known clients, internal |
| Examples | Stripe, GitHub | gRPC, tRPC, JSON-RPC, SOAP |

**The insight worth having:** some domains are genuinely noun-shaped (a payment, a customer, a repository) and some are genuinely verb-shaped (`transcodeVideo`, `runSimulation`, `sendOtp`, `rebalancePortfolio`). Forcing a verb-shaped domain into nouns produces the absurdities REST is mocked for — inventing a `TranscodeJob` resource just to say "please transcode."

Except… that invention is often *correct*, and this is the subtle bit. `POST /transcode_jobs` → `202` + a job resource you can poll, cancel, retry and audit is genuinely better than `transcodeVideo()`, because **the long-running operation deserves an identity.** You get status, cancellation, history and idempotency for free. So the honest position: *"resource-ify actions that have a lifecycle; use a sub-path action verb for actions that are instantaneous state transitions."*

---

## 7. Interview traps

**Q1. "What is REST?"**
Weak: *"Using HTTP methods with JSON."* Strong:
> *"An architectural style with six constraints — client-server, statelessness, cacheability, uniform interface, layered system, and optional code-on-demand — chosen to make a distributed system scalable and independently evolvable. In practice most 'REST APIs' are Richardson Level 2: resources, HTTP verbs and status codes, but no hypermedia."*

**Q2. "What does statelessness mean? Isn't a database state?"**
The distinction is **client session state** vs **resource state**. The server may store all the data it likes; it must not remember *who you are* between requests. What it buys: any instance can serve any request, so you get horizontal scaling, painless deploys and crash resilience.

**Q3. "So JWTs make you stateless and sessions don't?"**
Careful — this is a trap that catches people who learned it as a slogan.
> *"JWTs move the session data to the client, so no instance needs to remember you — that's genuinely stateless. But you pay for it with revocation: you can't invalidate a token you don't track, so you either accept a window of validity or reintroduce a denylist, at which point you're stateful again. A Redis session store isn't 'stateless' in the purist sense but it preserves the property that actually matters — any instance can serve any request — and revocation is trivial. Which is why plenty of serious systems choose sessions."*

**Q4. "Is `POST /users/123/deactivate` RESTful?"**
Strictly, no — it's an RPC verb in a URL. But it's the right choice, and here's the defence:
> *"The pure alternative is `PATCH /users/123 {"active": false}`. That's more RESTful, but it's worse in three ways: deactivation usually has side effects (revoke sessions, cancel subscriptions, notify) which a field-set doesn't communicate; it can't carry action-specific parameters like a reason code; and it can't be permissioned separately from other profile edits. I'd use the action sub-path, keep it a `POST` on a sub-resource so it's still noun-scoped, and accept that it's Level 2-with-an-exception."*
That answer beats both "yes it's fine" and "no, use PATCH."

**Q5. "What's HATEOAS and do you use it?"**
See §3, 4d. **Never** answer "I don't know what that is" — it's the single most common REST theory question. And never answer with pure enthusiasm either; the interviewer almost certainly doesn't use it in full and is testing your judgement.

**Q6. "Is GraphQL RESTful?"**
No — one endpoint, `POST` for reads, no resource identity, no HTTP caching, not self-descriptive to intermediaries. It's Level 0 by Richardson, which sounds like an insult but isn't: GraphQL **deliberately** trades those properties for client-specified queries. The right frame is that it optimises a different variable.

**Q7. "Your API has `/getUser`, `/createUser`, `/deleteUser`. What's wrong?"**
It's RPC wearing REST's clothes: the verb is in the URL, so it's Level 1 at best. Consequences, not just aesthetics: `GET /getUser` and `POST /createUser` can't be distinguished by an intermediary for caching or read/write rate-limiting; you'll accumulate `/getUserV2` and `/getUserByEmail` forever with no uniform pattern; and clients can't guess anything. Fix: `GET /users/{id}`, `POST /users`, `DELETE /users/{id}`.

**Q8. "Which REST constraint does a CDN depend on?"**
Layered system + cacheability + self-descriptive messages. **This is a great question to be asked** because it's checking whether you can connect the theory to infrastructure. A CDN can only work because the client can't tell it isn't the origin, and because responses declare their own cacheability.

**Q9. "Does REST require JSON?"**
No. REST says nothing about format — that's content negotiation. A REST API can serve XML, Protobuf, HTML or CSV for the same resource.

**Q10. "What's the difference between REST and HTTP?"**
HTTP is a protocol; REST is an architectural style that HTTP was designed to support (same author). You can use HTTP un-RESTfully (`POST /api` with an action field) and you can theoretically implement REST over another protocol.

---

## 8. Build & break

### Build — audit a real API against the six constraints
Pick **one** API you've used (or Ledger's design, once you've read the blueprint). Fill this table in your notes. Do it for real; it takes 20 minutes and it's the exact exercise a design review is.

| Constraint | Satisfied? | Evidence | If violated: what's the cost, and is it justified? |
|---|---|---|---|
| Client–server | | | |
| Stateless | | | |
| Cacheable | | | |
| Uniform: resource identity | | | |
| Uniform: representations | | | |
| Uniform: self-descriptive | | | |
| Uniform: HATEOAS | | | |
| Layered | | | |

Then write one paragraph: *"This API is Richardson Level __ because ___. The one change with the highest payoff would be ___."*

### Build — the same endpoint at all four levels
Write `scratch/maturity.http` and express *"refund payment pi_1 for 2000 cents"* four times:

```http
### Level 0 — tunnel
POST /api
{"method":"refundPayment","params":{"id":"pi_1","amount":2000}}

### Level 1 — resources, one verb
POST /payments/pi_1
{"action":"refund","amount_minor":2000}

### Level 2 — verbs + status codes  ← the target
POST /payments/pi_1/refunds
Idempotency-Key: 8f14e45f
{"amount_minor":2000,"reason":"requested_by_customer"}
→ 201 Created
  Location: /v1/refunds/re_9k2
  {"id":"re_9k2","status":"pending","amount_minor":2000}

### Level 3 — hypermedia
→ 201 Created
  {"id":"re_9k2","status":"pending",
   "_links":{"self":{"href":"/v1/refunds/re_9k2"},
             "cancel":{"href":"/v1/refunds/re_9k2/cancel","method":"POST"},
             "payment":{"href":"/v1/payments/pi_1"}}}
```

For each level write one sentence on what an intermediary (a CDN, a gateway) can and cannot do with it. That's the lesson made concrete.

### Break — feel statelessness fail
1. Write a tiny Express app with an in-memory `Map` of sessions: `POST /login` stores one, `GET /me` reads it.
2. Run **two** instances on ports 3001/3002 behind any round-robin proxy (or just alternate `curl` calls manually).
3. Log in against 3001, then `GET /me` on 3002 → `401`. You've just reproduced the *"users randomly logged out"* bug that has burned every engineer once.
4. Now fix it twice, and note the trade-offs you feel: (a) a Bearer token containing the identity — truly stateless, but how would you log someone out? (b) a shared Redis store — revocation is one `DEL`, but you've added a dependency that can go down.

Write down which you'd pick for Ledger, and why. You'll revisit this in [Lesson 13](../03-security/13-sessions-and-jwt.md).

### Explain out loud (90 seconds)
1. The six constraints, and what statelessness buys you.
2. Why HATEOAS is rare, and the two hypermedia patterns that *are* universal.
3. Where your API sits on Richardson, and why Level 2 is the sane destination.
4. One constraint you'd knowingly break, and what it costs.

---

## What's next

Constraints are the theory. Next is the craft: turning a domain into resources and URLs that other engineers can guess without reading your docs — including the hard cases (actions, nested resources, singletons, filters, and the question of what to do when your domain isn't nouns).

Next → **[Lesson 07: Resource modelling & URL standards](07-resource-modelling-and-urls.md)**
