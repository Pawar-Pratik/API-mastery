# Lesson 10 — Errors & status codes clients can act on

> **Why this lesson exists:** your error responses are the most-read part of your API, because that's what developers stare at when they're stuck at 2am. Yet errors are almost always designed last, by whoever hit the first exception. A good error contract is the cheapest possible improvement to your API's reputation — and *"design your error response"* is a question that instantly separates people who've supported an API from people who've only written one.

**Time:** ~70 minutes · **Prereq:** Lessons 03, 09

---

## 1. The idea in one sentence

> **An error response has exactly one job: tell the client's *code* what to do, and tell the client's *human* how to fix it — and those are two different audiences needing two different fields.**

Most error responses serve neither. `{"error": "Bad Request"}` tells the code nothing to branch on and the human nothing to fix.

---

## 2. The four audiences of an error

Every error response is read by four consumers with genuinely different needs. This framing is the lesson; the format falls out of it.

| Audience | Needs | Field that serves them |
|---|---|---|
| **The client's code** | A stable value to branch on. Must never change | `code` (and the HTTP status) |
| **The developer integrating** | What went wrong, in which field, and how to fix it | `detail`, `errors[].field`, docs link |
| **The end user** | A message that isn't terrifying or leaky | *Not your job* — give the client enough to write their own |
| **You, debugging at 2am** | Which request, which server, which trace | `request_id`, correlated logs |

Two consequences fall straight out:

1. **`message` is not a contract; `code` is.** If clients branch on message text — and they will, if you give them nothing else — you can never improve your wording, and any i18n breaks them. **Give them a `code` so they don't have to.**
2. **Never put an end-user-facing message in an API error.** You don't know their locale, their context, or their UI. Return a machine code plus a developer-facing detail, and let the client map it to their own copy. (Exception: consumer-facing APIs sometimes include a `user_message` field explicitly labelled as such — Stripe does this for card decline reasons, because the merchant genuinely can't write better copy than the issuer's.)

---

## 3. RFC 9457 — Problem Details for HTTP APIs

There *is* a standard (RFC 9457, which obsoletes RFC 7807), and using it means clients and tooling already know your shape.

```http
HTTP/1.1 422 Unprocessable Content
Content-Type: application/problem+json
X-Request-Id: 01HQ8ZK3M4N5P6Q7R8S9T0

{
  "type": "https://docs.ledger.dev/errors/validation-failed",
  "title": "Validation failed",
  "status": 422,
  "detail": "The request body failed validation. See 'errors' for details.",
  "instance": "/v1/payments",
  "code": "validation_failed",
  "request_id": "01HQ8ZK3M4N5P6Q7R8S9T0",
  "errors": [
    { "field": "amount_minor", "code": "too_small",       "detail": "Must be at least 50 (minimum charge is $0.50)" },
    { "field": "currency",     "code": "unsupported_value","detail": "'xyz' is not a supported currency", "allowed": ["usd","eur","gbp","inr"] }
  ]
}
```

The five standard members, and what each is actually for:

| Member | Meaning | Rules |
|---|---|---|
| `type` | A URI identifying the **problem type** | Should be a real, dereferenceable docs URL. **This is the stable identifier** in the RFC's model |
| `title` | Short, human-readable summary | Must **not** change per occurrence — it identifies the type |
| `status` | The HTTP status code, duplicated | Convenience for logs and clients that lost the response object |
| `detail` | Human-readable explanation **of this occurrence** | *Can* vary per occurrence. Never put a stack trace here |
| `instance` | URI of the specific occurrence | The path, or a URI for the error event itself |

Plus **any extension members you like** — which is where `code`, `request_id` and `errors` live. That extensibility is the reason the RFC is usable in practice.

> **Should you use `application/problem+json` as the `Content-Type`?** It's correct and I'd do it — but be aware some HTTP clients and browser tooling only auto-parse `application/json`, and a few older SDK generators choke. If you hit that, `application/json` with the same body shape is a reasonable compromise. Being able to name that trade-off is more valuable than picking either.

### Do you need a separate `code` if you have `type`?

Strictly the RFC says `type` is the identifier. In practice **include both**, because:
- `type` is a URL — verbose, and clients writing `if (err.type === "https://docs.ledger.dev/errors/validation-failed")` is ugly and brittle if your domain changes.
- `code` is a short, stable, greppable token — `"validation_failed"` — which is what clients actually want to switch on.

That's why Stripe, Google and GitHub all ship a short code even though they know the RFC. Follow them.

---

## 4. Designing your error code taxonomy

This is the actual design work. A good taxonomy is **hierarchical, stable, and finite**.

```
Layer 1 — HTTP status:  the class of problem (does the client retry?)
Layer 2 — code:         the specific problem (what does the client do?)
Layer 3 — errors[]:     per-field detail (what does the developer fix?)
```

### Ledger's taxonomy

| HTTP | `code` | Client's correct action |
|---|---|---|
| 400 | `malformed_json` | Fix the request body. Never retry unchanged |
| 400 | `unknown_filter` | Fix the query param name |
| 400 | `missing_required_header` | Add the header (e.g. `Idempotency-Key`) |
| 401 | `missing_credentials` | Send an `Authorization` header |
| 401 | `invalid_api_key` | The key is wrong/revoked. Do **not** retry |
| 401 | `token_expired` | **Refresh the token and retry** — distinct action, hence a distinct code |
| 403 | `insufficient_scope` | This key lacks the scope. Show which one: `"required_scope": "payments:write"` |
| 403 | `account_suspended` | Contact support; retrying is pointless |
| 404 | `payment_not_found` | The ID is wrong or belongs to someone else |
| 405 | `method_not_allowed` | Read the `Allow` header |
| 409 | `already_refunded` | Read current state and reconsider |
| 409 | `invalid_state_transition` | Include `current_status` + `required_status` |
| 409 | `duplicate_resource` | An equivalent resource exists; include its `id` |
| 412 | `stale_version` | Re-`GET`, merge, retry with a fresh `If-Match` |
| 413 | `payload_too_large` | Reduce the body; include `max_bytes` |
| 415 | `unsupported_media_type` | Send `application/json` |
| 422 | `validation_failed` | Fix the fields listed in `errors[]` |
| 422 | `insufficient_funds` | A business rule, not a syntax problem |
| 429 | `rate_limit_exceeded` | Back off for `Retry-After` seconds |
| 500 | `internal_error` | Retry with backoff; report `request_id` if persistent |
| 502/504 | `upstream_error` / `upstream_timeout` | Retry with backoff — **and if the write is idempotent, safely** |
| 503 | `service_unavailable` | Back off for `Retry-After` |

**Three rules that make a taxonomy good:**

1. **One code per distinct client action.** If two situations require the identical client response, they can share a code. If they require different actions — `token_expired` (refresh) vs `invalid_api_key` (stop) — they **must** be different codes even though both are `401`. This is the single most useful design principle here.
2. **Codes are `snake_case`, stable, and additive-only.** Renaming a code is a breaking change. Adding one is not — *provided* clients have a default branch, which you should tell them to write.
3. **Keep the list finite and reviewable.** If you have 400 error codes, nobody handles them, and you've achieved nothing over free-text messages.

---

## 5. Field-level validation errors

For `422`, the per-field array is where the real value is.

```json
{
  "type": "https://docs.ledger.dev/errors/validation-failed",
  "title": "Validation failed", "status": 422, "code": "validation_failed",
  "request_id": "01HQ8Z...",
  "errors": [
    { "field": "amount_minor",        "code": "too_small",        "detail": "Must be at least 50", "min": 50 },
    { "field": "currency",            "code": "unsupported_value","detail": "Not supported", "allowed": ["usd","eur"] },
    { "field": "customer",            "code": "not_found",        "detail": "No customer with id 'cus_nope'" },
    { "field": "metadata.order_id",   "code": "too_long",         "detail": "Max 500 characters", "max": 500 },
    { "field": "items[2].quantity",   "code": "invalid_type",     "detail": "Expected integer, got string" }
  ]
}
```

Design notes, each one earned from real pain:

- **Always an array, even for one error.** Clients write one code path. And **return all validation errors at once** — a client shouldn't have to make five round trips to discover five problems. (Frameworks that fail fast on the first error make for a genuinely worse developer experience.)
- **`field` uses dotted/bracketed paths** that match the request body exactly, including array indices. That's what lets a UI highlight the right input. Alternatively use a JSON Pointer (`/items/2/quantity`) — pick one and be consistent.
- **Include the constraint as structured data** (`min`, `max`, `allowed`) so clients can build messages in their own language without parsing your prose.
- **Per-field `code`s, not just messages** — same reason as top-level codes.

### The security line in validation errors

```json
// ❌ leaks internals
{ "detail": "ERROR: duplicate key value violates unique constraint \"customers_email_key\"\n  at PaymentService.create (/app/src/services/payment.ts:142)" }

// ✅
{ "code": "duplicate_resource", "detail": "A customer with this email already exists",
  "field": "email", "existing_id": "cus_9s2k" }
```

**Never expose:** stack traces, SQL, table/column names, internal hostnames or IPs, file paths, library versions, or raw upstream error bodies. Each is free reconnaissance, and stack traces in production responses are a genuine, reportable vulnerability.

**And be careful with helpfulness.** `"No user with email alice@x.dev"` on a login endpoint is a **user-enumeration** oracle. On a *signup* endpoint you have a real tension: you must tell the user the email is taken, but that also confirms an account exists. The standard resolution: return a generic "check your email" response and send the *email* the specific information, plus rate-limit the endpoint hard. Volunteering this tension unprompted is a strong signal.

---

## 6. The implementation

One error class hierarchy, one handler, zero ad-hoc `res.status(400).json(...)` scattered through your codebase.

```ts
// ---------- errors.ts ----------
export type FieldError = {
  field: string; code: string; detail: string;
  [k: string]: unknown;                     // min/max/allowed extensions
};

export class ApiError extends Error {
  constructor(
    readonly status: number,
    readonly code: string,
    readonly detail: string,
    readonly extra: Record<string, unknown> = {},
    readonly fieldErrors?: FieldError[],
  ) {
    super(detail);
    this.name = "ApiError";
  }

  static badRequest(code: string, detail: string, extra = {}) { return new ApiError(400, code, detail, extra); }
  static unauthorized(code: string, detail: string, extra = {}) { return new ApiError(401, code, detail, extra); }
  static forbidden(code: string, detail: string, extra = {}) { return new ApiError(403, code, detail, extra); }
  static notFound(code: string, detail: string, extra = {}) { return new ApiError(404, code, detail, extra); }
  static conflict(code: string, detail: string, extra = {}) { return new ApiError(409, code, detail, extra); }
  static preconditionFailed(code: string, detail: string) { return new ApiError(412, code, detail); }
  static unprocessable(fieldErrors: FieldError[]) {
    return new ApiError(422, "validation_failed",
      "The request body failed validation. See 'errors'.", {}, fieldErrors);
  }
  static rateLimited(retryAfterSec: number) {
    return new ApiError(429, "rate_limit_exceeded",
      "Too many requests. Retry after the period indicated.", { retry_after: retryAfterSec });
  }
}

const TITLES: Record<number, string> = {
  400: "Bad Request", 401: "Unauthorized", 403: "Forbidden", 404: "Not Found",
  405: "Method Not Allowed", 409: "Conflict", 412: "Precondition Failed",
  413: "Content Too Large", 415: "Unsupported Media Type",
  422: "Validation failed", 428: "Precondition Required", 429: "Too Many Requests",
  500: "Internal Server Error", 503: "Service Unavailable",
};

// ---------- the ONE error handler, registered last ----------
import type { ErrorRequestHandler } from "express";
import { ZodError } from "zod";

export const errorHandler: ErrorRequestHandler = (err, req, res, _next) => {
  const requestId = (req as any).id as string;

  // 1. Normalise every known error type into ApiError.
  let e: ApiError;
  if (err instanceof ApiError) {
    e = err;
  } else if (err instanceof ZodError) {
    e = ApiError.unprocessable(err.issues.map(i => ({
      field: i.path.join("."),
      code: zodCodeToApiCode(i.code),          // map zod's vocabulary to YOURS
      detail: i.message,
    })));
  } else if ((err as any)?.type === "entity.too.large") {
    e = new ApiError(413, "payload_too_large", "Request body exceeds the limit", { max_bytes: 102_400 });
  } else if ((err as any)?.type === "entity.parse.failed") {
    e = ApiError.badRequest("malformed_json", "Request body is not valid JSON");
  } else {
    // 2. UNKNOWN error: log everything, expose nothing.
    req.log.error({ err, requestId, stack: (err as Error)?.stack }, "unhandled_error");
    e = new ApiError(500, "internal_error",
      "An unexpected error occurred. Contact support with the request_id if it persists.");
  }

  // 3. 5xx is our fault → always log with full context.
  if (e.status >= 500) req.log.error({ err, requestId, code: e.code }, "server_error");
  else req.log.warn({ requestId, code: e.code, status: e.status, path: req.path }, "client_error");

  // 4. Protocol obligations that clients depend on (Lesson 03).
  if (e.status === 401) res.set("WWW-Authenticate", `Bearer realm="ledger", error="${e.code}"`);
  if (e.status === 429 || e.status === 503) res.set("Retry-After", String(e.extra.retry_after ?? 30));
  if (e.status === 405 && e.extra.allow) res.set("Allow", String(e.extra.allow));

  res.status(e.status)
     .type("application/problem+json")
     .json({
       type: `https://docs.ledger.dev/errors/${e.code.replaceAll("_", "-")}`,
       title: TITLES[e.status] ?? "Error",
       status: e.status,
       detail: e.detail,
       instance: req.originalUrl,
       code: e.code,
       request_id: requestId,
       ...(e.fieldErrors ? { errors: e.fieldErrors } : {}),
       ...e.extra,
     });
};
```

Then in handlers you just throw, and there is exactly one place that decides what an error looks like:
```ts
const payment = await repo.find(id, req.merchantId);
if (!payment) throw ApiError.notFound("payment_not_found", `No payment with id '${id}'`);
if (payment.status !== "requires_capture") {
  throw ApiError.conflict("invalid_state_transition",
    `Cannot capture a payment with status '${payment.status}'`,
    { current_status: payment.status, required_status: "requires_capture" });
}
```

> **Spring equivalent:** `@RestControllerAdvice` + `@ExceptionHandler` + `ProblemDetail` (Spring 6). Identical architecture: one central translator, domain code throws typed exceptions. See `spring boot/05-web/16-api-design-dto-validation-errors.md`.

---

## 7. Production rules

| Rule | Why |
|---|---|
| **One error handler. Zero inline error shapes** | Consistency is the entire value; scattered `res.status(400).json({msg})` guarantees inconsistency |
| **Never `200` with an error body** | Breaks every retry library, every dashboard, every client's error path (Lesson 03) |
| **Stable `snake_case` `code` on every error** | It's the only thing clients can safely branch on |
| **One code per distinct client action** | `token_expired` vs `invalid_api_key` need different code paths |
| **`request_id` in every error body *and* header** | The only way to connect "it failed at 14:03" to a log line |
| **Return all validation errors at once** | Five round trips to find five problems is a hostile DX |
| **Never leak stack traces, SQL, hostnames, paths, or upstream bodies** | Reconnaissance; a reportable vulnerability in production |
| **Log 5xx at error, 4xx at warn/info** | Otherwise client typos page your on-call at 3am |
| **Honour protocol obligations: `WWW-Authenticate`, `Retry-After`, `Allow`** | These are how automatic clients recover |
| **Document every code with a real docs URL** | The `type` URI should actually resolve; a 404 docs link is worse than none |
| **Never let a validation message echo the raw input back unescaped into HTML** | Reflected XSS via an API error rendered by a client |
| **Test your error paths** | Error handling is the least-tested and most-read code in most APIs |

---

## 8. Interview traps

**Q1. "Design an error response for your API."**
Structure the answer around the **four audiences** (§2) — that framing alone puts you ahead. Then show the shape: HTTP status + stable `code` + human `detail` + per-field `errors[]` + `request_id`, and mention RFC 9457 by name so they know you didn't invent it.

**Q2. "Why not just return a message string?"**
Because clients then branch on prose, which means you can never reword it, i18n breaks them, and every client re-implements string matching. `code` is the contract; `detail` is the courtesy.

**Q3. "400 vs 422?"**
400 = can't parse / structurally wrong. 422 = parsed fine, semantically invalid. Then the pragmatic half: *"the distinction buys clients little because they branch on `code`, not the status. I'd pick one convention and apply it consistently — many major APIs including Stripe use 400 for everything client-side."*

**Q4. "Someone requests a resource that exists but isn't theirs. 403 or 404?"**
`404` when the resource's existence is sensitive (which for multi-tenant data it almost always is), because `403` confirms it exists and enables enumeration. `403` when the caller legitimately knows it exists but lacks permission for that *action* (a viewer trying to delete a repo they can see). Name GitHub's private-repo behaviour as the reference implementation.

**Q5. "Your endpoint calls a payment processor that returns a 500. What do you return?"**
Not a 500 verbatim, and never their raw body. `502 upstream_error` or `504 upstream_timeout` with your own `request_id`, plus — the important part — **you must decide the state of the operation**. If you sent the charge and lost the response, the money may have moved. So: record the attempt with your idempotency key *before* calling out, return `502`, and reconcile asynchronously. *"The dangerous case isn't the error, it's the ambiguity"* is the sentence that shows you've thought about this ([Lesson 18](../04-production/18-reliability-and-idempotency.md)).

**Q6. "How do you handle an error in the middle of a bulk operation?"**
`207` with per-item statuses and the input index (Lesson 09). Never a single top-level error that discards the successes.

**Q7. "Should errors be localised?"**
Your API's `detail` is developer-facing — English is fine. **Localisation is the client's job**, and your job is to give them the structured data (`code`, `field`, `min`, `allowed`) they need to write localised copy. Exception: a `user_message` field, explicitly labelled, when only you have the information (card decline reasons).

**Q8. "A client complains about an intermittent 500. What do you need?"**
The `request_id` — which means you must have returned it. Then: structured logs keyed by it, the trace, the input that triggered it, and the deploy timeline. If you can't answer "what did *this* request do", your observability is the bug ([Lesson 20](../04-production/20-observability.md)).

**Q9. "What's wrong with `{"success": false, "error": "..."}` and a 200?"**
Everything downstream: LBs/CDNs cache it as a success, retry libraries don't retry, your error-rate dashboard reads 0%, alerting never fires, and every client must parse a body to learn whether the call worked. It also makes it impossible for infrastructure to act on failures at all.

---

## 9. Build & break

### Build — the error contract as a document
Create `docs/errors.md` for Ledger listing every code: HTTP status, `code`, when it occurs, what the client should do, and whether it's retryable. This document **is** part of your API contract, and writing it will expose codes you'd otherwise have invented ad hoc.

| status | code | retryable | client action |
|---|---|---|---|
| 401 | `token_expired` | after refresh | refresh, then retry once |
| 409 | `already_refunded` | no | read state; surface to user |
| 429 | `rate_limit_exceeded` | yes, after `Retry-After` | back off |
| 502 | `upstream_error` | yes, with backoff **if idempotent** | retry with the same `Idempotency-Key` |

That `retryable` column is the most valuable one, and almost no API publishes it. Publishing it is a differentiator.

### Build — the handler, then prove it's consistent
Implement §6, then write a test that hits **every** error path and asserts the shape:
```ts
const REQUIRED = ["type", "title", "status", "detail", "code", "request_id"];

test.each([
  ["malformed json",   () => post("/v1/payments", "{not json"),                400, "malformed_json"],
  ["wrong media type", () => postRaw("/v1/payments", "a=1", "text/plain"),     415, "unsupported_media_type"],
  ["no auth",          () => post("/v1/payments", {}, { auth: null }),         401, "missing_credentials"],
  ["validation",       () => post("/v1/payments", { amount_minor: -5 }),       422, "validation_failed"],
  ["not found",        () => get("/v1/payments/pi_nope"),                      404, "payment_not_found"],
  ["bad transition",   () => post("/v1/payments/pi_paid/capture", {}),         409, "invalid_state_transition"],
  ["method",           () => del("/v1/payments"),                             405, "method_not_allowed"],
])("%s → %i %s", async (_name, call, status, code) => {
  const res = await call();
  expect(res.status).toBe(status);
  expect(res.headers["content-type"]).toContain("problem+json");
  expect(res.body.code).toBe(code);
  for (const k of REQUIRED) expect(res.body).toHaveProperty(k);
  expect(res.headers["x-request-id"]).toBe(res.body.request_id);
  expect(JSON.stringify(res.body)).not.toMatch(/\/app\/src|at \w+\.\w+ \(|SELECT |node_modules/); // no leaks
});
```
That last assertion — grepping your own error bodies for leak signatures — is worth stealing for real projects.

### Break — four failures
1. **Throw a raw `Error` with a stack trace** and no handler. Look at what a client sees. Find your file paths and library versions in the response. That's the vulnerability.
2. **Return `200 {"success": false}`** for a validation error. Point a retry library at it, then look at your error-rate metric: 0%. Now imagine debugging a customer's "it silently doesn't work" report.
3. **Return only a `message`, no `code`.** Write a client that must distinguish "expired token" from "invalid key" and watch yourself write `if (msg.includes("expired"))`. Then reword the message and break your own client.
4. **Return one validation error at a time.** Submit a form with five bad fields and count the round trips. That's the DX your users would have had.

### Explain out loud (90 seconds)
1. The four audiences of an error and the field that serves each.
2. Why `code` is the contract and `message` isn't.
3. Your 403-vs-404 rule and the security reason.
4. What you return when an upstream payment processor times out, and why the ambiguity is worse than the error.

---

## What's next

Your API is designed and its failures are legible. The final Module 2 question is the one you asked at the start: **what happens when it has to change?** Versioning, breaking vs non-breaking, deprecation — and the specific question *"should my API reject an extra parameter it wasn't designed for?"*, which has three defensible answers.

Next → **[Lesson 11: Versioning, evolution & unknown fields](11-versioning-and-evolution.md)**
