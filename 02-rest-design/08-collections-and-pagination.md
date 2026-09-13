# Lesson 08 — Collections: pagination, filtering, sorting, search

> **Why this lesson exists:** the list endpoint is the single most-requested, most-expensive, and most-often-broken endpoint in any API. It's also the highest-signal interview question in API design, because *"design `GET /payments`"* looks trivial and quietly requires you to know database indexes, cursor encoding, concurrent-write semantics, and injection defence. Get it wrong and you don't find out until a customer has 4 million rows.

**Time:** ~95 minutes · **Prereq:** Lesson 07

---

## 1. The idea in one sentence

> **A collection endpoint is a query API, and every design decision in it is a trade between the client's convenience and your database's index — with `OFFSET` being the trade that looks free and isn't.**

---

## 2. Why offset pagination breaks (both ways)

Everyone starts here:

```
GET /v1/payments?page=3&limit=20
→ SELECT * FROM payments WHERE merchant_id = $1 ORDER BY created_at DESC LIMIT 20 OFFSET 40
```

It works beautifully until it doesn't, and it fails in **two independent ways**. Knowing both is what separates a real answer from a memorised one.

### Failure 1 — Performance: `OFFSET` is a scan, not a seek

`OFFSET 40` doesn't skip to row 41. The database **reads and discards** rows 1–40. So:

| Page | Rows the DB must read | Latency (typical, 4M rows) |
|---|---|---|
| 1 (`OFFSET 0`) | 20 | ~2 ms |
| 100 (`OFFSET 2000`) | 2,020 | ~15 ms |
| 10,000 (`OFFSET 200000`) | 200,020 | ~800 ms |
| 200,000 (`OFFSET 4000000`) | 4,000,020 | **~20 s, or a timeout** |

The cost is **O(offset)**, which means your endpoint gets slower the deeper anyone goes. And it's not hypothetical: an export script that walks every page will start fast and end in timeouts, and the *last* pages are the ones that hammer your database hardest.

Worse, `COUNT(*)` for the total is a **second** full scan of the matching set. Two expensive queries per request.

### Failure 2 — Correctness: drift

This is the failure people forget, and it's the more interesting one in an interview.

```
t0: Client fetches page 1 → payments [100, 99, 98, ... 81]   (newest first)
t1: Three new payments arrive → [103, 102, 101, 100, 99, ...]
t2: Client fetches page 2 (OFFSET 20) → starts at 83
```
Payments **84, 83, 82, 81** — wait: 81, 82, 83 were on page 1, and now appear again on page 2. Meanwhile if rows were *deleted*, items shift the other way and the client **never sees them at all**.

So with offset pagination over a live, newest-first dataset:
- **Duplicates** when rows are inserted
- **Silently skipped rows** when rows are deleted
- Both, invisibly, with no error

For a UI showing page numbers, mild annoyance. **For a script that syncs your payments into an accounting system, it means missing transactions.** That's the sentence to say out loud: *"offset pagination on a live dataset silently drops records, so it's unusable for anything that must be complete."*

---

## 3. Cursor (keyset) pagination — the correct default

Instead of *"skip 40 rows"*, say *"give me rows after this specific position."*

```
GET /v1/payments?limit=20
→ { "data": [...], "has_more": true, "next_cursor": "eyJjIjoiMjAyNi0wMy0xNFQwOToz..." }

GET /v1/payments?limit=20&cursor=eyJjIjoiMjAyNi0wMy0xNFQwOToz...
```

The SQL:
```sql
-- Page 1
SELECT * FROM payments
 WHERE merchant_id = $1
 ORDER BY created_at DESC, id DESC
 LIMIT 21;                       -- ← fetch limit+1 to know if there's more

-- Page N: seek directly to the position
SELECT * FROM payments
 WHERE merchant_id = $1
   AND (created_at, id) < ($2, $3)      -- ← row-value comparison. THIS is the trick
 ORDER BY created_at DESC, id DESC
 LIMIT 21;
```

**Why this is O(1) instead of O(offset):** with an index on `(merchant_id, created_at DESC, id DESC)`, the database *seeks* to the exact position in the B-tree and reads 21 rows. Page 200,000 costs exactly what page 1 costs. That's the whole point, and it's the sentence that wins the interview.

### The three details that make it actually work

**1. You must have a unique tiebreaker.** `created_at` alone is not unique — two payments in the same millisecond, and your cursor either skips or repeats them. Always sort by `(sort_column, unique_id)` and compare both. This is the #1 bug in hand-rolled cursor pagination.

**2. Use row-value comparison, not naive AND/OR.** `WHERE (created_at, id) < ($2, $3)` is correct and index-friendly in Postgres. The hand-expanded version is:
```sql
WHERE created_at < $2 OR (created_at = $2 AND id < $3)
```
Equivalent, but query planners often handle the row-value form better. MySQL 8 supports row values too; on older MySQL use the expanded form.

**3. The cursor must encode the sort position, and it must be opaque.**

```ts
type CursorPayload = {
  /** The sort key values of the last row returned, in sort order */
  k: (string | number)[];
  /** Fingerprint of the query — sort + filters. Prevents cursor reuse across queries */
  q: string;
};

export function encodeCursor(p: CursorPayload): string {
  return Buffer.from(JSON.stringify(p)).toString("base64url");
}

export function decodeCursor(s: string, expectedQ: string): CursorPayload {
  let p: CursorPayload;
  try {
    p = JSON.parse(Buffer.from(s, "base64url").toString("utf8"));
  } catch {
    throw new BadRequest("invalid_cursor", "Cursor is malformed");
  }
  // A cursor from a differently-sorted/filtered query is meaningless — and a
  // client that mixes them will silently get wrong pages. Fail loudly instead.
  if (p.q !== expectedQ) {
    throw new BadRequest("cursor_query_mismatch",
      "This cursor was issued for a different filter/sort combination");
  }
  return p;
}
```

**Why base64 rather than a readable `?after_id=pi_3Nx8`?** Two reasons that matter:
1. **It signals opacity.** A readable cursor invites clients to construct their own, and then you can never change the pagination scheme (Hyrum's Law). Stripe, GitHub, Slack and Twitter all return opaque cursors for exactly this reason.
2. **Composite keys need a container.** Once you're sorting by `(amount, created_at, id)` you can't express the position in a single scalar param anyway.

**Should you sign the cursor?** If a tampered cursor could leak data, yes — but note the correct fix is that **your query is always tenant-scoped anyway** (`WHERE merchant_id = $1` comes from the token, never the cursor). Then a tampered cursor can only produce a wrong page of the caller's *own* data, which is harmless. **Never put the tenant ID inside the cursor and trust it.** That's a one-line cross-tenant data breach, and it's a genuinely good thing to volunteer in an interview.

### The costs of cursor pagination (be honest about these)

| Cost | Detail | Mitigation |
|---|---|---|
| **No random access** | You cannot jump to "page 500". Only next/previous | Usually fine — nobody clicks page 500. If the UI needs it, that's an argument for search+filter instead |
| **No total count** (cheaply) | See §4 | `has_more`, or an approximate count |
| **Sort column must be indexed and stable** | Sorting by a mutable column (`amount`) means a row can move between pages if edited mid-scan | Accept it, or offer immutable sorts only for exports |
| **Bidirectional needs care** | `previous` means reversing the comparison and the `ORDER BY`, then re-reversing the results | Implement it once, in a helper |

> **Rule for choosing:** *"Cursor pagination by default. Offset only when the dataset is small and bounded (under ~10k rows) and the UI genuinely needs page numbers — an admin table of your 50 API keys, say. Never offset for anything a client might iterate completely."*

---

## 4. Total counts: the question everyone asks

Product wants *"Showing 1–20 of 4,213,904"*. That number costs you a full index scan on every request.

Your options, in order of how much I'd recommend them:

| Option | Cost | Honesty | When |
|---|---|---|---|
| **`has_more: true`** | Free (fetch `limit+1`) | Perfectly honest | **The default.** Infinite scroll, "Load more" |
| **Approximate count** | Cheap (`reltuples` in Postgres, or a maintained counter) | Say "about 4.2M" | Dashboards, where precision doesn't matter |
| **Cached exact count** | One expensive query, reused for N minutes | Slightly stale | Slow-changing collections |
| **Count only when filtered narrowly** | Cheap when the filter is selective | Exact when it's cheap | Return `total` only if it's under a threshold, else `null` |
| **Exact count every request** | Full scan, doubles your query load | Exact | Small collections only |

The pattern I'd ship:
```json
{
  "data": [ ... ],
  "has_more": true,
  "next_cursor": "eyJrIjpb...",
  "total_count": null,
  "total_count_hint": "not_computed"
}
```
And document: *"`total_count` is populated only when the result set is under 10,000; otherwise use `has_more`."* That's a design decision with a reason, which is exactly what a review wants.

> **Interview gold:** *"I'd push back on the requirement. 'Showing 1–20 of 4.2 million' is a number nobody acts on — the user's next action is 'search' or 'load more', never 'go to page 210,695'. If product insists, I'd serve an approximate count and label it as approximate, because an exact count doubles the cost of the most-called endpoint in the system to satisfy a cosmetic requirement."* Interviewers love this because it's engineering judgement, not just technique.

---

## 5. Filtering — conventions and the injection trap

### The four conventions in the wild

**A. Flat equality (simplest, most common)**
```
GET /payments?status=succeeded&currency=usd
```
Clean, readable, and covers 80% of real needs. Repeated keys = `IN`: `?status=succeeded&status=pending`.

**B. Suffix/bracket operators (Stripe's style — my recommendation)**
```
GET /payments?created[gte]=1773480600&created[lt]=1773567000&amount[gte]=1000
GET /payments?status=succeeded&status=pending          # IN
```
Explicit, discoverable, easy to parse and validate, no query language to learn. Slight ugliness in the URL; worth it.

**C. Operator-in-value**
```
GET /payments?amount=gte:1000&created=lt:2026-03-15
```
Compact. Ambiguous the moment a value legitimately contains a colon.

**D. A query language (RSQL / OData / JSON:API filters)**
```
GET /payments?filter=amount=ge=1000;status=in=(succeeded,pending)
GET /payments?$filter=amount ge 1000 and status eq 'succeeded'
```
Extremely powerful; you have now built a database query language as a public API. Costs: a parser to maintain, a much larger attack surface, unpredictable query plans (a client can write a query that table-scans), and an API surface you can never simplify. **Only worth it for genuinely ad-hoc analytical APIs** — and then consider whether GraphQL or a real reporting endpoint is more honest.

### The rules that keep filtering safe and fast

**1. Allowlist every filterable field. Never reflect client input into SQL identifiers.**

```ts
// ❌ Catastrophic — SQL injection via a column name, and unindexed scans
const sql = `SELECT * FROM payments WHERE ${req.query.field} = '${req.query.value}'`;

// ✅ Allowlist maps client-facing names → column + type + allowed operators
const FILTERABLE = {
  status:    { col: "status",     type: "enum",  ops: ["eq", "in"],  values: ["pending","succeeded","failed","refunded"] },
  currency:  { col: "currency",   type: "enum",  ops: ["eq", "in"],  values: ["usd","eur","gbp","inr"] },
  amount:    { col: "amount_minor", type: "int", ops: ["eq","gte","lte","gt","lt"] },
  created:   { col: "created_at", type: "date",  ops: ["gte","lte","gt","lt"] },
  customer:  { col: "customer_id", type: "id",   ops: ["eq"] },
} as const;
```

Note what the allowlist buys you beyond injection safety: **it's also your index contract.** A field is filterable *because* you have an index for it. That connection is the thing seniors articulate and juniors don't.

**2. Parameterise every value.** Always. Even after allowlisting the column.

**3. Bound everything.** Max `limit` (100), max number of filters, max date range for expensive queries, max `IN` list length. Every unbounded input is a resource-exhaustion vector.

**4. Reject unknown filters — loudly.**
```
GET /payments?statuss=succeeded     ← typo
```
Silently ignoring this returns **all** payments while the client believes they're filtered. In a financial reconciliation script, that's a wrong answer presented as a right one.

> This is the exception to the tolerant-reader principle you'll learn in [Lesson 11](11-versioning-and-evolution.md), and the distinction is important enough to be an interview answer on its own: **be tolerant about unknown fields in a request *body* (they're additive), be strict about unknown *query parameters* (they change which data you return).** A misspelled body field means one attribute is missing; a misspelled filter means the entire result set is wrong.

---

## 6. Sorting

```
GET /payments?sort=-created_at          # - prefix = descending (most common convention)
GET /payments?sort=-created_at,amount   # multi-key
```

Alternative: `?sort_by=created_at&order=desc`. Both fine; the `-` prefix composes better for multi-key sorts.

The four rules:

1. **Allowlist sortable fields**, same reasoning as filters — every sortable field needs a supporting index or it's a table scan you've made publicly callable.
2. **Always append a unique tiebreaker** (`, id DESC`). Without it, sorting by `amount` where 400 payments are 4999 gives you a **non-deterministic order**, which means cursor pagination will duplicate and skip rows. This is a subtle, real bug and a great thing to mention unprompted.
3. **Pick a documented default** (`-created_at` is almost always right) and never leave it to the database. `ORDER BY` absent means "any order the planner likes," and it *will* change when the plan changes.
4. **`NULLS FIRST`/`LAST` is a decision.** Postgres defaults to `NULLS LAST` for `ASC` and `NULLS FIRST` for `DESC`, which surprises people. Be explicit.

```sql
CREATE INDEX idx_payments_merchant_created
    ON payments (merchant_id, created_at DESC, id DESC);
-- Every sort+filter combination you expose needs an index like this,
-- with the tenant column FIRST. That's the real constraint on your API surface.
```

---

## 7. Search

**Simple search** — one query param, fuzzy across a few fields:
```
GET /payments?q=alice
```
Behind it: a full-text index (Postgres `tsvector`), a trigram index for fuzzy matching, or a dedicated engine (Elasticsearch/OpenSearch/Meilisearch). Document *which fields* `q` searches — otherwise clients can't reason about results.

**Complex search** — when filters exceed URL limits or need boolean logic:
```http
POST /v1/payments/search
Content-Type: application/json

{
  "filters": {
    "status": ["succeeded", "refunded"],
    "created": { "gte": "2026-01-01T00:00:00Z", "lt": "2026-04-01T00:00:00Z" },
    "amount":  { "gte": 1000 },
    "metadata": { "campaign": "spring_sale" }
  },
  "sort": ["-created_at"],
  "limit": 50,
  "cursor": null
}
```

You're breaking two rules deliberately (a `POST` for a read; not cacheable). Say so in the docs, and mention in an interview that the fully-RESTful alternative is a **saved search resource**:
```
POST /v1/searches      → 201 + /v1/searches/sr_1
GET  /v1/searches/sr_1/results?cursor=...
```
which is cacheable, shareable, re-runnable, and gives you a natural place to hang async execution for expensive queries. That's the answer that sounds like you've built this before.

---

## 8. The response envelope

```json
{
  "object": "list",
  "data": [ { "object": "payment", "id": "pi_3Nx8", ... } ],
  "has_more": true,
  "next_cursor": "eyJrIjpbIjIwMjYtMDMtMTRUMDk6MzA6MDBaIiwicGlfM054OCJdfQ",
  "prev_cursor": null,
  "total_count": null
}
```

Design decisions in there, each with a reason:

| Choice | Reason |
|---|---|
| **Object at the top level, never a bare array** | Room to add metadata later without a breaking change (Lesson 05) |
| **`data` as the key** | Stripe/JSON:API convention; `items`/`results` are equally fine — just be consistent across every collection |
| **`object` type discriminator** | Lets a typed client discriminate a heterogeneous list, and makes logs readable. Maps perfectly onto a TypeScript discriminated union ([TS L09](../../TypeScript/README.md)) |
| **`has_more` not `total_pages`** | Honest and free |
| **Cursors returned, never constructed by the client** | The one hypermedia pattern that genuinely works (Lesson 06) |

For an empty result: **`{"data": [], "has_more": false}` with `200`.** Not `404` — the collection exists, it's just empty. This is a real interview question and people get it wrong surprisingly often: *404 means "no such collection", not "no matching rows".*

---

## 9. The reference implementation

```ts
import { z } from "zod";

const FILTERABLE = {
  status:   { col: "status",        ops: ["eq","in"],                    schema: z.enum(["pending","succeeded","failed","refunded"]) },
  currency: { col: "currency",      ops: ["eq","in"],                    schema: z.enum(["usd","eur","gbp","inr"]) },
  amount:   { col: "amount_minor",  ops: ["eq","gte","lte","gt","lt"],  schema: z.coerce.number().int().nonnegative() },
  created:  { col: "created_at",    ops: ["gte","lte","gt","lt"],       schema: z.coerce.date() },
  customer: { col: "customer_id",   ops: ["eq"],                         schema: z.string().startsWith("cus_") },
} as const;

const SORTABLE = ["created_at", "amount_minor"] as const;

const ListQuery = z.object({
  limit:  z.coerce.number().int().min(1).max(100).default(20),   // bounded!
  cursor: z.string().max(512).optional(),
  sort:   z.string().default("-created_at"),
}).passthrough();   // filters arrive as arbitrary keys; we validate them explicitly below

export async function listPayments(req: Request, res: Response) {
  const parsed = ListQuery.safeParse(req.query);
  if (!parsed.success) return res.status(400).json(validationProblem(parsed.error));
  const { limit, cursor, sort, ...rest } = parsed.data;

  // --- Filters: allowlist, then parameterise. Unknown keys are an ERROR. ---
  const where: SqlFragment[] = [sql`merchant_id = ${req.merchantId}`];   // tenancy first, from the token
  for (const [rawKey, rawVal] of Object.entries(rest)) {
    const m = /^(\w+)(?:\[(\w+)\])?$/.exec(rawKey);
    const field = m?.[1];
    const op = m?.[2] ?? "eq";
    const def = field && (FILTERABLE as any)[field];

    if (!def) {
      return res.status(400).json(problem("unknown_filter",
        `Unknown filter '${rawKey}'. Allowed: ${Object.keys(FILTERABLE).join(", ")}`));
    }
    if (!def.ops.includes(op)) {
      return res.status(400).json(problem("unsupported_operator",
        `Operator '${op}' is not supported for '${field}'. Allowed: ${def.ops.join(", ")}`));
    }
    const values = Array.isArray(rawVal) ? rawVal : [rawVal];
    if (values.length > 50) return res.status(400).json(problem("filter_too_large", "Max 50 values"));

    const checked = values.map(v => def.schema.parse(v));               // throws → 400
    where.push(buildCondition(def.col, op, op === "in" ? checked : checked[0]));
  }

  // --- Sort: allowlist + mandatory unique tiebreaker ---
  const desc = sort.startsWith("-");
  const sortCol = desc ? sort.slice(1) : sort;
  if (!SORTABLE.includes(sortCol as any)) {
    return res.status(400).json(problem("unsortable_field",
      `Cannot sort by '${sortCol}'. Allowed: ${SORTABLE.join(", ")}`));
  }
  const dir = desc ? "DESC" : "ASC";

  // --- Cursor: bound to this exact query shape ---
  const queryFingerprint = fingerprint({ sort, filters: rest });
  if (cursor) {
    const { k } = decodeCursor(cursor, queryFingerprint);               // [sortValue, id]
    where.push(sql`(${raw(sortCol)}, id) ${raw(desc ? "<" : ">")} (${k[0]}, ${k[1]})`);
  }

  // --- Fetch limit+1 to determine has_more without a COUNT ---
  const rows = await db.query(sql`
    SELECT * FROM payments
     WHERE ${and(where)}
     ORDER BY ${raw(sortCol)} ${raw(dir)}, id ${raw(dir)}
     LIMIT ${limit + 1}
  `);

  const hasMore = rows.length > limit;
  const page = hasMore ? rows.slice(0, limit) : rows;
  const last = page[page.length - 1];

  res.json({
    object: "list",
    data: page.map(toDto),
    has_more: hasMore,
    next_cursor: hasMore && last
      ? encodeCursor({ k: [last[sortCol], last.id], q: queryFingerprint })
      : null,
    total_count: null,
  });
}
```

Read the comments — every one of them is a rule from this lesson. The required index:
```sql
CREATE INDEX idx_payments_merchant_created ON payments (merchant_id, created_at DESC, id DESC);
CREATE INDEX idx_payments_merchant_amount  ON payments (merchant_id, amount_minor DESC, id DESC);
CREATE INDEX idx_payments_merchant_status_created
    ON payments (merchant_id, status, created_at DESC, id DESC);   -- for the common filter+sort
```
**Notice: one index per sort/filter combination you expose.** That is the true cost of a flexible list endpoint, and it's why you allowlist rather than allowing anything.

---

## 10. Production rules

| Rule | Why |
|---|---|
| **Cursor pagination by default; offset only for small bounded sets** | O(1) vs O(offset), and no drift |
| **Always include a unique tiebreaker in `ORDER BY` and the cursor** | Non-unique sorts silently duplicate and skip rows |
| **Enforce a max `limit` (100 is standard) and a default (20)** | An unbounded `limit=1000000` is a DoS and an OOM |
| **Never compute an exact total on a large collection** | It's a second full scan on your hottest endpoint |
| **Allowlist filterable and sortable fields** | Injection defence *and* an index contract |
| **Reject unknown query parameters with 400** | A typo'd filter returns the wrong data set silently |
| **Tenant scope comes from the token, never from the cursor or a query param** | Otherwise a forged cursor/param is a cross-tenant breach |
| **Empty result = `200` with `[]`, never `404`** | The collection exists |
| **Return cursors; never document their structure** | Keeps the scheme changeable |
| **Cap `IN` list lengths and date ranges** | Unbounded query cost |
| **Expose `expand` with a depth cap, or accept the client's N+1** | Uncapped expansion is a self-inflicted DoS |
| **Add an index for every sort/filter you allow — verify with `EXPLAIN`** | An allowlisted field with no index is still a table scan |

---

## 11. Interview traps

**Q1. "Design `GET /payments` for a merchant with 4 million payments."**
The canonical question. Structure your answer: cursor pagination (with the O(1) vs O(offset) reason), the composite index with `merchant_id` first, `limit+1` for `has_more`, allowlisted filters and sorts, a unique tiebreaker, `total_count: null` with justification, and the tenancy scope coming from the token. Then volunteer the drift problem — most candidates never mention correctness, only performance, and mentioning it is what makes you memorable.

**Q2. "Why not offset?"**
Both reasons: **O(offset) scan cost**, and **drift** (duplicates on insert, skips on delete). Emphasise that drift is a *correctness* bug, which matters more than the latency.

**Q3. "How do you implement a `previous` page with cursors?"**
Reverse the comparison and the `ORDER BY`, fetch `limit+1`, then reverse the results back before returning them. The cursor payload needs to record its direction (or you issue separate `prev_cursor`/`next_cursor` values, which is cleaner).

**Q4. "The client wants 'jump to page 500'."**
Explain the trade-off honestly: cursors can't do random access. Options: (a) push back — nobody uses page 500; give them better filters and search instead; (b) hybrid — cursors for iteration, offset capped at page N (e.g. 100) for the UI; (c) if it's genuinely required, precompute page boundaries into a materialised table. **The recommended answer is (a) with (b) as the compromise**, because it names the real product need behind the request.

**Q5. "Client sends `?limit=1000000`. What happens?"**
Clamp to your documented max and — importantly — **return an error rather than silently clamping**, or at minimum document the clamp. Silent clamping means the client thinks it got everything. If you clamp, `has_more: true` must tell them the truth.

**Q6. "What's wrong with `?sort=amount` on a table where thousands of rows share an amount?"**
Non-deterministic order → cursor pagination duplicates and skips rows. Fix: always append `, id`. This question is a specific test of whether you've actually implemented cursor pagination or just read about it.

**Q7. "Could a client tamper with the cursor to see another merchant's data?"**
Only if you put the tenant in the cursor and trusted it. **Tenancy always comes from the authenticated token**, so the worst a forged cursor achieves is a wrong page of the caller's own data. Then add: *"I'd still validate the decoded cursor against the query fingerprint, so a cursor from a different filter doesn't produce silently wrong pages."*

**Q8. "How do you paginate a collection sorted by a mutable field?"**
Acknowledge the limitation: if `amount` changes mid-iteration, a row can move between pages. Options: sort only by immutable fields for exports, snapshot the query (a saved-search/materialised result), or accept it and document that iteration is not transactionally consistent. **For a complete export, the right answer is a dedicated async export job**, not pagination — it can run in one snapshot/transaction and produce a stable file.

**Q9. "Empty collection: 200 or 404?"**
`200` with `{"data": []}`. `404` means the *collection* doesn't exist. `GET /customers/cus_gone/payments` where the customer doesn't exist → `404`; where they exist with no payments → `200 []`.

**Q10. "Client needs each payment's customer email in a 20-row table."**
Without help, that's 1 + 20 requests — client-side N+1. Offer `?expand=customer` (with a depth cap), which turns it into one request and one server-side join. This is exactly the pain GraphQL was invented for, and saying so shows you understand *why* GraphQL exists rather than just what it is.

---

## 12. Build & break

### Build — prove the offset cliff with real data
```sql
-- Postgres. This is the most convincing 5 minutes in the whole track.
CREATE TABLE bench (id bigserial PRIMARY KEY, merchant_id int, created_at timestamptz, amount int);
INSERT INTO bench (merchant_id, created_at, amount)
SELECT 1, now() - (i || ' seconds')::interval, (random()*100000)::int
FROM generate_series(1, 3000000) AS s(i);
CREATE INDEX ON bench (merchant_id, created_at DESC, id DESC);
ANALYZE bench;

-- OFFSET: watch the cost grow
EXPLAIN ANALYZE SELECT * FROM bench WHERE merchant_id=1 ORDER BY created_at DESC, id DESC LIMIT 20 OFFSET 0;
EXPLAIN ANALYZE SELECT * FROM bench WHERE merchant_id=1 ORDER BY created_at DESC, id DESC LIMIT 20 OFFSET 100000;
EXPLAIN ANALYZE SELECT * FROM bench WHERE merchant_id=1 ORDER BY created_at DESC, id DESC LIMIT 20 OFFSET 2000000;

-- KEYSET: constant, regardless of depth
EXPLAIN ANALYZE SELECT * FROM bench
 WHERE merchant_id=1 AND (created_at, id) < ('2026-03-01 00:00:00+00', 1500000)
 ORDER BY created_at DESC, id DESC LIMIT 20;

-- And the count that "just shows a number"
EXPLAIN ANALYZE SELECT count(*) FROM bench WHERE merchant_id=1;
```
Write the four numbers in your notes. **Those numbers are your interview answer** — quoting a measurement you took yourself is worth more than any explanation.

### Build — reproduce drift
1. Seed 100 rows. Fetch page 1 with `OFFSET 0 LIMIT 20`, newest-first.
2. Insert 5 new rows.
3. Fetch page 2 with `OFFSET 20 LIMIT 20`.
4. Diff the two pages. Find the 5 duplicated IDs. Now delete 5 rows instead and find the 5 you never saw.
5. Redo it with a cursor. No duplicates, no gaps.

### Break — five failures
1. **Remove the tiebreaker.** Sort by a low-cardinality column (`status`) with cursor pagination and iterate all pages. Count distinct IDs vs total rows. They won't match.
2. **Remove the `limit` cap** and request `limit=500000`. Watch memory and latency. Now watch what `JSON.stringify` of that array does to your Node event loop (Lesson 02).
3. **Silently ignore unknown filters.** Query `?statuss=succeeded`, get all 3M rows back, and imagine that's a reconciliation script. Fix it to 400.
4. **Interpolate a sort column into SQL** in a scratch (never committed) branch, then pass `?sort=id; DROP TABLE bench;--`. See exactly why allowlisting isn't optional. Then delete the branch.
5. **Drop the index** and re-run your keyset query. Watch the seek become a scan. **The index is not an optimisation — it's part of the API contract.**

### Explain out loud (2 minutes)
1. Why offset fails, both ways, with the numbers you measured.
2. The keyset SQL, including the row-value comparison and why the tiebreaker is mandatory.
3. Why cursors are opaque, and why tenancy never lives in them.
4. What you'd tell a PM who wants an exact total count.

---

## What's next

You can read collections safely. Now the writes: the difference between `PUT` and `PATCH` in practice (including which patch format to pick), bulk operations and their nasty partial-failure semantics, and the concurrency problem — two clients updating the same resource, and how `ETag` + `If-Match` stops one of them silently erasing the other.

Next → **[Lesson 09: Writes — POST vs PUT vs PATCH, bulk & concurrency](09-writes-patch-and-bulk.md)**
