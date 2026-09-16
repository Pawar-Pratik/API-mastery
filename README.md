# APIs & REST: Zero → Absolute Authority

> The goal is not "I can build endpoints."
> The goal is: **you are the person who decides what the contract looks like** — and can defend every decision in a design review, at 3am during an incident, and in a Google/Amazon interview loop.

---

## Why most API knowledge is shallow (and how this fixes it)

Ask ten engineers "what is REST?" and nine will say *"it's when you use HTTP with GET/POST/PUT/DELETE and JSON."* That's not REST — that's a **description of the tools**, not the architecture. Then an interviewer asks:

- *"Is `POST /users/123/deactivate` RESTful? Should it be?"*
- *"Your client retried a `POST` after a timeout. Did the customer get charged twice? How do you know?"*
- *"A client sends a field you don't recognise. Should you 400? Why?"*
- *"You have 4 million rows. Design the list endpoint."*
- *"Why is `PUT` idempotent but `POST` isn't — and what breaks if you get it backwards?"*

…and the illusion collapses, because those questions aren't about syntax. They're about **the contract**, and contracts are the actual job.

This course teaches the mechanisms. Almost every "API best practice" turns out to be derived from a small number of boring, understandable facts:

1. **The network is unreliable** → timeouts, retries, and therefore idempotency
2. **The network is slow** → caching, compression, pagination, batching
3. **Clients deploy on their own schedule, not yours** → versioning, tolerant readers, deprecation
4. **Anyone can send anything** → validation, authentication, authorization, rate limits
5. **HTTP already solved most of this in 1999** → and you're probably ignoring it

Internalise those five and API "best practices" stop being a list to memorise. They become things you can *re-derive on the spot* — which is precisely what an interviewer is testing when they change the constraints mid-question.

---

## What you'll be able to do at the end

- Take a vague product requirement and produce a **complete API design** — resources, URLs, methods, status codes, payload shapes, errors, auth, pagination, rate limits, versioning policy — and justify each choice
- Read a raw HTTP exchange and explain **every header**, and what each one costs
- Answer *"REST or gRPC or GraphQL?"* by naming the constraints that decide it, not by preference
- Diagnose a production API problem from symptoms: intermittent 502s, duplicate charges, stale reads, a client that broke after your deploy
- Build a **real, hosted, documented payments API** with idempotency, webhooks, OAuth2, rate limiting, and OpenAPI docs
- Survive the follow-up questions, which is the part that actually separates levels

---

## The project: **Ledger**

A payments API — merchants create payment intents, money moves into a wallet ledger, webhooks fire, refunds and payouts happen, everything is audited.

Full blueprint: **[Ledger blueprint](07-project/28-ledger-blueprint.md)** · Build log: **[Ledger build log](07-project/29-ledger-build-log.md)**

Skim the blueprint after Lesson 06. Start building at Lesson 25.

---

## Setup

| Tool | Why |
|---|---|
| **Node 20+ / 22 LTS** | Runtime for all examples |
| **TypeScript 5.x** | All code is typed; matches the TS track |
| **curl** (built in) | **Non-negotiable.** You will learn HTTP with `curl -v`, not with Postman. Postman hides the thing you're trying to learn |
| **Bruno** or **Postman** | For collections you keep. Bruno is file-based, so it commits to git |
| **Docker Desktop** | Postgres + Redis from Module 4 on |
| **Wireshark** *(optional, once)* | Lesson 02 asks you to watch one TCP handshake with your own eyes. Do it once and TLS/HTTP stops being abstract forever |
| **httpbin.org / httpstat.us** | Free endpoints for experiments |

> Every lesson's shell commands run in **PowerShell or Git Bash on Windows**. Where `curl` differs on Windows PowerShell (it aliases to `Invoke-WebRequest`), the lesson tells you to use `curl.exe`.

---

## Table of contents

### Module 1 — Foundations: what actually happens
> Skip nothing here. Everyone who finds API design "opinion-based" is missing this module. Design rules are *consequences* of the mechanics below.

| # | Lesson | What you'll be able to do |
|---|---|---|
| 01 | [What an API really is](01-foundations/01-what-an-api-really-is.md) | Explain APIs as contracts and coupling; classify any API along five real axes (not the fake "3 types" list) |
| 02 | [The journey of one request: DNS → TCP → TLS → HTTP](01-foundations/02-journey-of-a-request.md) | Narrate every millisecond from `fetch()` to response, and know which stage your latency is in |
| 03 | [HTTP in full: methods, status, semantics](01-foundations/03-http-methods-and-status.md) | Choose the right method and status code every time; explain safe / idempotent / cacheable precisely |
| 04 | [Headers & payloads: the anatomy of a message](01-foundations/04-headers-and-payloads.md) | Explain any header on sight; know exactly what a "payload" is and how it's framed, encoded and negotiated |
| 05 | [JSON vs XML vs Protobuf vs the rest](01-foundations/05-data-formats.md) | Defend your serialization choice with numbers, and name JSON's five real limitations |

### Module 2 — REST design: the part you're judged on
| # | Lesson | What you'll be able to do |
|---|---|---|
| 06 | [REST, actually: constraints & maturity](02-rest-design/06-rest-constraints.md) | Explain the 6 constraints and what each buys you; place any API on the Richardson scale |
| 07 | [Resource modelling & URL standards](02-rest-design/07-resource-modelling-and-urls.md) | Design URLs a senior engineer approves on first read; handle actions that aren't nouns |
| 08 | [Collections: pagination, filtering, sorting, search](02-rest-design/08-collections-and-pagination.md) | Build a list endpoint that survives 4M rows and concurrent writes |
| 09 | [Writes: POST vs PUT vs PATCH, bulk & concurrency](02-rest-design/09-writes-patch-and-bulk.md) | Implement partial updates correctly (JSON Merge Patch), do bulk ops, prevent lost updates |
| 10 | [Errors & status codes that clients can act on](02-rest-design/10-errors-and-problem-details.md) | Design an error contract (RFC 9457) and pick between 400/401/403/404/409/422/429 without hesitating |
| 11 | [Versioning, evolution & unknown fields](02-rest-design/11-versioning-and-evolution.md) | Answer "should an extra parameter break my API?" three different correct ways, and pick one |

### Module 3 — Security (where APIs actually get breached)
| # | Lesson | What you'll be able to do |
|---|---|---|
| 12 | [The authentication landscape](03-security/12-authentication-landscape.md) | Choose between API keys, Basic, sessions, JWT, mTLS and HMAC signing — with reasons |
| 13 | [Sessions, JWT & token lifecycle](03-security/13-sessions-and-jwt.md) | Implement access + refresh with rotation and revocation; explain JWT's honest trade-off |
| 14 | [OAuth 2.1 & OpenID Connect](03-security/14-oauth2-and-oidc.md) | Draw the auth-code + PKCE flow from memory; know which grant to use where and why implicit died |
| 15 | [Authorization, multi-tenancy & BOLA](03-security/15-authorization-and-multitenancy.md) | Build tenant isolation that can't be bypassed; RBAC vs ABAC vs ReBAC |
| 16 | [Hardening: OWASP API Top 10 in practice](03-security/16-owasp-and-hardening.md) | Attack your own API, then fix it: injection, mass assignment, SSRF, CORS mistakes, secrets |

### Module 4 — Production concerns
| # | Lesson | What you'll be able to do |
|---|---|---|
| 17 | [Caching & HTTP performance](04-production/17-caching-and-performance.md) | Use ETag / `Cache-Control` / CDNs correctly; explain `no-cache` vs `no-store` vs `must-revalidate` |
| 18 | [Reliability: timeouts, retries, idempotency, circuit breakers](04-production/18-reliability-and-idempotency.md) | Design a retryable write; implement idempotency keys; stop a retry storm |
| 19 | [Rate limiting, quotas & fairness](04-production/19-rate-limiting.md) | Implement token bucket / sliding window in Redis; return correct 429s with headers |
| 20 | [Observability & debugging APIs](04-production/20-observability.md) | Add correlation IDs, structured logs, RED metrics, traces; debug from symptoms to root cause |

### Module 5 — Beyond REST (and when to leave it)
| # | Lesson | What you'll be able to do |
|---|---|---|
| 21 | [GraphQL, honestly](05-beyond-rest/21-graphql.md) | Build and secure a schema; explain N+1, depth limits, persisted queries, and why caching is hard |
| 22 | [gRPC & Protobuf](05-beyond-rest/22-grpc-and-protobuf.md) | Write `.proto` contracts, evolve them safely, explain streaming and why browsers need grpc-web |
| 23 | [Real-time: WebSockets, SSE & webhooks](05-beyond-rest/23-realtime-and-webhooks.md) | Choose between polling/SSE/WebSocket; ship webhooks with signatures, retries and replay protection |
| 24 | [Choosing the right API style](05-beyond-rest/24-choosing-an-api-style.md) | Answer the "REST vs GraphQL vs gRPC" question like an architect, plus SOAP/tRPC/OData context |

### Module 6 — Craft
| # | Lesson | What you'll be able to do |
|---|---|---|
| 25 | [Contract-first with OpenAPI](06-craft/25-openapi-contract-first.md) | Write the spec first, generate types/clients/mocks/validation from it |
| 26 | [Testing APIs properly](06-craft/26-testing-apis.md) | Unit → integration → contract → load; test the failure modes, not the happy path |
| 27 | [The complete API checklist & code review guide](06-craft/27-api-review-checklist.md) | Review anyone's API in 15 minutes and produce specific, defensible feedback |

### Module 7 — The project
| # | Lesson | What you'll be able to do |
|---|---|---|
| 28 | [Ledger blueprint](07-project/28-ledger-blueprint.md) | Understand the whole system: domain, data model, full API surface, architecture |
| 29 | [Ledger build log](07-project/29-ledger-build-log.md) | Build it phase by phase, each phase mapped to lessons |

### Module 8 — Interview mastery
| # | Lesson | What you'll be able to do |
|---|---|---|
| 30 | [Rapid-fire master Q&A (200+)](08-interview/30-rapid-fire-master-qa.md) | Revise the entire track in 2 hours; drill until answers are reflexes |
| 31 | [API design round playbook](08-interview/31-design-round-playbook.md) | Run a 45-minute API design interview with a repeatable framework |
| 32 | [Scenario gauntlet: "what would you do if…"](08-interview/32-scenario-gauntlet.md) | Handle 40 production scenarios — duplicate charges, breaking clients, 3am incidents |

---

## The 20 questions that define mastery of this track

If you can answer all twenty **cold, out loud, in under two minutes each**, you're done. Test yourself on these now, before Lesson 01, and note the score. Test again at the end.

1. What is the difference between *safe*, *idempotent* and *cacheable*? Give a method that is one but not the others.
2. Your client sends `POST /payments`, the connection times out, and it retries. How do you guarantee the customer isn't charged twice?
3. A client sends `{"amount": 100, "currncy": "USD"}` — a typo. What should the API do, and what does the answer depend on?
4. Why is `PATCH` not idempotent by default, and how would you make it so?
5. When would you return 422 instead of 400? 409 instead of 400? 403 instead of 404?
6. You have 4 million payments. Why is `?page=40000&limit=100` broken, and what replaces it?
7. Explain `Cache-Control: no-cache` vs `no-store` vs `private` vs `must-revalidate`.
8. What exactly does a CORS preflight check, and why does the browser do it at all?
9. Where do you store a JWT in a browser, and what attack does each option expose you to?
10. How do you revoke a JWT? What have you given up by needing to?
11. Why did OAuth2's implicit flow get deprecated? What does PKCE actually prevent?
12. What is BOLA and why is it the most common API vulnerability in the wild?
13. Design the versioning strategy for an API with 5,000 integrators. Defend it.
14. Your API calls a third party that starts taking 30 seconds. Describe everything that fails, in order.
15. When is GraphQL the wrong choice? When is gRPC the wrong choice?
16. How does a webhook receiver verify that the request genuinely came from you?
17. What is the difference between rate limiting and throttling and a quota? Where does each live?
18. Two clients `PUT` the same resource simultaneously. How do you prevent a lost update?
19. Why is JSON a bad choice for a high-throughput internal service, and what are the actual numbers?
20. Your p99 latency is 4s but p50 is 40ms. Walk me through your investigation.

Every one of these is answered somewhere in this track, and the lesson is named in [Lesson 30](08-interview/30-rapid-fire-master-qa.md).

---

## A note on "types of API"

If you've searched this before, you've seen the blog-post answer: *"There are 4 types of API: public, partner, private, composite."* That's one axis, and it's the least useful one.

APIs are classified along **five independent axes** at once, and confusing them is a common interview stumble:

| Axis | Values |
|---|---|
| **Audience** | Public / open · Partner · Private / internal |
| **Architectural style** | REST · GraphQL · gRPC · SOAP · WebSocket · SSE · Webhook · tRPC · OData |
| **Protocol / transport** | HTTP/1.1 · HTTP/2 · HTTP/3 · TCP · AMQP · MQTT |
| **Interaction pattern** | Request–response · Server push · Bidirectional stream · Fire-and-forget · Long-running with callback |
| **Layer** | Web/remote API · Library/SDK API · OS API · Database API · Hardware API |

"REST vs SOAP" compares styles. "Public vs private" compares audiences. **They are not alternatives to each other** — a private REST API over HTTP/2 with webhooks is one API sitting at one point on all five axes. [Lesson 01](01-foundations/01-what-an-api-really-is.md) builds this map properly, and [Lesson 24](05-beyond-rest/24-choosing-an-api-style.md) tells you what the industry actually uses where.

---

Start here → **[Lesson 01: What an API really is](01-foundations/01-what-an-api-really-is.md)**
