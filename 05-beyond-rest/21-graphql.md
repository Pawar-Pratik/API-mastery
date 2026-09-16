# Lesson 21 — GraphQL, honestly

> **Why this lesson exists:** GraphQL is either "the future of APIs" or "a solved problem people re-broke," depending on who you ask — and both camps are arguing from experience. You need the honest version: the real problem it solves, the four things it takes away, and the specific operational work you must do that REST gave you free. *"When would you not use GraphQL?"* is the interview question, and it filters hard.

**Time:** ~85 minutes · **Prereq:** Modules 1–4

---

## 1. The idea in one sentence

> **GraphQL moves the decision of *what data to return* from the server to the client — and pays for it by giving up every HTTP-level benefit that depends on the server knowing what a request will do.**

---

## 2. The problem it actually solves

Two real problems, and it's worth being precise because GraphQL is often adopted for reasons it doesn't address.

### Problem 1 — Over-fetching
Your mobile app's payment list needs `id`, `amount`, `status`. `GET /v1/payments` returns 40 fields per payment including nested customer objects and metadata. On a phone on 3G you've paid for 37 fields you'll never render.

REST's answers: sparse fieldsets (`?fields=`) and purpose-built endpoints (`/payments/summary`). Both work; both are manual and multiply with each screen.

### Problem 2 — Under-fetching / the client-side N+1
The dashboard needs, per payment: the customer's email, the payment method's last4, and the refund count.

```
GET /v1/payments?limit=20              → 1 request
GET /v1/customers/{id}       × 20      → 20 requests
GET /v1/payment-methods/{id} × 20      → 20 requests
GET /v1/payments/{id}/refunds × 20     → 20 requests
                                      = 61 requests
```

At 100ms RTT and HTTP/1.1's 6-connection limit, that's several seconds. REST's answers: `?expand=`, a composite/BFF endpoint, or a purpose-built `/dashboard/payments` endpoint.

**The pattern:** REST *can* solve both, but the solution is a server-side endpoint per client need. With three clients (web, iOS, Android) evolving on different schedules, you end up maintaining a growing set of bespoke endpoints — and every new screen is a backend ticket.

GraphQL's proposition: **one schema, and the client writes the query.**

```graphql
query DashboardPayments {
  payments(first: 20, filter: { status: SUCCEEDED }) {
    edges {
      node {
        id
        amountMinor
        status
        customer { email }
        paymentMethod { last4 brand }
        refunds { totalCount }
      }
    }
    pageInfo { hasNextPage endCursor }
  }
}
```

One round trip, exactly the fields needed, **no backend change when the screen changes.** That last clause is the real value: it decouples front-end iteration from backend deploys. That's why Facebook built it, and why it wins in products with many screens over a deep object graph — GitHub, Shopify, Meta.

---

## 3. The mechanism

```
POST /graphql
Content-Type: application/json

{ "query": "...", "variables": { "first": 20 }, "operationName": "DashboardPayments" }
```

**One endpoint. Always `POST`. Always `200`** (even for errors — see §5). That single fact is the origin of every trade-off in this lesson.

### The schema is the contract

```graphql
type Payment implements Node {
  id: ID!                          # ! = non-null
  amountMinor: Int!
  currency: Currency!
  status: PaymentStatus!
  customer: Customer               # nullable: a payment may have no customer
  paymentMethod: PaymentMethod
  refunds(first: Int = 10, after: String): RefundConnection!
  createdAt: DateTime!
  metadata: JSON
}

enum PaymentStatus { REQUIRES_PAYMENT_METHOD REQUIRES_CAPTURE SUCCEEDED FAILED REFUNDED }

type Query {
  payment(id: ID!): Payment
  payments(first: Int = 20, after: String, filter: PaymentFilter): PaymentConnection!
  me: User!
}

type Mutation {
  createPayment(input: CreatePaymentInput!): CreatePaymentPayload!
  refundPayment(input: RefundPaymentInput!): RefundPaymentPayload!
}

type Subscription {
  paymentUpdated(merchantId: ID!): Payment!
}
```

Three operation types: **Query** (read), **Mutation** (write, executed serially), **Subscription** (a stream, usually over WebSocket).

### Resolvers, and where the danger lives

Each field has a resolver. **The engine calls them per field, per object** — and that's the source of GraphQL's signature performance problem.

```ts
const resolvers = {
  Query: {
    payments: (_p, args, ctx) => ctx.repos.payments.list(args),   // tenancy from ctx, never args
  },
  Payment: {
    // ❌ called once PER PAYMENT → 20 payments = 20 queries. The N+1, structurally built in.
    customer: (payment, _a, ctx) => ctx.db.customers.findById(payment.customerId),
  },
};
```

**Why this is worse than REST's N+1:** in REST, an N+1 is a bug in *your* handler that you can see and fix. In GraphQL it's the *execution model* — the engine genuinely calls `Payment.customer` twenty times, because it has no idea those twenty calls could be one query.

The fix is **DataLoader**: batch and dedupe within a single tick.

```ts
import DataLoader from "dataloader";

/** ONE loader instance PER REQUEST. Never global — a global loader caches across
 *  users, which is a cross-tenant data leak. */
export function makeLoaders(merchantId: string) {
  return {
    customerById: new DataLoader<string, Customer | null>(async ids => {
      // One query for all ids collected during this tick — and scoped to the tenant.
      const rows = await db.customers.findMany([...ids], merchantId);
      const byId = new Map(rows.map(c => [c.id, c]));
      return ids.map(id => byId.get(id) ?? null);        // MUST return in the same order
    }),
    refundCountByPaymentId: new DataLoader<string, number>(async ids => {
      const rows = await db.query(
        `SELECT payment_id, count(*) AS n FROM refunds
          WHERE payment_id = ANY($1) AND merchant_id = $2 GROUP BY payment_id`,
        [[...ids], merchantId]);
      const byId = new Map(rows.map(r => [r.payment_id, Number(r.n)]));
      return ids.map(id => byId.get(id) ?? 0);
    }),
  };
}

const resolvers = {
  Payment: {
    customer: (p, _a, ctx) => p.customerId ? ctx.loaders.customerById.load(p.customerId) : null,
    refunds:  (p, _a, ctx) => ctx.loaders.refundCountByPaymentId.load(p.id),
  },
};
```

Two rules that are genuinely load-bearing:
1. **One DataLoader set per request.** A global loader caches across users → merchant B sees merchant A's customer. This is a real, shipped-in-production class of bug.
2. **The batch function must return results in the same order as the keys**, with `null` for misses. Getting this wrong silently returns the wrong object for the wrong key — a data-integrity bug that no type checker catches.

---

## 4. What GraphQL takes away (and what you must rebuild)

This is the section that matters, and it's the answer to *"what are the downsides?"*

| REST gave you free | In GraphQL you must build it |
|---|---|
| **HTTP caching** (`ETag`, `Cache-Control`, CDN, 304s) | Gone. One `POST` URL, body-dependent responses. You rebuild it as normalised client-side caching (Apollo/urql/Relay) + persisted queries + server-side caching |
| **Per-endpoint rate limits** | Gone — every request is `POST /graphql`. You need **query complexity analysis**, because one request can cost 1 unit or 10,000 |
| **Per-endpoint metrics/latency** | Gone. You must instrument per **operation name** and per **resolver**, and enforce that clients send `operationName` |
| **Per-endpoint authorization** | Gone. Authorization moves to **every field**, which is more places to get it wrong ([Lesson 15](../03-security/15-authorization-and-multitenancy.md)) |
| **Meaningful status codes** | Gone. `200` with an `errors` array — so every existing HTTP-aware client, proxy and dashboard reads failures as successes |
| **Predictable query cost** | Gone. A client can write a query that joins your whole database |
| **Debuggability via `curl`/logs** | Degraded. Your access log shows `POST /graphql 200` for everything |
| **File uploads** | Not in the spec. Needs the multipart-request extension, or a separate REST endpoint |

**None of these is fatal. All of them are work.** The honest framing: *"GraphQL trades a set of free HTTP-layer capabilities for client flexibility. If you need the flexibility, the trade is worth it — but you must budget for rebuilding caching, rate limiting, observability and field-level authorization, and teams that skip that budget are the ones with GraphQL horror stories."*

---

## 5. The `200 OK` problem, and partial errors

```json
{
  "data": {
    "payment": { "id": "pi_1", "amountMinor": 4999, "customer": null }
  },
  "errors": [
    { "message": "Not authorized to read customer",
      "path": ["payment", "customer"],
      "extensions": { "code": "FORBIDDEN" } }
  ]
}
```

HTTP `200`. `data` is partially populated. `errors` explains the holes.

The consequences are real:
- Your monitoring shows a 0% error rate during an incident.
- Retry libraries and load balancers see success.
- Every client must check `body.errors` on every response, forever.

**Partial success is genuinely a feature** — one failing field shouldn't kill the whole screen. But you must handle it deliberately:

```ts
// Server: put a stable machine-readable code in extensions. Never leak internals.
throw new GraphQLError("Not authorized to read customer", {
  extensions: { code: "FORBIDDEN", field: "customer" },
});

// Server: strip internal detail in production
formatError: (formatted, error) => {
  logger.error({ err: error }, "graphql_error");                 // full detail to logs
  const code = formatted.extensions?.code ?? "INTERNAL_SERVER_ERROR";
  return code === "INTERNAL_SERVER_ERROR"
    ? { message: "Internal server error", extensions: { code, requestId: current()?.requestId } }
    : formatted;
}

// Metrics: count GraphQL errors explicitly, or your dashboards lie.
metrics.inc("graphql_errors_total", { code, operation: operationName ?? "anonymous" });
```

> **The mature pattern for *expected* failures: put them in the schema.** Errors that are part of your domain (insufficient funds, card declined, payment already refunded) shouldn't be in the `errors` array at all — model them as union result types:
> ```graphql
> union RefundResult = RefundSuccess | InsufficientFunds | AlreadyRefunded | PaymentNotRefundable
> ```
> Now the client's type checker **forces** it to handle every case, and the `errors` array is reserved for genuine exceptions. This is the "errors as data" pattern, and describing it is a strong senior signal — it's the GraphQL equivalent of the discriminated-union discipline you'll learn in [TypeScript Lesson 09](../../TypeScript/README.md).

---

## 6. Security: GraphQL's specific attack surface

GraphQL introduces attacks REST doesn't have. **All five of these are exam material.**

### Attack 1 — Query depth
```graphql
query Bomb {
  payment(id: "pi_1") { customer { payments { edges { node {
    customer { payments { edges { node {
      customer { payments { edges { node { id } } } }   # ...×20 more
  } } } } } } } }
}
```
One request, exponential resolver calls. **Fix: a depth limit (7–10 typical).**

### Attack 2 — Query complexity / breadth
```graphql
query Expensive {
  payments(first: 1000) {
    edges { node { refunds(first: 1000) { edges { node { id } } } } }
  }
}
```
Depth 5, but 1,000,000 nodes. Depth limits don't catch it. **Fix: complexity scoring** — assign each field a cost, multiply by pagination arguments, and reject above a budget.

```ts
import { createComplexityRule, simpleEstimator, fieldExtensionsEstimator } from "graphql-query-complexity";

const complexityRule = createComplexityRule({
  maximumComplexity: 1000,
  estimators: [
    fieldExtensionsEstimator(),          // per-field cost declared in the schema
    simpleEstimator({ defaultComplexity: 1 }),
  ],
  onComplete: (complexity) => metrics.observe("graphql_query_complexity", complexity),
  createError: (max, actual) => new GraphQLError(
    `Query complexity ${actual} exceeds the maximum of ${max}`,
    { extensions: { code: "QUERY_TOO_COMPLEX", max, actual } }),
});
```
```graphql
type Query {
  payments(first: Int = 20): PaymentConnection!
    @complexity(multipliers: ["first"], value: 2)     # cost = 2 × first
}
```
**Then rate limit on complexity, not request count** — that's the GraphQL translation of [Lesson 19](../04-production/19-rate-limiting.md)'s cost-weighting, and it's the correct answer to "how do you rate limit GraphQL?"

### Attack 3 — Introspection and field suggestions
```graphql
query { __schema { types { name fields { name type { name } } } } }
```
Introspection hands an attacker your complete schema, including internal fields and admin mutations. **Disable it in production** for a non-public API. Also disable **"did you mean" field suggestions** — they let an attacker enumerate your schema field by field even with introspection off, which most people don't realise.

### Attack 4 — Batching abuse
The spec allows an array of operations in one request. `[{...}, {...}, × 1000]` — one HTTP request, 1,000 operations, bypassing any per-request limit. Also enables **brute forcing in a single request**:
```graphql
mutation { a: login(pw:"0000"){t} b: login(pw:"0001"){t} c: login(pw:"0002"){t} ... }
```
**Fixes:** disable array batching (or cap it hard), limit **aliases per operation**, and rate limit on complexity rather than requests.

### Attack 5 — Field-level authorization gaps
```graphql
query { payment(id: "pi_mine") { customer { email ssn internalRiskScore } } }
```
You authorized `payment`. Did you authorize `Customer.ssn`? **Every field is an endpoint.** Enforce in resolvers (or a schema directive), never rely on "nobody would query that."

```graphql
type Customer {
  id: ID!
  email: String! @auth(requires: CUSTOMER_READ)
  ssn: String    @auth(requires: PII_READ)
  internalRiskScore: Float @auth(requires: PLATFORM_ADMIN)
}
```

### The production security checklist
```
[ ] Depth limit (7–10)
[ ] Complexity limit + complexity-based rate limiting
[ ] Introspection disabled in production (non-public APIs)
[ ] Field suggestions disabled
[ ] Array batching disabled or capped; alias count capped
[ ] Persisted queries / allowlist for first-party clients  ← the strongest control
[ ] Field-level authorization on every sensitive field
[ ] Per-request DataLoaders (never global)
[ ] Query timeout + resolver-level timeouts
[ ] Max query length in bytes
[ ] Error masking in production (no stack traces, no SQL)
[ ] Cost/latency metrics per operationName; require operationName
```

> **Persisted queries deserve emphasis.** Clients register queries at build time and send a hash at runtime (`{"id": "a3f9c1", "variables": {...}}`). This gives you: an **allowlist** (arbitrary queries become impossible), smaller requests, **and cacheability** — because a hash in a URL is a cache key, so you can use `GET` and get your CDN back. For a first-party app it's close to strictly better, and naming it as "how you get HTTP caching back" is a great answer.

---

## 7. Federation and schema design, briefly

At scale, one team can't own one giant schema. **Apollo Federation** (or schema stitching) lets each service own part of it:

```graphql
# payments service
type Payment @key(fields: "id") {
  id: ID!
  amountMinor: Int!
  customer: Customer                       # a stub reference
}

# customers service
type Customer @key(fields: "id") {
  id: ID!
  email: String!
}
```
A gateway composes the supergraph and routes each field to its owner. It's genuinely useful and genuinely complex — the gateway becomes a critical, latency-adding, single point of failure with its own N+1 problems across service boundaries. Know it exists; don't reach for it before you have several teams.

### Schema conventions worth following
| Convention | Why |
|---|---|
| **Relay-style connections** (`edges`/`node`/`pageInfo`/cursors) | It's the de facto standard; client caches (Relay, Apollo) understand it natively, and it's cursor pagination done right ([Lesson 08](../02-rest-design/08-collections-and-pagination.md)) |
| **`input` types for mutations**, one argument named `input` | Evolvable without changing the signature |
| **Mutation payload types** (`CreatePaymentPayload`) | Room to add fields, and a place for domain errors |
| **Nullable by default; `!` only when truly guaranteed** | A non-null field that errors **nulls out its whole parent** — a cascading failure that surprises everyone |
| **Enums for closed sets** | But remember: adding an enum value is a client-breaking change ([Lesson 11](../02-rest-design/11-versioning-and-evolution.md)) |
| **No versioning; deprecate fields with `@deprecated(reason:)`** | GraphQL's evolution story: add fields, deprecate old ones, and use field-usage metrics to know when removal is safe |

That non-null cascade deserves a line of its own: if `Payment.customer` is `Customer!` and the customer resolver errors, GraphQL cannot return `null` for it, so it nulls the **parent** `Payment` — and if `payments` is `[Payment!]!`, it nulls the entire list. **One failing sub-field can blank the whole screen.** That's why "nullable by default" is the real advice, and it's a favourite gotcha question.

---

## 8. Production rules

| Rule | Why |
|---|---|
| **Depth + complexity limits before you ship** | Without them, one query can take down your database |
| **Rate limit on complexity, not request count** | All requests are one `POST` |
| **DataLoader for every relationship, one set per request** | The N+1 is structural, and a global loader leaks across tenants |
| **Field-level authorization on everything sensitive** | Every field is an endpoint |
| **Persisted queries for first-party clients** | Allowlist + smaller payloads + cacheability |
| **Disable introspection and field suggestions in production** | Free schema reconnaissance otherwise |
| **Disable or cap array batching and aliases** | Bypasses per-request limits; enables single-request brute force |
| **Nullable by default; `!` deliberately** | A non-null error nulls the parent, cascading upward |
| **Model expected failures as union result types** | Reserve `errors` for genuine exceptions; force clients to handle cases |
| **Require `operationName`; alert on cost and latency per operation** | Your only route back to per-endpoint observability |
| **Track field-level usage** | The only safe way to deprecate a field |
| **Query timeout, max query bytes, resolver timeouts** | Bound every dimension |
| **Never trust arguments for tenancy — take it from context** | Same rule as REST, more places to forget it |

---

## 9. Interview traps

**Q1. "What problem does GraphQL solve?"**
Over-fetching and the client-side N+1 — and, more importantly, **decoupling front-end iteration from backend deploys** when several clients evolve independently over a deep object graph. REST can solve both, but with a bespoke endpoint per client need.

**Q2. "When would you NOT use GraphQL?"**
The filter question. Strong answer, with reasons:
- **A public API for third parties.** REST is more familiar, `curl`-able, and cacheable — and you'd be exposing an unbounded query cost surface to strangers.
- **Simple CRUD with one client.** You'd add a schema layer, DataLoaders, complexity analysis and a client cache to solve a problem you don't have.
- **When HTTP caching or CDN offload is a primary requirement.** You'd be rebuilding it.
- **File uploads / binary streaming.** Not what it's for.
- **Service-to-service at high volume.** gRPC is smaller, faster and typed.
- **When the team can't fund the operational work** — complexity limits, field authorization, per-operation observability. An under-invested GraphQL API is worse than a plain REST one.

**Q3. "How do you prevent an expensive query from killing your API?"**
Depth limit **and** complexity scoring (because depth alone misses `first: 1000` breadth), complexity-based rate limiting, query timeouts, and persisted queries as an allowlist for first-party clients. Mention that pagination arguments must multiply the cost.

**Q4. "Why is caching hard in GraphQL?"**
Every request is `POST /graphql` with a body-dependent response, so URL-keyed HTTP caches (browser, CDN, proxy) can't help. Solutions: normalised client-side caching by object ID, persisted queries over `GET` (a hash makes a cache key — this gets your CDN back), and server-side caching at the resolver/DataLoader layer.

**Q5. "What's the N+1 problem in GraphQL and how is DataLoader different from just batching in REST?"**
In REST an N+1 is a bug you wrote and can see; in GraphQL it's the execution model — resolvers genuinely run per object. DataLoader intercepts at the tick boundary, collects keys, and issues one batched query. Then the two rules: per-request instances (or you leak across tenants) and order-preserving batch functions.

**Q6. "Everything returns 200. What's the impact?"**
Monitoring reads a 0% error rate during incidents; retry logic and LBs see success; every client must inspect `body.errors`. Mitigations: count GraphQL errors as explicit metrics keyed by `extensions.code`, and model expected domain failures as union result types so `errors` means "something genuinely broke."

**Q7. "How do you version a GraphQL API?"**
You generally don't. Add fields, mark old ones `@deprecated(reason:)`, and use **field-level usage metrics** to know when zero clients read a field — then remove it. That's a real advantage over REST versioning, and the metrics part is the piece people forget.

**Q8. "How do you do authorization?"**
Per field, not per endpoint — enforced in resolvers or via a schema directive, with tenancy from the request context (never from arguments). Note that GraphQL *increases* your authorization surface, which is why the field-level discipline matters more than in REST.

**Q9. "Should you expose GraphQL publicly?"**
You can (GitHub and Shopify do) but it's a bigger commitment: you're exposing an unbounded query planner to strangers, so complexity limits, per-caller cost accounting and abuse monitoring become mandatory rather than nice-to-have. Note that GitHub's public GraphQL API meters by a **calculated point cost**, not request count — that's the real-world confirmation of the §6 advice.

**Q10. "REST or GraphQL for Ledger?"**
**REST**, and be able to defend it: Ledger's consumers are third-party developers and server integrations who expect REST; payments need idempotency keys, cacheable reads and precise per-endpoint rate limits; and the object graph is shallow. Then the nuance that shows range: *"I'd consider GraphQL for the internal Console API specifically, since it's one first-party client with many screens — but I'd more likely just add `?expand=` to REST, which gets 80% of the benefit for 5% of the operational cost."*

---

## 10. Build & break

### Build — a small, correctly-secured GraphQL layer
Add `/graphql` over Ledger's existing service layer (don't replace REST — this is a learning exercise, and it also mirrors what GitHub actually did).

```ts
import { ApolloServer } from "@apollo/server";
import depthLimit from "graphql-depth-limit";

const server = new ApolloServer({
  typeDefs, resolvers,
  introspection: process.env.NODE_ENV !== "production",
  validationRules: [depthLimit(8), complexityRule],
  formatError,                                     // §5: mask internals, keep codes
  plugins: [
    operationMetricsPlugin(),                      // per-operationName latency + cost
    requireOperationNamePlugin(),                  // anonymous queries are unobservable
  ],
});

// Context: per-request loaders + tenancy. This function is your security boundary.
const context = async ({ req }) => {
  const principal = await authenticate(req);
  return {
    principal,
    repos: scopedRepos(principal.merchantId),      // Lesson 15
    loaders: makeLoaders(principal.merchantId),    // per request, tenant-scoped
  };
};
```

Implement: `Query.payments` (Relay connection over your existing cursor pagination), `Payment.customer` and `Payment.refunds` via DataLoader, `Mutation.refundPayment` returning a **union result type**, and `@auth` on two sensitive fields.

### Break — six experiments
1. **The structural N+1.** Resolve `Payment.customer` directly, query 20 payments with `customer { email }`, and count DB queries: 21. Add DataLoader: 2. Log both numbers.
2. **The cross-tenant leak.** Make the DataLoader **global** instead of per-request. Query as merchant A, then as merchant B for the same customer ID, and watch B receive A's cached customer. **This is the most important experiment in the lesson.**
3. **The depth bomb.** Write a 20-level nested query. Watch latency (or your process) collapse. Add `depthLimit(8)`.
4. **The breadth bomb.** `payments(first: 1000) { refunds(first: 1000) { ... } }` — passes an 8-level depth limit and is a million nodes. Add complexity scoring and watch it get rejected.
5. **The alias brute force.** One mutation with 500 aliased `login` attempts. Confirm your per-request rate limiter allows it. Now limit aliases and switch to complexity-based limiting.
6. **The non-null cascade.** Make `Payment.customer` non-null (`Customer!`), then make the resolver throw. Watch the entire `payments` list come back `null`. Change it back to nullable and watch only that field null out.

### Explain out loud (2 minutes)
1. The two problems GraphQL solves, and the deeper organisational one.
2. The four things it takes away, and what you must rebuild.
3. Depth vs complexity limits, and why you need both.
4. Why DataLoader must be per request.
5. Three situations where you'd choose REST instead.

---

## What's next

GraphQL optimises for client flexibility. The next alternative optimises for the opposite end of the spectrum — raw speed and strict contracts between services you control: gRPC and Protobuf, where the schema is mandatory and the bytes are 5× smaller.

Next → **[Lesson 22: gRPC & Protobuf](22-grpc-and-protobuf.md)**
