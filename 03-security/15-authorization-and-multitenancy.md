# Lesson 15 — Authorization, multi-tenancy & BOLA

> **Why this lesson exists:** this is the most important security lesson in the track. **Broken Object Level Authorization is OWASP's #1 API risk**, and it's #1 because it's the easiest bug to write and the hardest to notice: the code works, the tests pass, and the endpoint returns 200 — just with someone else's data. Every real API breach you've read about was more likely this than a cryptography failure.

**Time:** ~85 minutes · **Prereq:** Lessons 12–14

---

## 1. The idea in one sentence

> **Authentication tells you *who* is calling; authorization must be re-established for *every object* they touch — and the only reliable way to do that is to make it structurally impossible to write a query that isn't scoped to the caller.**

The key word is **structurally**. Authorization enforced by developer discipline fails, because it only takes one forgotten `WHERE` clause in one new endpoint written on a Friday.

---

## 2. BOLA: the bug, in five lines

```ts
// ❌ BOLA. This is the #1 API vulnerability in the world.
app.get("/v1/payments/:id", authenticate, async (req, res) => {
  const payment = await db.payments.findById(req.params.id);   // ← no tenancy check!
  res.json(toDto(payment));
});
```

The caller is authenticated — a real merchant with a real token. They then request `GET /v1/payments/pi_someone_elses`, and get it. Every layer worked as designed: TLS was valid, the token verified, the query succeeded, the response was 200. **The only thing missing is the question "is this yours?"**

Why this bug is so common:
1. **It requires no attacker sophistication.** Change an ID in a URL. That's it.
2. **It doesn't look like a bug.** The code reads naturally and the happy path is correct.
3. **Tests miss it**, because tests are written from the perspective of the owner. You test that Alice can read Alice's payment; you rarely test that Bob *can't*.
4. **Every new endpoint is a fresh opportunity.** You can get it right 200 times and wrong once.

Sequential IDs make it trivially exploitable (Lesson 07) — but note that **opaque IDs are not a fix**, they're a speed bump. `pi_3Nx8kQ2mR` can't be guessed, but it *can* be learned: from a shared CSV export, a support ticket, a webhook that went to the wrong endpoint, a URL in a screenshot, or a previous employee. **Authorization is the fix. Unguessable IDs are defence in depth.**

### The whole family of "broken authorization" bugs

| Name | The bug |
|---|---|
| **BOLA** (Broken Object Level Authorization) | Reading/writing another tenant's object by ID |
| **BOPLA** (Broken Object Property Level Authorization) | You may edit the object, but not *that field* — e.g. a user editing their own `role` or `balance`. This is mass assignment ([Lesson 09](../02-rest-design/09-writes-patch-and-bulk.md)) |
| **BFLA** (Broken Function Level Authorization) | Calling an endpoint your role shouldn't reach — a viewer hitting `DELETE /payments/:id`, or a user hitting `/admin/*` |
| **Nested-resource bypass** | `GET /payments/pi_mine/refunds/re_theirs` — the parent is yours, the child isn't (Lesson 07) |
| **Filter bypass** | `GET /payments?merchant_id=someone_else` — you trusted a query param over the token |

All five have the same root cause and the same fix.

---

## 3. The fix: make it structural

Four layers, from weakest to strongest. **Do at least the first three.**

### Layer 1 — Never take tenancy from the request

```ts
// ❌ Any of these is a cross-tenant hole
const merchantId = req.body.merchant_id;
const merchantId = req.query.merchant;
const merchantId = req.header("x-merchant-id");
const merchantId = decodedCursor.merchantId;

// ✅ Exactly one source of truth: the verified credential
const merchantId = req.principal.merchantId;
```

This is the rule from [Lesson 12](12-authentication-landscape.md), and it's absolute. If a client can *name* the tenant, the tenant boundary doesn't exist. The only exception is a genuine admin/impersonation path, which needs its own explicit permission, its own audit log entry, and ideally its own endpoint namespace.

### Layer 2 — Tenancy in the repository signature, not the handler

The cheapest structural improvement available: make it **impossible to call the data layer without a tenant.**

```ts
// ❌ A function that CAN be called unsafely will eventually be called unsafely
db.payments.findById(id);

// ✅ Tenant is a required parameter. Forgetting it is a compile error.
class PaymentRepo {
  async findById(id: PaymentId, merchantId: MerchantId): Promise<Payment | null> {
    return this.db.oneOrNone(
      `SELECT * FROM payments WHERE id = $1 AND merchant_id = $2`, [id, merchantId]);
  }

  async list(merchantId: MerchantId, opts: ListOpts): Promise<Payment[]> { /* ... */ }
}
```

Even better: **a per-request scoped repository**, so the tenant can only be supplied once, at the boundary.

```ts
/** Built once per request from the authenticated principal. */
export function scopedRepos(merchantId: MerchantId) {
  return {
    payments: {
      findById: (id: PaymentId) =>
        db.oneOrNone(`SELECT * FROM payments WHERE id=$1 AND merchant_id=$2`, [id, merchantId]),
      list: (o: ListOpts) => listPayments(merchantId, o),
      updateStatus: (id: PaymentId, from: Status, to: Status) =>
        db.oneOrNone(
          `UPDATE payments SET status=$3, version=version+1
            WHERE id=$1 AND merchant_id=$2 AND status=$4 RETURNING *`,
          [id, merchantId, to, from]),
    },
    refunds: { /* ... */ },
  };
}

// middleware
app.use((req, _res, next) => { req.repos = scopedRepos(req.principal.merchantId); next(); });

// handler — there is no unscoped call available to write
const payment = await req.repos.payments.findById(req.params.id as PaymentId);
```

**Now a developer physically cannot write the BOLA bug in a handler**, because the unscoped query isn't in scope. That's the difference between a rule and a guarantee, and it's the answer that impresses in an interview: *"I don't rely on remembering the check — I remove the ability to skip it."*

### Layer 3 — Postgres Row-Level Security (defence in depth at the database)

```sql
ALTER TABLE payments ENABLE ROW LEVEL SECURITY;
ALTER TABLE payments FORCE ROW LEVEL SECURITY;      -- applies even to the table owner

CREATE POLICY tenant_isolation ON payments
  USING (merchant_id = current_setting('app.merchant_id', true)::uuid);

-- Per request/transaction, set the tenant:
SET LOCAL app.merchant_id = 'mrc_9s2k';
```

Now **even a forgotten `WHERE` clause returns zero rows.** The database enforces the boundary.

Costs, stated honestly: it requires setting the variable on every connection *after* checkout from the pool (easy to get wrong with pooling — use `SET LOCAL` inside the transaction), it makes some query plans harder to reason about, and migrations/admin jobs need a deliberate bypass role. **Worth it for anything holding financial or health data**; often skipped elsewhere. Knowing both the mechanism and the pooling gotcha is a strong signal.

### Layer 4 — Return 404, not 403, for cross-tenant access

```ts
const payment = await req.repos.payments.findById(id);
if (!payment) throw ApiError.notFound("payment_not_found", `No payment with id '${id}'`);
```
Note this falls out for free from Layer 2: a scoped query returns nothing, so you naturally 404 without ever learning whether the object exists elsewhere. **A 403 would confirm existence and enable enumeration** (Lesson 10).

---

## 4. Permission models: RBAC vs ABAC vs ReBAC

Once tenancy is solved, you still need *"may this user do this action?"* Three models, and you should know when each breaks.

### RBAC — Role-Based Access Control
Users have roles; roles have permissions.

```ts
const ROLE_PERMISSIONS = {
  owner:      ["payments:*", "refunds:*", "payouts:*", "members:*", "keys:*", "settings:*"],
  admin:      ["payments:*", "refunds:*", "payouts:read", "members:read", "keys:*"],
  developer:  ["payments:read", "payments:write", "refunds:read", "keys:*"],
  accountant: ["payments:read", "refunds:read", "payouts:read", "reports:*"],
  viewer:     ["payments:read", "refunds:read"],
} as const satisfies Record<Role, readonly string[]>;

export function can(role: Role, permission: string): boolean {
  return ROLE_PERMISSIONS[role].some(p =>
    p === permission || (p.endsWith(":*") && permission.startsWith(p.slice(0, -1))));
}
```

**Pros:** simple, auditable, easy to explain to an auditor and to a customer. **Covers ~80% of real needs.**
**Where it breaks:** *"an accountant can view payouts, but only for the branch they manage, and only if the payout is over ₹50,000, and only during business hours."* Encode that in roles and you get `accountant_branch_mumbai_large_payouts`, and then 400 more roles. That explosion is the signal you've outgrown RBAC.

### ABAC — Attribute-Based Access Control
Decisions from attributes of the subject, object, action and environment.

```ts
type Decision = { allow: boolean; reason: string };

export function canRefund(user: User, payment: Payment, now: Date): Decision {
  if (user.merchantId !== payment.merchantId) return { allow: false, reason: "different_tenant" };
  if (!can(user.role, "refunds:write"))       return { allow: false, reason: "role_lacks_permission" };
  if (payment.status !== "succeeded")         return { allow: false, reason: "payment_not_refundable" };
  if (daysSince(payment.capturedAt, now) > 180) return { allow: false, reason: "refund_window_expired" };
  if (payment.amountMinor > 500_000 && user.role !== "owner")
    return { allow: false, reason: "amount_exceeds_role_limit" };
  return { allow: true, reason: "ok" };
}
```

**Pros:** expressive; handles the real world. **Cons:** harder to audit ("who can refund this?" needs evaluation, not a lookup), and the logic sprawls unless you centralise it.

Note the design detail: **returning a `reason`, not a boolean.** That gives you precise 403 messages, an audit trail, and testability. Always do this.

### ReBAC — Relationship-Based Access Control
Permission derives from a graph of relationships: *"you can view a document if you're a member of a team that has access to a folder that contains it."*

That's Google Docs, GitHub, and Notion. The reference implementation is **Google Zanzibar**, and its open-source descendants are **SpiceDB**, **OpenFGA** and **Ory Keto**.

**Use it when** permissions are inherited through nesting or sharing. **Don't** build it yourself — the hard parts are consistency and latency at scale, and Zanzibar's paper exists precisely because those are hard.

### Choosing

```
Flat tenants, a handful of roles                    → RBAC
Roles + conditions (amount, state, time, ownership) → RBAC for the coarse gate + ABAC for the fine one
Nested/shared resources, user-granted sharing       → ReBAC (SpiceDB / OpenFGA)
Regulated environment needing provable audit        → policy-as-code (OPA/Cedar) + full decision logs
```

**The practical answer for 90% of products, and the one I'd give in an interview:** *"RBAC for the endpoint-level gate, plus an explicit object-level policy function that returns a reason. RBAC alone can't express object conditions, and pure ABAC scattered through handlers becomes unauditable."*

---

## 5. Where authorization checks live

Three enforcement points; you need all three, and knowing why is the lesson.

```ts
v1.post("/payments/:id/refunds",
  authenticate,                     // 1. WHO           → 401
  requireScope("refunds:write"),    // 2. app-level     → 403 insufficient_scope
  requirePermission("refunds:write"), // 3. role-level  → 403 (BFLA defence)
  createRefund,                     // 4. object-level inside → 403/404 (BOLA defence)
);

async function createRefund(req: Request, res: Response) {
  // Object-level: fetch scoped (so cross-tenant is a 404), then check the policy.
  const payment = await req.repos.payments.findById(req.params.id as PaymentId);
  if (!payment) throw ApiError.notFound("payment_not_found", `No payment with id '${req.params.id}'`);

  const decision = canRefund(req.principal, payment, new Date());
  if (!decision.allow) {
    await audit.log("refund_denied", { userId: req.principal.userId, paymentId: payment.id, reason: decision.reason });
    throw ApiError.forbidden(decision.reason, refundDenialMessage(decision.reason));
  }
  // ...
}
```

| Layer | Catches | Skipping it means |
|---|---|---|
| **Endpoint/role** (middleware) | BFLA — a viewer calling a write endpoint | Anyone authenticated can call anything |
| **Scope** (middleware) | An OAuth app exceeding its grant | Granted scopes are decoration |
| **Object** (in the handler, or the scoped repo) | **BOLA** | Cross-tenant data access |
| **Field** (in the input schema) | BOPLA / mass assignment | Users editing their own `role` or `balance` |

**Middleware alone is never enough**, because middleware sees the URL, not the object. `POST /payments/:id/refunds` passing a role check tells you the caller may refund *something*, not *this*.

### The gateway question
API gateways can do authentication and coarse authorization, and doing so is good (cheap rejection at the edge). **But object-level authorization cannot move to the gateway**, because the gateway doesn't know who owns `pi_3Nx8`. So: **authenticate at the edge, authorize objects in the service.** That division is a clean, senior-sounding answer.

---

## 6. Tenant isolation strategies

Multi-tenancy is an architecture decision with an authorization consequence.

| Strategy | How | Isolation | Cost |
|---|---|---|---|
| **Shared schema, `tenant_id` column** | One table, filtered by `tenant_id` | Weakest — one bug leaks | Cheapest; best resource efficiency. **The default** |
| **+ Row-Level Security** | Above, enforced by the DB | Strong | Small complexity + pooling care |
| **Schema per tenant** | `tenant_9s2k.payments` | Stronger; per-tenant migration risk | Hundreds of schemas hurt migrations and connection pooling |
| **Database per tenant** | Separate DB, possibly separate instance | Strongest; per-tenant backup/restore and residency | Expensive; painful at 10,000 tenants |
| **Cell / silo architecture** | Whole stack per tenant group | Strongest; blast-radius isolation | Most expensive; what AWS itself does |

**The realistic path:** shared schema + `tenant_id` + RLS for everyone, with **database-per-tenant available as an enterprise tier** for customers who require data residency or contractual isolation. Being able to say *"I'd start shared with RLS and offer dedicated for enterprise, because the cost curve only justifies isolation when the customer pays for it"* is exactly the trade-off framing interviews reward.

Also design for these from the start, because retrofitting is expensive:
- **`tenant_id` on every table**, in every index, as the **leading column** (Lesson 08 — it's your most selective predicate).
- **No cross-tenant foreign keys.** Ever.
- **Tenant ID in every log line**, so you can answer "what did merchant X experience?"
- **Per-tenant rate limits and quotas**, so one tenant can't starve the rest ([Lesson 19](../04-production/19-rate-limiting.md)).
- **A tenant-deletion path** (GDPR/DPDP right to erasure). If tenant data is smeared across 40 tables with no consistent key, deletion becomes a project.

---

## 7. Testing authorization (the part that's usually missing)

Authorization is the least-tested and highest-risk code in most APIs. The fix is a **matrix test** that runs against every endpoint automatically.

```ts
// One fixture: two tenants, and every role within one of them.
const ACTORS = ["owner", "admin", "developer", "accountant", "viewer", "other_tenant_owner", "anonymous"] as const;

const CASES: Array<{ name: string; call: (as: Actor) => Promise<Res>; expect: Record<Actor, number> }> = [
  {
    name: "GET /payments/:id (own)",
    call: as => api(as).get(`/v1/payments/${fixtures.tenantA.payment.id}`),
    expect: { owner: 200, admin: 200, developer: 200, accountant: 200, viewer: 200,
              other_tenant_owner: 404,          // ← BOLA: not 403, so existence isn't leaked
              anonymous: 401 },
  },
  {
    name: "POST /payments/:id/refunds",
    call: as => api(as).post(`/v1/payments/${fixtures.tenantA.payment.id}/refunds`, { amount_minor: 100 }),
    expect: { owner: 201, admin: 201, developer: 403, accountant: 403, viewer: 403,
              other_tenant_owner: 404, anonymous: 401 },
  },
  {
    name: "GET /payments/:pid/refunds/:rid (refund belongs to ANOTHER payment)",
    call: as => api(as).get(`/v1/payments/${fixtures.tenantA.payment.id}/refunds/${fixtures.tenantA.otherRefund.id}`),
    expect: { owner: 404, admin: 404, developer: 404, accountant: 404, viewer: 404,
              other_tenant_owner: 404, anonymous: 401 },   // ← nested-resource bypass
  },
  {
    name: "PATCH /me { role: 'owner' }  (privilege escalation via mass assignment)",
    call: as => api(as).patch(`/v1/me`, { role: "owner" }),
    expect: { owner: 422, admin: 422, developer: 422, accountant: 422, viewer: 422,
              other_tenant_owner: 422, anonymous: 401 },   // ← BOPLA: unknown_field, never 200
  },
  {
    name: "GET /payments?merchant_id=<tenantB>  (filter bypass)",
    call: as => api(as).get(`/v1/payments?merchant_id=${fixtures.tenantB.id}`),
    expect: { owner: 400, admin: 400, developer: 400, accountant: 400, viewer: 400,
              other_tenant_owner: 400, anonymous: 401 },   // unknown_filter, never tenantB's data
  },
];

describe.each(CASES)("$name", ({ call, expect: table }) => {
  test.each(ACTORS)("as %s", async actor => {
    const res = await call(actor);
    expect(res.status).toBe(table[actor]);
  });
});
```

Then add the rule that makes it durable:

```ts
// A route with no explicit authorization test is a build failure.
test("every registered route appears in the authorization matrix", () => {
  const routes = listRegisteredRoutes(app);        // walk the router
  const covered = new Set(CASES.map(c => c.name.match(/\w+ \/[^\s(]+/)?.[0]));
  const missing = routes.filter(r => !covered.has(`${r.method} ${r.path}`));
  expect(missing).toEqual([]);
});
```

**That last test is the single highest-value test in an API codebase.** It converts "we should remember to check authorization" into "you cannot merge an unauthorized endpoint." Describe it in an interview and you sound like someone who has cleaned up after a breach.

---

## 8. Production rules

| Rule | Why |
|---|---|
| **Tenancy comes from the credential. Never from body, query, header or cursor** | Otherwise the boundary doesn't exist |
| **Make the tenant a required argument of every data-access function** | Turns a forgotten check into a compile error |
| **Prefer per-request scoped repositories** | Removes the ability to write the bug |
| **Enable Postgres RLS for sensitive data, with `SET LOCAL` inside the transaction** | Defence in depth against a single forgotten clause; pooling makes session-level `SET` unsafe |
| **404, not 403, for cross-tenant** | Prevents existence disclosure and ID enumeration |
| **Validate the entire parent chain on nested routes** | `/payments/mine/refunds/theirs` |
| **`.strict()` input schemas; never bind a body onto an entity** | BOPLA / privilege escalation |
| **Check permissions at endpoint *and* object level** | Middleware can't see the object |
| **Policy functions return a reason, not a boolean** | Precise 403s, audit trails, testability |
| **Log every authorization denial with actor, object and reason** | Denials are your earliest breach signal |
| **Run an actor × endpoint authorization matrix in CI, and fail on uncovered routes** | The only way this stays true as the API grows |
| **Never build an "internal" endpoint that skips authorization** | It will be exposed. It always is |
| **Admin impersonation is a separate, audited, permissioned path** | Not a flag on the normal path |

---

## 9. Interview traps

**Q1. "What's the most common API vulnerability?"**
**BOLA** — broken object level authorization, OWASP API Top 10 #1. Explain it in one line (authenticated caller, someone else's object ID, 200 OK) and say *why* it's #1: trivial to exploit, invisible in review, and every new endpoint is a fresh chance.

**Q2. "How do you prevent it?"**
Lead with **structural** prevention, not vigilance: tenancy from the credential only, tenant as a required repository parameter, per-request scoped repos, RLS as a backstop, 404-not-403, and an authorization matrix in CI that fails on uncovered routes. *"I don't rely on remembering the check; I remove the ability to skip it."*

**Q3. "You use UUIDs, so IDs can't be guessed. Is that enough?"**
No. Unguessable IDs are defence in depth, not authorization. IDs leak through exports, screenshots, support tickets, logs, misrouted webhooks and ex-employees. **Security through unguessability isn't security.**

**Q4. "RBAC vs ABAC?"**
RBAC: roles → permissions; simple and auditable; breaks when conditions enter (amount, state, time, ownership) because roles explode combinatorially. ABAC: decisions from attributes; expressive but harder to audit. **Recommend both: RBAC as the coarse endpoint gate, an object-level policy function for the fine-grained rules.**

**Q5. "How would you implement Google Docs-style sharing?"**
ReBAC — permissions from a relationship graph, with inheritance (document ← folder ← team ← org) and direct grants. Name Zanzibar and one implementation (SpiceDB/OpenFGA), and say you wouldn't build it yourself because the hard parts are consistency and check latency at scale.

**Q6. "403 or 404 for a resource that exists but isn't theirs?"**
404 when existence is sensitive — for tenant data, essentially always — because 403 confirms existence and enables enumeration. 403 when the caller legitimately knows it exists but lacks permission for that action (a viewer trying to delete something they can see). GitHub's private repos are the canonical 404 example.

**Q7. "Where should authorization live — gateway, middleware or service?"**
Authentication and coarse role/scope checks at the edge (cheap rejection). **Object-level authorization must live in the service**, because only the service knows who owns the object. Mention that pushing object authorization to a gateway is a common architecture mistake.

**Q8. "A user PATCHes their own profile with `{"role": "owner"}`. What happens?"**
With a `.strict()` input DTO containing only client-settable fields: `422 unknown_field`. Without it, and with `Object.assign`, they're now an owner. That's BOPLA, and it's the same mass-assignment class that compromised GitHub in 2012.

**Q9. "How do you test authorization?"**
The actor × endpoint matrix from §7, plus the meta-test that every registered route must appear in it. Add negative tests as first-class: *"the test that Bob **cannot** read Alice's payment is more valuable than the test that Alice can."*

**Q10. "How do you support an enterprise customer who demands data isolation?"**
Explain the ladder (shared + `tenant_id` → RLS → schema-per-tenant → DB-per-tenant → cells), then commit: shared + RLS by default, DB-per-tenant as a paid enterprise tier for residency/contractual requirements. Note the operational costs you're accepting — migrations across N databases, per-tenant backup and restore, and connection-pool pressure.

---

## 10. Build & break

### Build — scoped repositories for Ledger
Refactor every data-access call so that:
1. No function that reads or writes tenant data can be called without a `merchantId`.
2. Handlers use `req.repos.*`, and the unscoped `db` object isn't importable from the handler layer (enforce it with an ESLint `no-restricted-imports` rule — that's how the guarantee survives your next teammate).
3. Nested reads verify the parent chain.
4. Every input schema is `.strict()` and contains only client-settable fields.

Then enable RLS on `payments`, `refunds` and `customers`, and prove it: comment out a `WHERE merchant_id = $2` and confirm the query returns **zero rows** instead of someone else's data.

### Build — the authorization matrix
Implement §7 for every existing endpoint, including the route-coverage meta-test. Expect to find at least one real bug — that's the point of the exercise.

### Break — attack your own API
Two tenants, A and B, both authenticated. As B, try:

1. `GET /v1/payments/{A_payment_id}` → must be **404**
2. `GET /v1/payments?merchant_id={A_id}` → must be **400 unknown_filter**, never A's data
3. `POST /v1/payments {"merchant_id": "{A_id}", "amount_minor": 100}` → must be **422 unknown_field**
4. `GET /v1/payments/{B_payment}/refunds/{A_refund}` → must be **404**
5. `PATCH /v1/me {"role": "owner", "merchant_id": "{A_id}"}` → must be **422**
6. Tamper with a pagination cursor to inject A's ID → must return **B's own data or 400**, never A's
7. As a `viewer` in tenant B: `POST /v1/payments/{B_payment}/refunds` → must be **403**
8. `DELETE /v1/api-keys/{A_key_id}` → must be **404**

Write each of these as a permanent test. **Any one of them returning 200 is a reportable vulnerability** — and the exercise of writing them is how you internalise that authorization is per-object, not per-request.

### Explain out loud (2 minutes)
1. What BOLA is, and why it's #1.
2. Four structural defences, in order of strength.
3. RBAC vs ABAC vs ReBAC, and when each breaks.
4. Why object-level authorization can't live in the gateway.
5. The one test you'd add to a codebase to stop this class of bug recurring.

---

## What's next

Authorization is the biggest single risk, but it isn't the only one. Next: the rest of the OWASP API Top 10 in practice — injection, SSRF (the one that gets cloud metadata stolen), mass assignment, unrestricted resource consumption, and the CORS/secrets mistakes that show up in every security review.

Next → **[Lesson 16: Hardening — OWASP API Top 10 in practice](16-owasp-and-hardening.md)**
