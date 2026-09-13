# Lesson 09 — Writes: POST vs PUT vs PATCH, bulk & concurrency

> **Why this lesson exists:** reads are forgiving — a wrong read is a bug you can fix and re-run. **A wrong write destroys data.** This lesson covers the three things that go wrong: choosing the wrong update semantics (so clients can't express "clear this field"), the lost-update race (two clients, one silently erased), and bulk operations (where half the batch succeeds and nobody knows what to do). All three are common interview material, and the concurrency one is where seniority shows.

**Time:** ~85 minutes · **Prereq:** Lessons 03, 07

---

## 1. The idea in one sentence

> **Every write API must answer three questions explicitly: what does the client mean by "not sending a field", what happens if two clients write at once, and what happens if half the request succeeds — and if you don't answer them, you've answered them badly by default.**

---

## 2. The three write shapes, decided properly

| | `POST /payments` | `PUT /payments/pi_1` | `PATCH /payments/pi_1` |
|---|---|---|---|
| Who picks the URI | Server | **Client** | Existing |
| Creates? | Yes | Yes (upsert) | No (usually) |
| Body is | The thing to create | **The complete new state** | **A description of changes** |
| Idempotent | ❌ (needs a key) | ✅ | ❌ (unless `If-Match`) |
| Omitted field means | Use the default | **Clear it** | **Leave it alone** |

**That last row is the whole design decision**, and it's the one that's usually undocumented. Given `{"name": "Alice"}` against a customer who has a phone number:
- **`PUT`** → the phone is **deleted**. The client said "the customer is exactly this."
- **`PATCH`** → the phone is **kept**. The client said "change the name."

If your `PUT` handler leaves the phone alone, you have implemented `PATCH` and named it `PUT`. Clients that use `PUT` to clear fields — the correct usage — will silently fail, and they will not find out until a customer complains that they can't remove their phone number.

> **The pragmatic reality:** most APIs offer only `POST`, `PATCH` and `DELETE`, skipping `PUT` entirely — Stripe does exactly this. That's a perfectly defensible choice: full replacement is rarely what clients want, and offering it invites accidental data loss. **Say that in an interview:** *"I'd offer POST/PATCH/DELETE and skip PUT unless there's a real upsert-with-client-ID use case like provisioning a webhook endpoint by a known key. Half-implementing PUT is worse than not offering it."*

---

## 3. Choosing your PATCH format

HTTP defines `PATCH` but deliberately **not** the body format. You must choose one, and the choice has consequences.

### Option A — JSON Merge Patch (RFC 7386) · **my default recommendation**

```http
PATCH /v1/customers/cus_9s2k
Content-Type: application/merge-patch+json

{ "name": "Alice Smith", "phone": null, "metadata": { "tier": "gold" } }
```

Rules, and they're precise:
- A key present with a value → **set it**
- A key present with `null` → **delete/clear it**
- A key **absent** → leave it alone
- Nested objects **merge recursively**; arrays are **replaced wholesale**

That last point is where it hurts: you cannot append one item to an array — you must send the whole array. In practice that's fine, because array mutation usually deserves its own sub-resource endpoint anyway (`PUT /payments/pi_1/tags/urgent` from Lesson 07).

**Why this is the default:** it's what clients naïvely expect, it's readable, it maps directly onto how ORMs apply partial updates, and `null`-means-delete gives you the one thing ad-hoc PATCH can't express.

### Option B — JSON Patch (RFC 6902)

```http
PATCH /v1/customers/cus_9s2k
Content-Type: application/json-patch+json

[
  { "op": "test",    "path": "/version", "value": 7 },
  { "op": "replace", "path": "/name",    "value": "Alice Smith" },
  { "op": "remove",  "path": "/phone" },
  { "op": "add",     "path": "/tags/-",  "value": "vip" }
]
```

Ops: `add`, `remove`, `replace`, `move`, `copy`, `test`. Genuinely more powerful:
- **Array operations** (`/tags/-` appends, `/tags/0` targets an index)
- **`test`** gives you built-in optimistic concurrency without headers — *"only apply if version is still 7"*
- **Atomic** — the whole document of operations applies or none does

Costs: verbose, harder for clients to construct, JSON Pointer escaping is fiddly (`~1` for `/`, `~0` for `~`), and validating an arbitrary op list against your business rules is real work — you must apply the ops to a copy, then validate the *result*, not the ops.

**When to actually pick it:** collaborative editing, document-shaped resources, config management (Kubernetes uses both JSON Patch and its own strategic merge patch), or anywhere clients need to express array edits precisely.

### Option C — Ad-hoc "send the changed fields" with `application/json`

What ~90% of APIs ship. It's Merge Patch minus the specification, and it has two ambiguities you must resolve and document:

1. **Does `null` clear the field, or is it invalid?** Pick one. If `null` is invalid you have **no way for a client to clear an optional field**, which is a real functional gap people discover late.
2. **Do nested objects merge or replace?** `{"metadata": {"a": 1}}` on `{"metadata": {"a": 0, "b": 2}}` → is `b` still there?

> **The `undefined` vs `null` trap in TypeScript/JS.** `JSON.stringify({phone: undefined})` produces `{}` — the key vanishes. So a client doing `{...form, phone: form.phone || undefined}` can never clear a field, while `{phone: null}` can. When you validate with Zod, `.optional()` (may be absent) and `.nullable()` (may be null) are **different**, and for PATCH you often need `.optional().nullable()` plus the ability to distinguish "absent" from "null":
> ```ts
> const PatchCustomer = z.object({
>   name:  z.string().min(1).optional(),          // absent = unchanged
>   phone: z.string().nullable().optional(),      // null = clear, absent = unchanged
> }).strict();
>
> // Distinguish absent from null — `in` works, truthiness does not:
> if ("phone" in body) { patch.phone = body.phone; }   // null lands here correctly
> ```
> This is exactly the kind of detail that turns into a data-loss bug, and it's a great answer to *"what's tricky about implementing PATCH?"*

---

## 4. Concurrency: the lost update problem

This is the section to slow down on. It's the highest-value concurrency question in API interviews and most candidates only get halfway.

### The race

```
t0  Client A: GET /customers/cus_1  → { name: "Alice", tier: "gold", phone: "+1..." }
t1  Client B: GET /customers/cus_1  → { name: "Alice", tier: "gold", phone: "+1..." }
t2  Client A: PUT { name: "Alice Smith", tier: "gold", phone: "+1..." }   ✅ 200
t3  Client B: PUT { name: "Alice",       tier: "platinum", phone: "+1..." } ✅ 200

Final state: { name: "Alice", tier: "platinum" }
```

**A's name change is gone.** No error, no warning, no log line saying anything happened. This is the **lost update problem**, and note that `PATCH` only narrows it — two clients patching the *same field* still race, and "last write wins" is a policy you chose by accident.

### Solution 1 — Optimistic concurrency with `ETag` + `If-Match` (the HTTP-native answer)

```http
GET /v1/customers/cus_1
→ 200 OK
  ETag: "7"
  { "name": "Alice", "tier": "gold" }

PATCH /v1/customers/cus_1
If-Match: "7"                       ← "only if it's still version 7"
{ "name": "Alice Smith" }

→ 200 OK, ETag: "8"                 ← A succeeds, version bumps

PATCH /v1/customers/cus_1
If-Match: "7"                       ← B still thinks it's 7
{ "tier": "platinum" }

→ 412 Precondition Failed           ← B is told, instead of silently winning
```

Now B's client can re-`GET`, merge, and retry — or show the user *"this record changed while you were editing."* **The write is no longer silently destructive.**

Implementation, and the important part is that the check must be **atomic with the write**:

```ts
export async function patchCustomer(req: Request, res: Response) {
  const ifMatch = req.header("if-match");
  if (!ifMatch) {
    // 428 tells the client the *mechanism*, not just "no".
    return res.status(428).json(problem("precondition_required",
      "Send If-Match with the ETag from your last GET"));
  }

  const expectedVersion = Number(ifMatch.replace(/^W\//, "").replace(/"/g, ""));
  if (!Number.isInteger(expectedVersion)) {
    return res.status(400).json(problem("invalid_if_match", "ETag must be a quoted integer version"));
  }

  // ✅ The version check and the write are ONE statement. This is the whole trick.
  const { rowCount, rows } = await db.query(
    `UPDATE customers
        SET name = COALESCE($1, name),
            phone = CASE WHEN $2::bool THEN $3 ELSE phone END,
            version = version + 1,
            updated_at = now()
      WHERE id = $4 AND merchant_id = $5 AND version = $6
      RETURNING *`,
    [patch.name ?? null, "phone" in patch, patch.phone ?? null,
     req.params.id, req.merchantId, expectedVersion],
  );

  if (rowCount === 0) {
    // Distinguish "gone" from "stale" — the client's next action differs.
    const exists = await db.exists(req.params.id, req.merchantId);
    return exists
      ? res.status(412).json(problem("stale_version",
          "The customer was modified since you read it. GET it again and retry."))
      : res.status(404).json(problem("customer_not_found", "No such customer"));
  }

  res.set("ETag", `"${rows[0].version}"`).json(toDto(rows[0]));
}
```

> **The mistake to avoid:** `SELECT version` → compare in application code → `UPDATE`. That's a race between your read and your write — the exact bug you're trying to fix, moved one layer down. **The version predicate must be in the `WHERE` clause of the `UPDATE`.**

### What to put in the ETag

| Source | Pros | Cons |
|---|---|---|
| **A `version` integer column** | Cheap, obvious, monotonic, works with the atomic UPDATE above | Requires a schema column |
| **Hash of the representation** | No schema change; also enables `304` caching | Must compute the body to know the ETag; changes when *serialization* changes, not just data |
| **`updated_at` timestamp** | Free | Clock resolution collisions; two updates in the same millisecond are indistinguishable |

Use a `version` column for concurrency, and a content hash if you also want `If-None-Match` caching. **Strong vs weak ETags:** `W/"7"` (weak) means "semantically equivalent"; `"7"` (strong) means byte-identical. `If-Match` requires a strong comparison, so **use strong ETags for concurrency control.**

> **Spring equivalent:** JPA's `@Version` is exactly this mechanism, throwing `OptimisticLockException` which you map to `412`/`409`. You already have this in `spring boot/06-data/23-transactions-deep-dive.md`; the API-layer job is exposing it as `ETag`/`If-Match` instead of leaking a `version` field into the body.

### Solution 2 — Pessimistic locking
`SELECT ... FOR UPDATE`, holding a row lock for the transaction's duration. Correct, but it holds a database lock across a *network round trip* if you lock during user think-time — which is how you get lock-wait timeouts and deadlocks under load. **Use for short, server-internal critical sections** (moving money between two balances), never for "user is editing a form."

### Solution 3 — Conflict-free design (the best answer when it applies)
Avoid the race entirely by making operations commutative:
```
❌ PATCH /balance {"amount": 5000}                  ← last write wins, money vanishes
✅ POST  /balance/entries {"delta": 500, ...}       ← append-only; sum is the balance
```
This is why **ledgers are append-only**, and it's the correct design for Ledger's balances. There is no lost update if you never update. Mention this and you've shown you can design *around* concurrency rather than only defending against it — a distinctly senior move.

### Which to use where

```
Money / balances / counters           → append-only entries (Solution 3)
User-editable records (forms)         → ETag + If-Match (Solution 1)
Short server-side critical section    → SELECT FOR UPDATE (Solution 2)
State machine transitions             → conditional UPDATE on the current state:
                                        WHERE id=$1 AND status='requires_capture'
                                        → 0 rows means 409, and it's atomic
```

That last one deserves emphasis because it's Ledger's core pattern:

```sql
-- Capture a payment: legal only from requires_capture. Atomic, no locks, no races.
UPDATE payments SET status='succeeded', captured_at=now(), version=version+1
 WHERE id=$1 AND merchant_id=$2 AND status='requires_capture'
 RETURNING *;
-- 0 rows → either it doesn't exist (404) or it's in the wrong state (409). Check which.
```

**State machines belong in the `WHERE` clause.** An `if (payment.status === 'requires_capture')` in application code is a race; a `WHERE status='requires_capture'` is a guarantee.

---

## 5. Bulk and batch operations

You'll be asked for these, and the interesting part is not the happy path.

### Two different things, often confused

| | **Bulk** | **Batch** |
|---|---|---|
| Meaning | Same operation, many items | Different operations, one request |
| Example | `POST /payments/bulk` with 500 payments | One request containing a create, an update and a delete |
| Complexity | Moderate | High — ordering, dependencies, partial failure |

Batch is what Google's `/batch` endpoints and Facebook's Graph batching do. **Don't build it unless you must**; it's an API for building APIs, and it reproduces HTTP inside HTTP.

### The hard question: what happens when item 237 fails?

Three answers. **You must pick one and document it** — this is the whole design decision.

**A. All-or-nothing (transactional)**
```http
POST /v1/payments/bulk
{ "atomic": true, "items": [ ...500... ] }

→ 422 Unprocessable Content
{ "error": { "code": "bulk_validation_failed",
             "failures": [ { "index": 237, "code": "invalid_currency", "field": "currency" } ] } }
```
Simple to reason about; one bad row rejects 499 good ones. Correct when the items are interdependent (a double-entry ledger transaction).

**B. Best-effort with per-item results (`207 Multi-Status`)** — usually what clients want
```http
POST /v1/payments/bulk
{ "atomic": false, "items": [ ...500... ] }

→ 207 Multi-Status
{
  "object": "bulk_result",
  "summary": { "total": 500, "succeeded": 498, "failed": 2 },
  "results": [
    { "index": 0,   "status": 201, "data": { "id": "pi_1", ... } },
    { "index": 237, "status": 422, "error": { "code": "invalid_currency", "field": "currency" } },
    { "index": 499, "status": 201, "data": { "id": "pi_500", ... } }
  ]
}
```
Rules that make this usable:
- **Always echo the `index`** (or a client-supplied `reference` per item). Without it, the client can't correlate results to inputs — and if you filter out successes, they *definitely* can't.
- **Per-item status codes**, so the client's existing error handling works per item.
- **`207` at the transport level** — never `200`, because monitoring and clients need to know it was mixed.
- **Idempotency at the item level**, ideally: an `Idempotency-Key` per item, or one key for the batch plus deterministic item references, so a retry doesn't re-create the 498 that succeeded. This is the detail that makes bulk endpoints genuinely hard.

**C. Async job** — the right answer for large batches
```http
POST /v1/payments/bulk    (10,000 items, or a file upload)
→ 202 Accepted
  Location: /v1/bulk-jobs/bj_1
  { "id": "bj_1", "status": "processing", "total": 10000 }

GET /v1/bulk-jobs/bj_1
→ { "status": "completed", "succeeded": 9987, "failed": 13,
    "errors_url": "/v1/bulk-jobs/bj_1/errors" }
```
**Rule of thumb: over ~100 items or ~10 seconds of work, go async.** A synchronous 10,000-item request will hit an LB timeout, be retried by the client, and double your load — while the first attempt is still running.

### Bulk limits, and why each exists
```
max items per request:     100 (sync) / 10,000 (async file)
max body size:             1 MB (sync)
per-item validation:       before ANY processing, so you can reject early
rate limiting:             count each item toward the quota, not each request
                           ← otherwise bulk is a free rate-limit bypass
```
That last line is a real vulnerability and an excellent thing to volunteer: *"I'd charge the rate limiter per item, because otherwise a client sends 100-item batches and gets 100× their quota."*

---

## 6. Production rules

| Rule | Why |
|---|---|
| **Document what an omitted field means, per method** | It's the one semantic clients cannot guess, and guessing wrong destroys data |
| **If you offer `PUT`, make it a true full replacement** | A half-`PUT` breaks the only promise `PUT` makes |
| **Prefer `POST`/`PATCH`/`DELETE` and skip `PUT`** unless you have a real client-ID upsert | Fewer ways to lose data |
| **Pick a PATCH format and put it in `Content-Type`** | `application/merge-patch+json` documents itself and lets you add JSON Patch later without ambiguity |
| **Distinguish "absent" from `null` in your validator** | Otherwise clients can never clear an optional field |
| **`.strict()` on write schemas — reject unknown fields** | Prevents mass assignment; see the nuance in [Lesson 11](11-versioning-and-evolution.md) |
| **Never accept ownership/tenant fields in a write body** | `{"merchant_id": "..."}` is a cross-tenant write. Derive from the token |
| **Never accept server-controlled fields** (`id`, `created_at`, `status`, `balance`) | Mass assignment: a client setting `status: "succeeded"` on a payment is fraud |
| **Put the version check in the `UPDATE ... WHERE`, never in app code** | A read-then-write check is itself a race |
| **Put state-machine guards in the `WHERE` clause** | Same reason. `WHERE status='requires_capture'` is atomic; an `if` is not |
| **412 for stale, 409 for illegal state, 404 for missing — distinguish them** | The client's next action differs for each |
| **Return the updated resource + new `ETag` on writes** | Saves a round trip and keeps the client's version current |
| **Bulk: echo the input index, use 207, cap the size, meter per item** | Correlation, honesty, DoS and quota-bypass defence |
| **Over ~100 items → async job** | Sync bulk hits LB timeouts and gets retried, doubling load |

---

## 7. Interview traps

**Q1. "Two clients update the same resource simultaneously. How do you prevent a lost update?"**
The flagship question. Answer in three layers, in this order:
1. **Optimistic concurrency:** `ETag` on `GET`, `If-Match` on write, `412` on mismatch — and the version predicate must be inside the `UPDATE ... WHERE`, not a separate read.
2. **Pessimistic locking** for short server-side critical sections; explain why you wouldn't hold a lock across user think-time.
3. **Design it away:** append-only entries for anything additive, so concurrent writes commute and there's nothing to lose.
Then close with: *"and for state transitions I'd put the expected state in the WHERE clause, so an illegal transition is a 0-row update rather than a race."* That last sentence is the one that lands.

**Q2. "How do you make `PATCH` idempotent?"**
Require `If-Match`. The first request applies and bumps the version; the retry sends the same now-stale ETag and gets `412` instead of applying twice. Alternatively use an `Idempotency-Key`, or make the patch semantics replacement-style rather than relative (`{"status":"x"}` rather than `{"increment":1}`).

**Q3. "Client sends `{"phone": null}`. What does it mean?"**
Under Merge Patch: **clear the field.** Under a strict-typed schema that rejects null: an error, which means the client has no way to clear it. **This is a design decision you must make explicitly** — and the follow-up is the `undefined`-disappears-in-`JSON.stringify` trap from §3.

**Q4. "Design a bulk create endpoint for 10,000 records."**
Async: `POST` → `202` + job resource → poll or webhook → per-item error report. Cover per-item idempotency (so a retry doesn't duplicate the successes), per-item rate metering, size limits, and streaming/file upload for the input. If they say "make it synchronous," push back with the LB-timeout-plus-retry-storm reasoning and offer 100 items as the sync cap.

**Q5. "Half your bulk operation failed. What status code?"**
`207 Multi-Status` with per-item statuses, **never** `200`. Then the important follow-up: *"and I'd echo the request index per item, because otherwise the client can't tell which of its 500 inputs failed."*

**Q6. "What's mass assignment and how do you prevent it?"**
Binding a request body directly onto your entity, so a client can set fields you never intended: `{"role": "admin"}`, `{"merchant_id": "other"}`, `{"balance": 999999}`, `{"status": "succeeded"}`. Prevention: an explicit input DTO with **only** client-settable fields, `.strict()` validation, and never `Object.assign(entity, req.body)`. Name the real-world instance if you can: this class of bug is how GitHub was compromised in 2012 (a mass-assignment on a Rails model).

**Q7. "Why is `PUT` idempotent but `POST` isn't, and what breaks if you get it backwards?"**
`PUT` states the final state, so repeating it converges; `POST` requests processing, so repeating it processes again. What breaks: every retry layer — client libraries, service meshes, load balancers — retries idempotent methods automatically. Make `POST` your update method and a network blip becomes a duplicate charge; make `PUT` non-idempotent and the infrastructure will duplicate it *for* you, without asking.

**Q8. "`409` or `412`?"**
`412` = a **precondition you supplied** failed (`If-Match` mismatch) — retry after re-reading. `409` = the request conflicts with the resource's **current state** independent of preconditions (duplicate email, refunding an already-refunded payment) — retrying unchanged will never work. Different client actions, hence different codes.

**Q9. "Can `DELETE` have a body?"**
Spec-wise it's not forbidden but has no defined semantics, and intermediaries may drop it. So: no, don't design for it. For bulk delete use `POST /resource/bulk-delete` or `DELETE /resource?ids=a,b,c` (watching URL length).

---

## 8. Build & break

### Build — a PATCH endpoint that can't lose data
```ts
import { z } from "zod";

/** Only client-settable fields. Nothing else is even representable. */
const PatchCustomer = z.object({
  name:     z.string().min(1).max(200).optional(),
  email:    z.string().email().optional(),
  phone:    z.string().max(32).nullable().optional(),      // null = clear
  metadata: z.record(z.string().max(500)).nullable().optional(),
}).strict();          // ← unknown key → error, so `merchant_id` or `balance` can't sneak in

type PatchInput = z.infer<typeof PatchCustomer>;

/** Turn "present vs absent" into explicit column instructions. */
function buildSet(patch: PatchInput) {
  const sets: string[] = [];
  const vals: unknown[] = [];
  const push = (col: string, v: unknown) => { sets.push(`${col} = $${vals.length + 1}`); vals.push(v); };

  if ("name"     in patch) push("name", patch.name);
  if ("email"    in patch) push("email", patch.email);
  if ("phone"    in patch) push("phone", patch.phone ?? null);      // null clears
  if ("metadata" in patch) push("metadata", patch.metadata ?? null);

  return { sets, vals };
}

export async function patchCustomer(req: Request, res: Response) {
  const parsed = PatchCustomer.safeParse(req.body);
  if (!parsed.success) return res.status(422).json(validationProblem(parsed.error));
  if (Object.keys(parsed.data).length === 0)
    return res.status(400).json(problem("empty_patch", "Provide at least one field to change"));

  const ifMatch = req.header("if-match");
  if (!ifMatch) return res.status(428).json(problem("precondition_required",
    "Send If-Match with the ETag from your last GET of this customer"));
  const version = Number(ifMatch.replaceAll('"', ""));
  if (!Number.isInteger(version))
    return res.status(400).json(problem("invalid_if_match", "ETag must be a quoted integer"));

  const { sets, vals } = buildSet(parsed.data);
  const { rows } = await db.query(
    `UPDATE customers SET ${sets.join(", ")}, version = version + 1, updated_at = now()
      WHERE id = $${vals.length + 1} AND merchant_id = $${vals.length + 2} AND version = $${vals.length + 3}
      RETURNING *`,
    [...vals, req.params.id, req.merchantId, version],
  );

  if (rows.length === 0) {
    const exists = await db.oneOrNone(
      `SELECT 1 FROM customers WHERE id=$1 AND merchant_id=$2`, [req.params.id, req.merchantId]);
    return exists
      ? res.status(412).json(problem("stale_version", "Modified since your last read. GET and retry."))
      : res.status(404).json(problem("customer_not_found", "No such customer"));
  }

  res.set("ETag", `"${rows[0].version}"`).json(toDto(rows[0]));
}
```

### Build — an atomic state transition
```ts
/** Capture: legal ONLY from requires_capture. The guard is in the WHERE clause. */
export async function capturePayment(req: Request, res: Response) {
  const { rows } = await db.query(
    `UPDATE payments
        SET status = 'succeeded', captured_at = now(), version = version + 1
      WHERE id = $1 AND merchant_id = $2 AND status = 'requires_capture'
      RETURNING *`,
    [req.params.id, req.merchantId],
  );

  if (rows.length === 0) {
    const current = await db.oneOrNone(
      `SELECT status FROM payments WHERE id=$1 AND merchant_id=$2`, [req.params.id, req.merchantId]);
    if (!current) return res.status(404).json(problem("payment_not_found", "No such payment"));
    return res.status(409).json(problem("invalid_state_transition",
      `Cannot capture a payment with status '${current.status}'`,
      { current_status: current.status, required_status: "requires_capture" }));
  }
  res.set("ETag", `"${rows[0].version}"`).json(toDto(rows[0]));
}
```
Note there is **no `if (payment.status === ...)` anywhere.** That's the point.

### Break — four experiments
1. **Reproduce the lost update.** Two terminals. `GET` the same customer in both, then `PATCH` different fields from each without `If-Match`. Confirm one change vanished with no error. Now add `If-Match` and watch the second get `412`.
2. **Prove the read-then-write race is still broken.** Implement version checking as `SELECT version` → `if (v !== expected) throw` → `UPDATE`. Fire 50 concurrent patches with the same starting ETag (`Promise.all`) and count how many succeed. It will be more than one. Move the predicate into the `WHERE` and re-run: exactly one succeeds.
3. **Mass assignment.** Remove `.strict()` and use `Object.assign(customer, req.body)`. Then `PATCH {"merchant_id": "mrc_someone_else", "balance": 99999999}`. Watch it work. Put `.strict()` back and feel why it matters.
4. **Double capture.** Fire two concurrent `POST /payments/pi_1/capture`. With the `WHERE status=...` guard, exactly one gets `200` and one gets `409`. Remove the guard and use an `if` instead — watch both succeed and the payment get captured twice.

### Explain out loud (2 minutes)
1. What an omitted field means for `PUT` vs `PATCH`, and why that's the key decision.
2. The lost-update race, and three ways to prevent it, ranked.
3. Why the version check must live in the `UPDATE ... WHERE`.
4. Bulk partial failure: which status, and what must be in the response.

---

## What's next

You can write safely. But every failure so far has produced an error, and errors are a contract too — arguably the most-read part of your API, because that's what developers see when they're stuck. Next: an error format clients can act on programmatically, and the taxonomy that makes 400 vs 409 vs 422 a decision rather than a coin flip.

Next → **[Lesson 10: Errors & status codes clients can act on](10-errors-and-problem-details.md)**
