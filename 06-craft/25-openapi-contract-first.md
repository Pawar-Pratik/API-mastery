# Lesson 25 — Contract-first with OpenAPI

> **Why this lesson exists:** everything in Modules 2–5 was *what* to design. This is *how to make the design real and keep it real* — because the most common failure of a well-designed API is **drift**: the docs say one thing, the code does another, and the client team discovers the difference in production. Contract-first is the discipline that makes drift structurally impossible, and *"how do you keep docs in sync with code?"* is a routine interview question with a specific right answer.

**Time:** ~80 minutes · **Prereq:** Modules 2–4

---

## 1. The idea in one sentence

> **Write the contract first, then generate everything you can from it — types, validation, clients, mocks, docs and tests — so that the contract and the implementation cannot disagree without something failing loudly.**

---

## 2. Code-first vs contract-first

| | **Code-first** (annotate the code, generate the spec) | **Contract-first** (write the spec, generate the code) |
|---|---|---|
| Source of truth | The implementation | **The spec** |
| Design happens | While coding | **Before coding, and reviewably** |
| Can front-end start early? | No — wait for the backend | **Yes — mock from the spec on day one** |
| Drift risk | Low (generated from code) | Low (code validated against spec) |
| Design quality | Whatever fell out of the code | Reviewed as an artifact |
| Typical tools | `tsoa`, `springdoc`, `FastAPI`, decorators | `openapi-typescript`, `oapi-codegen`, Prism, Spectral |

**Code-first is not wrong** — `springdoc-openapi` and FastAPI produce excellent specs, and if your team's discipline is good it works fine. Its real weakness is *sequencing*: the contract only exists once someone has implemented it, so nobody can review the design before it's built, and the front end waits.

**Contract-first's payoff is parallelism and reviewability:**

```
Day 1:  Write and review openapi.yaml  ← the design review happens HERE, on a diff
Day 2:  Backend starts on handlers      │  Frontend starts against a generated mock
        Generated types keep both honest│  Generated client, fully typed
Day 5:  Integrate — and it just works, because both built against the same contract
```

That "design review on a diff" is the underrated part. **A PR that changes `openapi.yaml` is a reviewable API design change** — you can see a breaking change in the diff, before anyone writes code. Compare that with reviewing a design by reading 400 lines of controller.

> **The pragmatic position to hold in an interview:** *"Contract-first when more than one team consumes the API, because the spec becomes the coordination artifact and lets clients start immediately. Code-first is fine for a single-team internal service — the important thing either way is that the spec is generated or validated in CI, so it can't drift. A hand-maintained spec next to hand-written code is the only genuinely bad option."*

---

## 3. Writing the spec

```yaml
openapi: 3.1.0
info:
  title: Ledger API
  version: "2026-03-14"
  description: Payments API for merchants.
  contact: { name: Ledger Support, url: https://docs.ledger.dev }
servers:
  - { url: https://api.ledger.dev/v1, description: Production }
  - { url: https://api.sandbox.ledger.dev/v1, description: Sandbox }

security:
  - apiKey: []            # applied globally; overridden per-operation where needed

tags:
  - { name: Payments, description: Create and manage payments }
  - { name: Refunds }

paths:
  /payments:
    get:
      operationId: listPayments        # ← becomes the generated client's method name
      summary: List payments
      tags: [Payments]
      parameters:
        - { $ref: '#/components/parameters/Limit' }
        - { $ref: '#/components/parameters/Cursor' }
        - name: status
          in: query
          description: Filter by status. Repeat for multiple values (OR).
          schema:
            type: array
            items: { $ref: '#/components/schemas/PaymentStatus' }
        - name: created
          in: query
          description: 'Range filter, e.g. created[gte]=2026-01-01T00:00:00Z'
          style: deepObject
          schema:
            type: object
            properties:
              gte: { type: string, format: date-time }
              lt:  { type: string, format: date-time }
      responses:
        '200':
          description: A page of payments
          headers:
            RateLimit-Remaining: { $ref: '#/components/headers/RateLimitRemaining' }
          content:
            application/json:
              schema: { $ref: '#/components/schemas/PaymentList' }
        '400': { $ref: '#/components/responses/BadRequest' }
        '401': { $ref: '#/components/responses/Unauthorized' }
        '429': { $ref: '#/components/responses/RateLimited' }

    post:
      operationId: createPayment
      summary: Create a payment
      tags: [Payments]
      parameters:
        - name: Idempotency-Key
          in: header
          required: true                       # ← documented as required, Lesson 18
          description: A unique key per logical operation. Reuse it on retries.
          schema: { type: string, minLength: 8, maxLength: 255 }
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: '#/components/schemas/CreatePaymentRequest' }
            examples:
              minimal:
                summary: Minimum viable payment
                value: { amount_minor: 4999, currency: usd }
              withCustomer:
                value: { amount_minor: 4999, currency: usd, customer: cus_9s2k }
      responses:
        '201':
          description: Payment created
          headers:
            Location: { schema: { type: string }, description: URL of the new payment }
          content:
            application/json:
              schema: { $ref: '#/components/schemas/Payment' }
        '200':
          description: Idempotent replay — an identical request already succeeded
          content:
            application/json:
              schema: { $ref: '#/components/schemas/Payment' }
        '409': { $ref: '#/components/responses/Conflict' }
        '422': { $ref: '#/components/responses/ValidationFailed' }

components:
  securitySchemes:
    apiKey:
      type: http
      scheme: bearer
      description: 'Your secret API key, e.g. `sk_live_...`'

  parameters:
    Limit:
      name: limit
      in: query
      schema: { type: integer, minimum: 1, maximum: 100, default: 20 }
    Cursor:
      name: cursor
      in: query
      description: Opaque cursor from a previous response's `next_cursor`. Do not construct.
      schema: { type: string, maxLength: 512 }

  headers:
    RateLimitRemaining:
      schema: { type: integer }
      description: Requests remaining in the current window

  schemas:
    PaymentStatus:
      type: string
      enum: [requires_payment_method, requires_capture, succeeded, failed, refunded]
      description: |
        New values may be added over time. Clients MUST handle unknown values
        gracefully (treat them as "processing").          # ← Lesson 11, in the contract

    Payment:
      type: object
      required: [object, id, amount_minor, currency, status, created_at]
      properties:
        object:       { type: string, enum: [payment] }
        id:           { type: string, pattern: '^pi_[A-Za-z0-9]+$', examples: [pi_3Nx8kQ2mR] }
        amount_minor:
          type: integer
          format: int64
          minimum: 0
          description: Amount in the currency's smallest unit. 4999 usd = $49.99.
        currency:     { type: string, enum: [usd, eur, gbp, inr] }
        status:       { $ref: '#/components/schemas/PaymentStatus' }
        customer:     { type: [string, 'null'], description: Customer ID, or null }
        created_at:   { type: string, format: date-time }
        metadata:
          type: object
          additionalProperties: { type: string, maxLength: 500 }

    CreatePaymentRequest:
      type: object
      required: [amount_minor, currency]
      additionalProperties: false          # ★ reject unknown fields — Lesson 11
      properties:
        amount_minor: { type: integer, format: int64, minimum: 50 }
        currency:     { type: string, enum: [usd, eur, gbp, inr] }
        customer:     { type: string, pattern: '^cus_[A-Za-z0-9]+$' }
        description:  { type: string, maxLength: 1000 }
        metadata:
          type: object
          additionalProperties: { type: string, maxLength: 500 }

    PaymentList:
      type: object
      required: [object, data, has_more]
      properties:
        object:      { type: string, enum: [list] }
        data:        { type: array, items: { $ref: '#/components/schemas/Payment' } }
        has_more:    { type: boolean }
        next_cursor: { type: [string, 'null'] }

    Problem:                               # RFC 9457 — Lesson 10
      type: object
      required: [type, title, status, code, request_id]
      properties:
        type:       { type: string, format: uri }
        title:      { type: string }
        status:     { type: integer }
        detail:     { type: string }
        instance:   { type: string }
        code:       { type: string, description: Stable machine-readable code. Branch on THIS. }
        request_id: { type: string }
        errors:
          type: array
          items:
            type: object
            required: [field, code]
            properties:
              field:  { type: string }
              code:   { type: string }
              detail: { type: string }

  responses:
    BadRequest:
      description: Malformed request
      content: { application/problem+json: { schema: { $ref: '#/components/schemas/Problem' } } }
    Unauthorized:
      description: Missing or invalid credentials
      headers:
        WWW-Authenticate: { schema: { type: string } }
      content: { application/problem+json: { schema: { $ref: '#/components/schemas/Problem' } } }
    Conflict:
      description: Conflicts with the resource's current state
      content: { application/problem+json: { schema: { $ref: '#/components/schemas/Problem' } } }
    ValidationFailed:
      description: Semantically invalid
      content: { application/problem+json: { schema: { $ref: '#/components/schemas/Problem' } } }
    RateLimited:
      description: Too many requests
      headers:
        Retry-After: { schema: { type: integer } }
      content: { application/problem+json: { schema: { $ref: '#/components/schemas/Problem' } } }
```

### The eight things that make a spec *good* rather than merely valid

1. **`operationId` on every operation.** It becomes the generated client's method name. Without it you get `getPaymentsIdRefundsGet`.
2. **`$ref` everything reusable** — parameters, responses, headers, schemas. A spec with copy-pasted error responses will drift internally.
3. **Document *all* the error responses**, not just the happy path. Half-specified error handling is the most common spec defect, and clients need it most.
4. **`additionalProperties: false` on request bodies.** This is your unknown-field policy, encoded, and it makes generated validators reject typos ([Lesson 11](../02-rest-design/11-versioning-and-evolution.md)).
5. **Real `examples`**, plural, including the awkward cases. They power docs, mocks and tests simultaneously — the single highest-leverage thing you can add.
6. **Semantics in `description`.** The schema can say `integer`; only prose can say *"minor units, 4999 usd = $49.99"* and *"new enum values may be added"*. That's layer 2 from [Lesson 01](../01-foundations/01-what-an-api-really-is.md), and the spec is where it belongs.
7. **Constraints as machine-readable schema** (`minimum`, `maxLength`, `pattern`, `enum`), so validators and clients both enforce them.
8. **Document headers** — `Location`, `Retry-After`, `RateLimit-*`, `WWW-Authenticate`. Undocumented headers are undiscoverable.

> **OpenAPI 3.1 vs 3.0:** use **3.1**. It is fully JSON Schema 2020-12 compatible, which means your spec schemas and your runtime validator can be *the same schemas*. In 3.0 they were a near-miss dialect, so nullable was the awkward `nullable: true` instead of `type: [string, 'null']`, and you couldn't share schemas with standard JSON Schema tooling. That compatibility is the whole reason the generate-everything workflow below actually works.

---

## 4. What you generate from it

This is the payoff. One artifact, seven outputs.

```
openapi.yaml
   ├── TypeScript types            openapi-typescript
   ├── A typed client              openapi-fetch / orval / oazapfts
   ├── Server request validation   express-openapi-validator / fastify plugin
   ├── A mock server               Prism  → the front end starts on day 1
   ├── Docs                        Scalar / Redoc / Stoplight
   ├── Contract tests              Dredd / Schemathesis (property-based!)
   └── Lint + breaking-change CI   Spectral + oasdiff
```

### Types and a typed client
```bash
npx openapi-typescript openapi.yaml -o src/generated/api.d.ts
```
```ts
import createClient from "openapi-fetch";
import type { paths } from "./generated/api";

const api = createClient<paths>({ baseUrl: "https://api.ledger.dev/v1" });

// Fully typed: path, params, body and response all checked against the spec.
const { data, error } = await api.POST("/payments", {
  params: { header: { "Idempotency-Key": crypto.randomUUID() } },
  body: { amount_minor: 4999, currency: "usd" },
  //     ^ a typo here, or a missing required field, is a COMPILE error
});
if (error) console.error(error.code);      // typed as the Problem schema
data?.status;                               // typed as PaymentStatus
```
**This is where the React and TypeScript tracks connect to this one:** the Ledger Console consumes generated types from Ledger's spec, so a backend contract change breaks the front-end build rather than production.

### Server-side request validation from the same spec
```ts
import * as OpenApiValidator from "express-openapi-validator";

app.use(OpenApiValidator.middleware({
  apiSpec: "./openapi.yaml",
  validateRequests: { allowUnknownQueryParameters: false },   // ← Lesson 08's rule, enforced
  validateResponses: process.env.NODE_ENV !== "production",   // dev/CI only: catches drift instantly
  validateSecurity: false,                                    // your own auth middleware does this
}));
```
Two things worth noticing:
- `allowUnknownQueryParameters: false` gives you the reject-unknown-query-params rule for free.
- **`validateResponses` in CI is the anti-drift mechanism.** If a handler returns a field that isn't in the spec, or omits a required one, **the test fails**. That single flag is most of what "contract-first" buys you.

### Mock server — the parallelism unlock
```bash
npx @stoplight/prism-cli mock openapi.yaml --port 4010
curl http://localhost:4010/payments -H "Prefer: example=withCustomer"
curl http://localhost:4010/payments -H "Prefer: code=422"     # force an error response
```
The front end develops against real-shaped data on day one, **including error states** — which teams almost never test against otherwise. That `Prefer: code=422` trick is worth remembering.

### Linting and breaking-change detection in CI
```yaml
# .github/workflows/api-contract.yml
- run: npx @stoplight/spectral-cli lint openapi.yaml --fail-severity=warn
- run: npx oasdiff breaking main-openapi.yaml openapi.yaml --fail-on ERR
- run: npm test        # includes response validation against the spec
```

```yaml
# .spectral.yaml — house rules, enforced
extends: ["spectral:oas"]
rules:
  operation-operationId: error
  operation-tag-defined: error
  oas3-valid-schema-example: error
  # Custom: every operation must document 401 and 429
  ledger-documents-auth-errors:
    given: $.paths[*][get,post,patch,delete]
    then:
      - field: responses.401
        function: truthy
      - field: responses.429
        function: truthy
  ledger-no-additional-properties:
    given: $.components.schemas[?(@.type=='object')]
    message: Request schemas must set additionalProperties:false
    severity: warn
```

**`oasdiff breaking` is the crown jewel of this setup.** It compares your spec against the previous version and fails the build on a breaking change — removing a field, adding a required parameter, narrowing an enum. That converts [Lesson 11](../02-rest-design/11-versioning-and-evolution.md)'s "breaking change" table from a document into a **build gate.** *"I'd enforce compatibility in CI with oasdiff"* is a specific, credible answer to "how do you avoid breaking clients?"

---

## 5. Documentation as a product

For a public API, docs *are* the product — that's the Lesson 01 point that developers evaluate you by them.

| What good docs have | Why |
|---|---|
| **A quickstart that works in under 5 minutes** | If a developer can't make one successful call quickly, they leave |
| **Copy-pasteable `curl` for every endpoint** | The universal first test |
| **Real examples, including error responses** | The happy path is the easy part |
| **A published error-code table with a `retryable` column** | [Lesson 10](../02-rest-design/10-errors-and-problem-details.md) — almost nobody does this, and it's a differentiator |
| **Explicit compatibility policy** (what may change without notice) | [Lesson 11](../02-rest-design/11-versioning-and-evolution.md) |
| **Sandbox/test mode with test credentials** | Nobody wants to charge a real card to try your API |
| **A dated changelog** | The artifact integrators actually trust |
| **Guides, not just reference** | "How do I handle a failed payment?" spans five endpoints |
| **Rate limits, per tier, in numbers** | Integrators design around published limits |

Reference docs can be generated (Scalar, Redoc, Stoplight). **Guides must be written** — and the highest-value guide is always the one about failure: *"handling declines, retries and webhooks"*, not *"creating your first payment"*.

---

## 6. Production rules

| Rule | Why |
|---|---|
| **One spec, in version control, reviewed on PRs** | The design review happens on the diff |
| **Use OpenAPI 3.1** | Real JSON Schema, so spec schemas and runtime validators can be the same thing |
| **Generate types and clients; never hand-write them** | Hand-written clients drift silently |
| **Validate requests against the spec at runtime** | The spec becomes executable, not decorative |
| **Validate *responses* against the spec in CI** | The single most effective anti-drift mechanism |
| **`additionalProperties: false` on every request schema** | Encodes your unknown-field policy |
| **`operationId` on every operation** | Readable generated clients |
| **Document every error response, and every meaningful header** | Undocumented behaviour is undiscoverable |
| **Lint with Spectral, including custom house rules** | Consistency without arguing in every PR |
| **`oasdiff breaking` as a required CI check** | Turns your breaking-change policy into a build gate |
| **Serve the live spec at `/v1/openapi.json`** | Tooling can always fetch the current contract |
| **Mock from the spec so clients start on day one** | The parallelism payoff |
| **Put semantics in `description`** | Units, nullability rules, enum openness — the schema can't say these |

> **Spring equivalent:** `springdoc-openapi` generates the spec from your annotated controllers (code-first) — and you can still run Spectral and `oasdiff` against the generated output in CI, which gets you most of the contract-first safety with a code-first workflow. See `spring boot/05-web/17-openapi-cors-filters-interceptors.md`.

---

## 7. Interview traps

**Q1. "How do you keep documentation in sync with the code?"**
The specific answer: don't keep them in sync — **make them the same artifact.** Either generate the spec from code, or generate/validate the code against the spec. Then the mechanism: `validateResponses` in CI, so a handler returning an undocumented field fails the build. A hand-maintained spec beside hand-written code is the one configuration guaranteed to drift.

**Q2. "Contract-first or code-first?"**
Contract-first when multiple teams consume it (the spec is the coordination artifact, and clients can start immediately against a mock); code-first is fine for a single-team service. **What matters either way is that the spec is machine-checked in CI.**

**Q3. "What do you generate from OpenAPI?"**
Types, clients, request validation, response validation, mocks, docs, contract tests, and breaking-change detection. Naming **response validation and `oasdiff`** is what distinguishes a real answer from a list of buzzwords.

**Q4. "How do you prevent someone shipping a breaking change?"**
`oasdiff breaking` as a required CI check, plus a written definition of "breaking" in the repo, plus consumer-driven contract tests for known clients, plus per-client usage instrumentation for anything you plan to remove.

**Q5. "OpenAPI 3.0 or 3.1?"**
3.1 — it's JSON Schema 2020-12 compatible, so the spec's schemas and your runtime validator can be literally the same schemas. In 3.0 they were a subtly different dialect (`nullable: true` etc.), which forced you to maintain two sources of truth for validation.

**Q6. "How would you let the front end start before the backend exists?"**
Write and review the spec, then run Prism as a mock. The front end builds against generated types and mocked responses — including error paths via `Prefer: code=422`, which teams otherwise never exercise. That parallelism is contract-first's main practical benefit.

**Q7. "What can't OpenAPI express?"**
An important question, because it shows the limits of tooling. It can't express: **semantics** (units, meaning, what "expires_at" is inclusive of), cross-field validation rules ("`end_date` must follow `start_date`" — only awkwardly, via `allOf`/`if-then`), state machines (which transitions are legal), operational guarantees (consistency, ordering, delivery), rate-limit *behaviour* beyond documenting headers, or workflow sequencing. **Those live in prose and in your error contract** — which is why layer 2 of the Lesson 01 model can never be fully generated.

**Q8. "Is API-first the same as contract-first?"**
Related but not identical. **Contract-first** is a development workflow (spec before code). **API-first** is an organisational stance: the API is the product, designed and reviewed before implementation, and your own product consumes the same API you publish — no privileged internal backdoor. Amazon's 2002 mandate is the canonical example.

---

## 8. Build & break

### Build — Ledger's spec, properly
Write `openapi.yaml` covering the full URL set from [Lesson 07 §6](../02-rest-design/07-resource-modelling-and-urls.md). Requirements:
- Every operation has an `operationId`, `summary`, tags, and documented `400/401/403/404/409/422/429` responses via `$ref`.
- All request schemas have `additionalProperties: false`.
- The `Problem` schema is shared everywhere.
- `Idempotency-Key` is a documented **required** header on every money-moving `POST`.
- Cursor and limit parameters are `$ref`'d.
- At least two `examples` per request body, including one that triggers a 422.
- Enum descriptions state that new values may be added.

Then wire the full pipeline:
```bash
npx @stoplight/spectral-cli lint openapi.yaml                 # house rules
npx openapi-typescript openapi.yaml -o src/generated/api.d.ts # types
npx @stoplight/prism-cli mock openapi.yaml --port 4010        # mock
npx oasdiff breaking baseline.yaml openapi.yaml               # compatibility
```
And add `express-openapi-validator` with `validateResponses: true` in test mode.

### Break — five experiments that prove the pipeline works
1. **Cause drift.** Add a field to a handler's response that isn't in the spec. With `validateResponses` on, the test fails. **That failure is the entire value of this lesson** — feel it once.
2. **Ship a breaking change.** Remove a required response field, or add a required request parameter. Watch `oasdiff breaking` fail CI. Then look at the exact message — it names the rule from Lesson 11.
3. **Break the generated client.** Change `amount_minor` from `integer` to `string` in the spec, regenerate types, and watch the Console's TypeScript build fail with a precise error. **That's a production incident that became a compile error.**
4. **Prove `additionalProperties: false` works.** POST `{"amount_minor": 4999, "currency": "usd", "currncy": "eur"}` and confirm you get a 400 naming the unknown field — from the validator, with no code written.
5. **Develop against the mock.** Build a small page against Prism, including the 422 path via `Prefer: code=422`. Then point it at the real API and confirm it works unchanged.

### Explain out loud (90 seconds)
1. Contract-first vs code-first, and what actually matters in both.
2. The seven things you generate from one spec.
3. The two CI checks that prevent drift and breakage.
4. What OpenAPI cannot express, and where that information lives instead.

---

## What's next

The contract is enforced. Now the tests that prove the *behaviour* is right — including the failure modes that matter most and are tested least: retries, concurrency, authorization, and the boundary conditions where APIs actually break.

Next → **[Lesson 26: Testing APIs properly](26-testing-apis.md)**
