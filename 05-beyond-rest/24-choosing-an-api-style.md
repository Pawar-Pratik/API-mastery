# Lesson 24 — Choosing the right API style

> **Why this lesson exists:** *"REST vs GraphQL vs gRPC — which would you use?"* is asked in nearly every senior API interview, and the failure mode isn't ignorance, it's **advocacy**. Candidates pick a favourite and defend it. What's actually being tested is whether you can name the *constraints that decide*, and then commit. This lesson gives you that framework plus the styles we haven't covered yet, so you're never surprised.

**Time:** ~70 minutes · **Prereq:** Lessons 21–23

---

## 1. The idea in one sentence

> **API style is determined by four constraints — who your clients are, what shape your domain is, what your traffic volume is, and what your team can operate — and any answer that doesn't reference those is a preference dressed up as an argument.**

---

## 2. The styles we haven't covered

### SOAP — not dead, load-bearing

```xml
<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
  <soap:Header>
    <wsse:Security><wsse:UsernameToken>...</wsse:UsernameToken></wsse:Security>
  </soap:Header>
  <soap:Body>
    <GetPayment xmlns="http://ledger.dev/"><PaymentId>pi_3Nx8</PaymentId></GetPayment>
  </soap:Body>
</soap:Envelope>
```

XML envelopes, a **WSDL** machine-readable contract, and the WS-* stack. What it genuinely offers that REST doesn't:

| Feature | Why it mattered |
|---|---|
| **WSDL** | A formal, complete, machine-readable contract — before OpenAPI existed |
| **WS-Security** | Message-level signing and encryption. **Survives TLS termination** — the message itself is signed, so an intermediary can't alter it undetectably |
| **WS-AtomicTransaction** | Distributed two-phase-commit transactions across services |
| **WS-ReliableMessaging** | Guaranteed, ordered delivery at the protocol layer |
| **Transport-agnostic** | Runs over HTTP, SMTP, JMS, message queues |

**Where you'll actually meet it:** banking core systems, insurance, healthcare (HL7), telco provisioning, government filings, airline GDS, and enterprise middleware. If you interview at a bank or an enterprise-integration role, *"SOAP is legacy"* is the wrong answer. The right one:

> *"SOAP is verbose and the tooling is dated, but it solved things REST still hasn't standardised — message-level security that survives TLS termination, and formal transactions. In regulated environments where a counterparty mandates a signed, non-repudiable message, that's not legacy, it's a requirement. I'd wrap it behind a REST façade rather than propagate it, but I wouldn't argue against a counterparty's mandate."*

### tRPC — typed RPC for TypeScript monorepos

```ts
// server
export const appRouter = router({
  getPayment: publicProcedure
    .input(z.object({ id: z.string() }))
    .query(({ input, ctx }) => ctx.repos.payments.findById(input.id)),
});
export type AppRouter = typeof appRouter;

// client — full type inference, zero codegen, zero schema file
const payment = await trpc.getPayment.query({ id: "pi_3Nx8" });
//    ^? Payment  — inferred straight from the server's return type
```

**What it optimises:** end-to-end type safety with *no schema, no codegen, no build step*. Rename a server field and the client fails to compile instantly. For a solo dev or a small team on a TS monorepo, the iteration speed is genuinely remarkable.

**What it costs — and this is the important half:** it works by importing the server's **types** into the client, which means:
- **TypeScript only, both ends.** No mobile native, no Python consumer, no third parties.
- **The client and server must version together.** There's no independent contract, so it's structurally unsuited to a public API.
- It is, deliberately, the *opposite* of everything Lesson 01 said about decoupling — and that's fine when client and server are one deployable.

**When to use it:** a Next.js app or TS monorepo where the front end and back end ship together. **When not to:** literally anything with an external consumer.

### JSON-RPC — the minimal one

```json
{ "jsonrpc": "2.0", "method": "getPayment", "params": { "id": "pi_3Nx8" }, "id": 1 }
```
A tiny spec: method, params, id. No schema, no types, batching built in. Where you'll see it: **Ethereum and most blockchain nodes**, the Language Server Protocol (every editor's language tooling), and Bitcoin Core. Worth recognising; rarely worth choosing for a new web API.

### OData & JSON:API — standardised REST conventions

**OData** (Microsoft/SAP) adds a full query language to REST:
```
GET /Payments?$filter=amountMinor gt 1000 and status eq 'succeeded'&$orderby=createdAt desc&$expand=Customer&$select=id,amountMinor
```
**JSON:API** standardises the *document shape* — resource objects, relationships, includes, pagination, errors:
```json
{ "data": [{ "type": "payments", "id": "pi_1",
             "attributes": {...},
             "relationships": { "customer": { "data": { "type":"customers","id":"cus_1" } } } }],
  "included": [{ "type": "customers", "id": "cus_1", "attributes": {...} }] }
```

**The genuine value of both: you stop bikeshedding.** Pagination, filtering, errors and includes are decided by the spec, and generic client libraries just work. **The cost:** verbosity, a learning curve, and — for OData — you've exposed a query language, with all the unbounded-cost problems from [Lesson 21](21-graphql.md).

Worth knowing so you can say *"if the team wants convention over argument, JSON:API is a reasonable pick; I'd otherwise document a house style, which is what most successful public APIs do."*

---

## 3. The decision framework

Four questions, in order. **Each one eliminates options.**

### Question 1 — Who are your clients?

| Client | Eliminates | Points to |
|---|---|---|
| **Third-party developers** | gRPC (needs a proxy), tRPC (TS-only), SOAP (unless mandated) | **REST**; GraphQL if the graph is deep |
| **Browser** | gRPC (native) | REST, GraphQL, SSE/WS |
| **Mobile app you own** | — | REST or GraphQL (bandwidth matters), gRPC if you control both ends |
| **Your own services** | — | **gRPC** (or REST if simple) |
| **A TS monorepo, shipped together** | — | **tRPC** |
| **A bank/insurer/government counterparty** | your preference | **Whatever they mandate** — often SOAP + mTLS |
| **Another company's server, notifying them** | all of the above | **Webhooks** |

### Question 2 — What shape is your domain?

| Shape | Points to |
|---|---|
| **Nouns with lifecycles** (payments, customers, orders) | REST |
| **A deep, interconnected graph** with many client views | GraphQL |
| **Verbs/procedures** (`transcode`, `rebalance`, `sendOtp`) | RPC (gRPC/tRPC) — or resource-ify the *job* ([Lesson 06](../02-rest-design/06-rest-constraints.md)) |
| **Continuous streams** | gRPC streaming, WebSocket, SSE |
| **Documents needing signatures/transactions** | SOAP |

### Question 3 — What are your volume and latency requirements?

| Situation | Points to |
|---|---|
| < 1,000 req/s, human-facing | Anything. Optimise for developer experience |
| 10k+ req/s internal | gRPC — the CPU and bytes are a real bill |
| Bandwidth-constrained clients | GraphQL (fetch less) or Protobuf (smaller) |
| Sub-millisecond internal budget | gRPC, or a binary protocol over a mesh |
| Heavy read traffic that could be cached | **REST** — HTTP caching is free and the others give it up |

### Question 4 — What can your team operate?

The question candidates skip, and often the deciding one.

| Style | Operational burden you're signing up for |
|---|---|
| **REST** | Low. Everyone knows it; every tool supports it |
| **GraphQL** | **Meaningful**: complexity limits, DataLoaders, field-level authz, per-operation observability, a client cache. An under-invested GraphQL API is worse than a plain REST one |
| **gRPC** | **Meaningful**: codegen in the build, a proto repo with compatibility CI, client-side LB or a mesh, and losing casual `curl` debuggability |
| **tRPC** | Very low — but you've coupled client and server releases |
| **SOAP** | High, and the tooling is dated |

> **Saying this out loud is a strong senior signal:** *"the best style for a team is often the one they can operate well, not the one that's technically optimal. I've seen GraphQL adopted without the complexity-limit and observability work, and it was strictly worse than the REST API it replaced."*

---

## 4. The composite answer — what real systems do

Nobody picks one. **The correct architecture uses several, each where it fits**, and describing this shape is the answer to the interview question.

```
                    ┌──────────────────────────────────┐
   Browser / SPA ───┤  REST  (or GraphQL)  ← at the edge│
   Mobile app    ───┤  + SSE / WebSocket for live data  │
   3rd-party     ───┤  + Webhooks outbound              │
                    └────────────┬─────────────────────┘
                                 │ API gateway: TLS, authn, rate limit
                    ┌────────────┴─────────────────────┐
                    │        gRPC between services      │
                    │  payments · ledger · notifications│
                    └────────────┬─────────────────────┘
                                 │
                    ┌────────────┴─────────────────────┐
                    │  Kafka / queues for async events  │
                    └──────────────────────────────────┘
```

Real examples worth citing by name:

| Company | Edge | Internal | Push |
|---|---|---|---|
| **Stripe** | REST | (internal RPC) | Webhooks |
| **GitHub** | REST **and** GraphQL | — | Webhooks |
| **Netflix** | GraphQL federation (studio/UI) | gRPC | — |
| **Shopify** | GraphQL (primary) + REST (legacy) | — | Webhooks |
| **Slack** | REST + Events API | — | Webhooks + WebSocket (Socket Mode) |
| **Uber** | REST | gRPC | — |
| **Kubernetes** | REST (+ watch streams) | gRPC (CRI/CSI/CNI) | Watch API |

**Two patterns to notice and be able to explain:**
1. **GitHub ships both REST and GraphQL, deliberately.** REST for scriptable, simple, cacheable use (`curl` in a CI script); GraphQL for clients that need deep, custom queries. That's not indecision — it's serving two genuinely different consumer profiles. Doubling your API surface is a real cost they chose to pay.
2. **Slack's Socket Mode inverts webhooks.** Customers behind corporate firewalls can't receive inbound HTTP, so Slack lets the *app* open an outbound WebSocket instead. A pure constraint-driven design decision, and a great example of "who initiates?" from [Lesson 23](23-realtime-and-webhooks.md).

---

## 5. The full comparison table

Worth committing to memory — it's the substance behind any answer.

| | REST | GraphQL | gRPC | tRPC | SOAP | WebSocket |
|---|---|---|---|---|---|---|
| **Contract** | OpenAPI (optional) | SDL (required) | Protobuf (required) | TS types | WSDL (required) | none |
| **Transport** | HTTP/1.1–3 | HTTP (POST) | HTTP/2 | HTTP | HTTP/SMTP/JMS | TCP upgraded |
| **Format** | JSON | JSON | Protobuf | JSON | XML | anything |
| **Browser native** | ✅ | ✅ | ❌ | ✅ | ✅ | ✅ |
| **HTTP caching** | ✅ **free** | ❌ | ❌ | ❌ | ❌ | ❌ |
| **Streaming** | SSE/chunked | subscriptions | ✅ **4 modes** | subscriptions | ❌ | ✅ |
| **Payload size** | baseline | smaller (fetch less) | **~0.3×** | baseline | **~1.5×** | small |
| **Codegen quality** | good | excellent | **excellent** | n/a (inferred) | good | n/a |
| **Per-endpoint limits/metrics** | ✅ **free** | ✗ rebuild | ✅ per method | ✗ | ✅ | ✗ |
| **`curl`-able** | ✅ | ~ | ❌ | ~ | ~ | ❌ |
| **Learning curve** | low | medium | medium | low | high | medium |
| **Best for** | public APIs | many clients, deep graph | internal, high volume | TS monorepo | regulated enterprise | bidirectional real-time |

---

## 6. The answers to the actual interview questions

### "REST or GraphQL?"
> *"It depends on the client profile and how much operational work I can fund. GraphQL wins when several clients evolve independently over a deep object graph and over-fetching is measurably hurting — that's the Facebook problem it was built for. REST wins when consumers are third parties who expect it, when HTTP caching or CDN offload matters, or when I need precise per-endpoint rate limits and metrics. And GraphQL isn't free: I'd need complexity limits, DataLoaders, field-level authorization and per-operation observability before shipping it. For a payments API like Ledger I'd choose REST, and get most of GraphQL's benefit from `?expand=` and sparse fieldsets."*

### "REST or gRPC?"
> *"Different sides of the same system. gRPC internally: 3× smaller payloads, mandatory schemas, generated clients, built-in deadlines and declarative retries — all of which matter at volume and across languages. REST at the edge: browsers can't speak gRPC without a proxy, third parties expect REST, and I keep HTTP caching. The pattern I'd actually build is one `.proto` as the source of truth with `grpc-gateway` generating a REST façade — one contract, two protocols, no drift."*

### "Would you use GraphQL for a public API?"
> *"You can — GitHub and Shopify do — but it's a bigger commitment than people expect. You're exposing an unbounded query planner to strangers, so complexity budgets, per-caller cost accounting and abuse monitoring become mandatory. GitHub meters its GraphQL API by calculated point cost rather than request count, which tells you how real that problem is. For most public APIs REST is the better default, and I'd add GraphQL later if I had evidence that clients were struggling with over-fetching."*

### "How would you migrate a REST API to GraphQL?"
> *"Incrementally, and probably not entirely. I'd put GraphQL alongside REST — same service layer, second transport — starting with one read-heavy screen that demonstrably over-fetches. Measure the improvement in requests and bytes. Keep REST for third parties and for anything where caching matters. What I wouldn't do is a big-bang migration: you'd rebuild caching, rate limiting and observability all at once, with no way to attribute the regressions."*

### "Design the API layer for a new product." *(the open-ended one)*
Answer with the framework, out loud, in this order:
1. **Who are the clients?** Browser + mobile + third-party integrations + internal services → so I need something at the edge and something internal.
2. **Edge:** REST, versioned, OpenAPI-first, cursor-paginated, `problem+json` errors, idempotency keys on writes.
3. **Live data to the browser:** SSE (upgrade to WebSocket only if bidirectional turns out to be needed).
4. **Notifying third parties:** webhooks with HMAC signing, backoff, dead-lettering, and a pollable event log.
5. **Internal:** REST until service count and volume justify gRPC; then gRPC with a proto repo and compatibility CI.
6. **Cross-cutting:** gateway for TLS/authn/rate limiting; object-level authorization in the services; OpenTelemetry throughout.
7. **What I'd revisit:** *"if the front end starts needing three round trips per screen, that's my signal to add `?expand=` and then, if it persists, GraphQL for the first-party client only."*

That last point — naming the **trigger** that would change your decision — is the single strongest thing you can add. It shows the decision is reasoned rather than dogmatic.

---

## 7. Anti-patterns (each one is a real system someone shipped)

| Anti-pattern | Why it's wrong |
|---|---|
| **GraphQL because it's modern** | Without complexity limits and per-operation observability you've built a slower, less safe REST API with extra steps |
| **gRPC for a public third-party API** | Your integrators can't `curl` it, can't use a browser, and must run codegen. Adoption dies |
| **REST for a genuinely bidirectional real-time feature** | Polling a chat endpoint at 1 Hz is worse than a WebSocket in every dimension |
| **tRPC for an API with external consumers** | You've coupled client and server releases, permanently |
| **WebSocket for one-way push** | You've hand-written reconnection, resume, auth and heartbeats that SSE gives free |
| **`POST /api` with an `action` field** | Level 0. No caching, no method semantics, no per-operation limits or metrics ([Lesson 06](../02-rest-design/06-rest-constraints.md)) |
| **One style for everything** | The edge and the interior have genuinely different constraints |
| **Migrating styles to fix a design problem** | A badly-modelled domain is badly modelled in every style. GraphQL will not fix your resource model |

That last row is worth saying in an interview, because it's the deepest point in the lesson: **style choice is a smaller lever than domain modelling.** A well-designed REST API beats a poorly-modelled GraphQL one every time.

---

## 8. Interview traps

**Q1. "Which is best: REST, GraphQL or gRPC?"**
Refuse the premise politely and name the constraints: client profile, domain shape, volume, and team capacity. Then commit to a recommendation for *their* scenario. **Never** answer with a favourite.

**Q2. "Is REST dead?"**
No — it's ~80% of public web APIs and rising in absolute terms. What changed is that it's no longer the *only* answer: gRPC took internal traffic, GraphQL took client-flexible products, and REST kept the edge. *"Boring and universally understood is a feature for a public contract."*

**Q3. "Is SOAP dead?"**
No, and this question is a trap for people who only know modern stacks. It's load-bearing in banking, insurance, healthcare and government, because it standardised message-level security that survives TLS termination, and formal transactions. Wrap it, don't propagate it, and don't argue with a counterparty's mandate.

**Q4. "Why can't you just use GraphQL for everything?"**
You'd lose HTTP caching, per-endpoint rate limits, per-endpoint metrics, meaningful status codes and predictable query cost — then rebuild all five. Plus it's a poor fit for file uploads, binary streaming, and high-volume service-to-service calls.

**Q5. "What would make you change a style decision?"**
The best question they can ask, because it tests whether your reasoning is real. Concrete triggers: *"three or more round trips per screen and rising → add `?expand=`, then GraphQL for the first-party client. Internal JSON parsing showing up in CPU profiles → gRPC for the hottest paths. Clients polling more than ~4× per minute → SSE. A third party asking for guaranteed delivery → webhooks with a pollable event log."*

**Q6. "You inherit a `POST /api` Level-0 API with 200 consumers. What do you do?"**
Not a rewrite. Strangler-fig it: put the new REST surface alongside, migrate consumers endpoint by endpoint with instrumented per-client usage ([Lesson 11](../02-rest-design/11-versioning-and-evolution.md)), keep the old one working, and retire it only when usage is zero. Say explicitly that **you'd fix the domain model as you go**, because otherwise you've just re-shaped the same bad contract.

**Q7. "What does the API style choice NOT fix?"**
A badly-modelled domain, missing authorization, absent idempotency, no observability, and an unversioned contract. **These matter far more than the style**, and every one of them is style-independent. Ending an architecture answer here is a genuinely memorable move.

---

## 9. Build & break

### Build — the decision document
For Ledger, write `docs/api-architecture.md`. One page, and it's the artifact an architecture review actually wants:

```markdown
## Client profiles
| Client | Style | Reason |
|---|---|---|
| Ledger Console (React SPA) | REST + SSE | First-party; SSE for the live payment feed |
| Merchant server integrations | REST + webhooks | Third-party developers; REST is the expectation |
| Platform partners | REST + OAuth 2.1 | Delegated access on behalf of merchants |
| Internal services (ledger, notifications, risk) | REST now, gRPC when >5 services or >5k rps | Don't pay gRPC's operational cost before it's earned |

## Explicitly rejected
- **GraphQL** — consumers are third-party developers who expect REST; reads are cacheable;
  we need precise per-endpoint rate limits. Revisit if the Console needs >3 round trips per screen.
- **gRPC at the edge** — browsers need a proxy; third parties would need codegen.
- **tRPC** — the Console and API must be independently deployable.
- **WebSocket** — the Console needs one-way push only; SSE gives reconnect + resume for free.

## Triggers that would change these decisions
- Console p75 screen load >2s attributable to round trips → add ?expand=, then reconsider GraphQL
- JSON parse time >5% of service CPU → gRPC for the hot internal path
- A partner requiring signed, non-repudiable messages → evaluate JWS-signed payloads (not SOAP)
```

The "explicitly rejected" and "triggers" sections are what make this a good document rather than a decision log. **Most engineers never write down what they rejected and why**, which is why the same argument gets relitigated every six months.

### Build — one endpoint, five ways
Implement `getPayment` as: REST, GraphQL, gRPC, tRPC, and JSON-RPC. All on the same service layer. Then measure and record:

| | Bytes on the wire | p50 latency | Lines of client code | `curl`-able? | Cacheable? |
|---|---|---|---|---|---|
| REST | | | | | |
| GraphQL | | | | | |
| gRPC | | | | | |
| tRPC | | | | | |
| JSON-RPC | | | | | |

**That table, filled in with your own numbers, is worth more in an interview than any amount of reading.** *"I measured it"* ends arguments.

### Explain out loud (2 minutes)
1. The four questions of the framework, in order.
2. The composite architecture diagram, and why the edge and interior differ.
3. Your answer to "REST or GraphQL for a payments API", with the trigger that would change it.
4. Why SOAP isn't dead.
5. What style choice doesn't fix.

---

## Module 5 complete — checkpoint

- [ ] The two problems GraphQL solves, and the four things it takes away
- [ ] Depth vs complexity limits, and why you need both
- [ ] Why DataLoader must be per request (and the leak if it isn't)
- [ ] Why Protobuf is smaller/faster — two mechanisms
- [ ] The Protobuf evolution rules, and why `reserved` matters
- [ ] Why browsers can't speak gRPC
- [ ] The gRPC load-balancing gotcha
- [ ] The five real-time options and the two questions that pick one
- [ ] Why SSE is the default for one-way push, and its HTTP/2 prerequisite
- [ ] The four hard problems of WebSockets
- [ ] The full webhook architecture, including ordering and duplicates
- [ ] The four-question style framework, and a trigger that would change your answer
- [ ] Why SOAP is still load-bearing

---

## What's next

Module 6 is craft: how you make all of this *repeatable*. Contract-first development with OpenAPI — writing the spec before the code and generating types, clients, mocks, validation and docs from it, so the contract can't drift from the implementation.

Next → **[Lesson 25: Contract-first with OpenAPI](../06-craft/25-openapi-contract-first.md)**
