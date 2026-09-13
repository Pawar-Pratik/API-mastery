# Lesson 05 — JSON vs XML vs Protobuf vs the rest

> **Why this lesson exists:** you asked *"why JSON in the payload, why not XML, and what are their limitations?"* That question has a real, technical answer — not "JSON is modern." And JSON's limitations are not academic: **one of them silently corrupts money and IDs, and you will hit it in Ledger.** Serialization choice is also the cleanest "do you know the numbers?" interview probe in the whole API space.

**Time:** ~70 minutes · **Prereq:** Lesson 04

---

## 1. The idea in one sentence

> **Serialization format is a trade between human readability, size, speed, and schema enforcement — and you can only have three at a time.**

JSON picks readability + ubiquity and gives up size, speed and schema. Protobuf picks size, speed and schema and gives up readability. That's the whole design space, and every format is a point in it.

---

## 2. Why JSON won — the actual history, briefly

Because the reason is *not* "it's nicer," and knowing the real one makes you sound like you've been in the industry longer than you have.

**2000–2008: XML was the default.** SOAP, WSDL, XSD, WS-Security — an enormous, genuinely well-engineered enterprise stack. It had things JSON still lacks: a real schema language, namespaces, digital signatures, transformation (XSLT), and validation as a first-class step.

Three things killed it for web APIs:

1. **The browser.** JSON is *literally JavaScript object syntax*. In 2006, parsing JSON in a browser was `eval(response)` — free, instant, zero library. Parsing XML meant walking a DOM with `getElementsByTagName`, then hand-converting every string. When AJAX exploded, one format was 1 line and the other was 40. That asymmetry decided it.
2. **Verbosity.** The same data is typically 30–50% larger in XML, and in 2007 that mattered on 3G.
3. **Accidental complexity.** XML gives you two ways to express everything — attribute or child element — and no rule for choosing. `<payment amount="4999"/>` vs `<payment><amount>4999</amount></payment>`. Every team chose differently, so every integration needed a conversation. Add namespaces, and the "simple" case stopped being simple.

The deeper point, worth saying in an interview: **XML is a document markup language that was pressed into service as a data interchange format.** It's excellent at documents — mixed content, ordering, annotation, `<p>text <b>bold</b> more</p>`. Data interchange doesn't need any of that, and pays for all of it. JSON is a data format that can't describe documents at all, which is exactly why it's better at data.

> **XML is not dead, and saying so is a mistake.** It's load-bearing in banking (ISO 20022, SWIFT MX), healthcare (HL7 v2/v3, CDA), government filings, telco provisioning, SAML SSO (which is XML + XML-DSig and used by most enterprises today), Office formats (`.docx` is zipped XML), RSS, and SVG. If you interview at a bank or any enterprise integration role, "XML is legacy" is the wrong answer; "XML is the right tool where documents, signatures and formal schemas matter" is the right one.

---

## 3. JSON's five real limitations

This is the meat of the lesson. Anyone can say "JSON has no comments." These five cause outages.

### Limitation 1 — Numbers. **This is the one that will bite you.**

JSON's spec defines numbers as arbitrary-precision decimal *text*. But **JavaScript parses every JSON number into an IEEE-754 double**, which has 53 bits of integer precision. So:

```js
JSON.parse('{"id": 12345678901234567890}').id   // → 12345678901234567000   ← WRONG
JSON.parse('{"id": 9007199254740993}').id       // → 9007199254740992       ← off by one
Number.MAX_SAFE_INTEGER                          // → 9007199254740991
```

And floats:
```js
JSON.parse('{"total": 0.1}').total + JSON.parse('{"total": 0.2}').total  // 0.30000000000000004
19.99 * 100                                                              // 1998.9999999999998
Math.round(1998.9999999999998) / 100                                     // 19.99  ← survived by luck
0.615.toFixed(2)                                                         // "0.61" ← not 0.62
```

**Three consequences, all of which have caused real production incidents:**

| Problem | Real-world manifestation | Fix |
|---|---|---|
| **Large integers lose precision** | Twitter's numeric tweet IDs broke every JS client; they added `id_str` alongside `id` and told everyone to use it. Snowflake IDs, BigInt DB keys, and Java `long`s all hit this | **Send large IDs as strings.** Always |
| **Money in floats is wrong** | `0.1 + 0.2 !== 0.3`. Sum 10,000 invoice lines and your total is cents off, forever, in a financial report | **Integer minor units** (`4999` = $49.99), or a decimal string (`"49.99"`) parsed by a decimal library |
| **No decimal type exists** | You cannot represent `0.1` exactly in binary floating point. Ever | Same as above |

**This is why Stripe's `amount` is an integer number of cents** — the answer to the homework from Lesson 01. It's not a style choice; it's the only representation that's exactly correct in every language that will ever parse it.

> **The rule for Ledger, and for your career:** money is `{ amount_minor: 4999, currency: "usd" }`. Never a float, never a decimal number in JSON. IDs are strings. If you take one thing from this entire lesson, take this — it's also a *superb* interview answer because most candidates have never thought about it.

For genuinely huge numbers you can't restructure, JS has `JSON.parse(text, reviver)` with `BigInt`, or libraries like `json-bigint`. But restructuring is better: **make the wire format string-based and the ambiguity disappears.**

### Limitation 2 — No date/time type

JSON has strings, numbers, booleans, null, objects, arrays. **That's it.** No dates, no UUIDs, no binary, no decimals, no sets, no regex.

So dates are a convention, and conventions get violated:

| Format | Example | Verdict |
|---|---|---|
| **ISO 8601 / RFC 3339 UTC** | `"2026-03-14T09:30:00Z"` | ✅ **Use this.** Unambiguous, sortable as a string, human-readable, universally parseable |
| Unix seconds | `1773480600` | ✅ Acceptable (Stripe uses it). Compact, no timezone confusion — but unreadable in logs, and ambiguous s vs ms |
| Unix milliseconds | `1773480600000` | ⚠️ Fine if documented. The s/ms ambiguity is a real, frequent bug |
| **Local time with no offset** | `"2026-03-14T09:30:00"` | ❌ **Never.** Which 9:30? Whose 9:30? |
| Locale strings | `"14/03/2026"` | ❌ Never. `03/14` vs `14/03` has caused financial errors |
| .NET legacy | `"/Date(1773480600000)/"` | ❌ Historical horror |

Rules: **always UTC on the wire with an explicit `Z`**, always ISO 8601, name fields with a `_at` suffix (`created_at`, `expires_at`), and separately document whether ranges are inclusive or exclusive — the schema can't say it and *someone will assume the other one*.

> Note the sneaky one: `new Date("2026-03-14")` in JS is parsed as **UTC midnight**, but `new Date("2026-03-14T00:00:00")` is parsed as **local midnight**. Same-looking strings, up to a day apart. Date-only values should be a separate documented type (`"date"` vs `"date-time"` in OpenAPI).

### Limitation 3 — No binary type

Binary must be base64'd into a string: **+33% size**, CPU on both ends, no streaming (must be fully buffered), and it clutters logs. See [Lesson 04 §6](04-headers-and-payloads.md) — use `multipart` or pre-signed URLs instead.

### Limitation 4 — No schema, natively

JSON alone tells you nothing about what's required, what type a field is, or what values are legal. You bolt on:
- **JSON Schema** — the standard for validation (what OpenAPI uses)
- **OpenAPI** — schema + endpoints + everything else ([Lesson 25](../06-craft/25-openapi-contract-first.md))
- **Runtime validators** — Zod, Valibot, io-ts in TS; Bean Validation in Java

Contrast XSD, which is part of the XML ecosystem by design, and Protobuf, where the schema *is* the format. This is JSON's most legitimate weakness, and the one Protobuf/Avro exist to solve.

### Limitation 5 — Ambiguities the spec permits

| Issue | Detail |
|---|---|
| **Duplicate keys** | `{"a":1,"a":2}` is *valid* JSON. Parsers disagree: JS takes the last, some libraries take the first, some error. **This is a genuine attack vector** — a validating proxy sees `{"role":"user","role":"admin"}` as `user` while the backend sees `admin` |
| **Key order** | Objects are unordered by spec. Most parsers preserve insertion order, but you must not rely on it (and Hyrum's Law says someone will) |
| **No comments** | Deliberate — Crockford removed them to stop people putting parsing directives in them. Painful for config; irrelevant for APIs |
| **No trailing commas** | Endless hand-written-JSON friction. Not an API concern |
| **`NaN` / `Infinity` are illegal** | `JSON.stringify({x: NaN})` silently emits `{"x":null}`. Your invalid number becomes a valid null and travels downstream |
| **`undefined` disappears** | `JSON.stringify({a: undefined, b: 1})` → `{"b":1}`. The key is *gone*, not null — so "field absent" and "field explicitly null" mean different things, which matters enormously for `PATCH` ([Lesson 09](../02-rest-design/09-writes-patch-and-bulk.md)) |
| **UTF-8 only, but lone surrogates possible** | `JSON.stringify` can emit unpaired surrogates that aren't valid UTF-8; some strict parsers reject them |

> **The `undefined` vs `null` distinction is a real design decision, not trivia.** In a `PATCH` body, `{"phone": null}` means "delete the phone" and `{}` means "don't touch the phone." If your validator or ORM collapses those, users lose data. Test it explicitly.

---

## 4. The alternatives, with numbers

Same object — a payment with 8 fields — serialized every way:

| Format | Bytes | Relative | Human-readable | Schema | Parse speed (rel.) |
|---|---|---|---|---|---|
| **JSON** | 220 | 1.0× | ✅ | bolt-on | 1.0× |
| **JSON, gzipped** | 165 | 0.75× | ❌ | bolt-on | 0.9× (+CPU) |
| **XML** | 340 | 1.55× | ✅ | ✅ XSD | ~0.4× (slower) |
| **MessagePack** | 150 | 0.68× | ❌ | bolt-on | ~2× |
| **CBOR** | 148 | 0.67× | ❌ | CDDL | ~2× |
| **Protobuf** | 62 | **0.28×** | ❌ | ✅ **required** | ~3–5× |
| **Avro** | 58 | 0.26× | ❌ | ✅ required | ~3× |
| **CSV** (tabular only) | 48 | 0.22× | ✅ | ❌ | very fast |

Treat these as orders of magnitude, not gospel — actual ratios depend heavily on your data (long field names favour Protobuf more; deeply nested data favours it less; repetitive data favours gzip more). **The honest headline: Protobuf is roughly 3–5× smaller and 3–5× faster to parse than JSON. Gzipped JSON closes much of the size gap but not the CPU gap** — in fact it adds CPU.

### Protobuf (Protocol Buffers)

```proto
syntax = "proto3";
message Payment {
  string id           = 1;   // ← the field NUMBER is the wire identity, not the name
  int64  amount_minor = 2;
  string currency     = 3;
  Status status        = 4;
  enum Status { STATUS_UNSPECIFIED = 0; PENDING = 1; SUCCEEDED = 2; }
}
```

The mechanism: fields are encoded as `(field_number, wire_type, value)` — **names never travel**. That's where the size win comes from, and it's also why renaming a field is free while changing a number is catastrophic.

| Strength | Weakness |
|---|---|
| Smallest and fastest of the mainstream options | Not human-readable — you cannot `curl` and eyeball it |
| Schema is mandatory → generated types in 10+ languages | Requires codegen in your build, and a schema registry at scale |
| Strong, well-defined evolution rules | Browsers need grpc-web + a proxy ([Lesson 22](../05-beyond-rest/22-grpc-and-protobuf.md)) |
| Field numbers make renames free | Field numbers are permanent; reusing one silently corrupts data |

**Use Protobuf when:** internal service-to-service, high volume, polyglot teams, streaming. **Don't when:** it's a public API for third parties (the DX cost is real), or debuggability matters more than 3× CPU.

### MessagePack / CBOR
"Binary JSON" — same data model, compact encoding, no schema requirement. CBOR is an IETF standard (RFC 8949) and is what WebAuthn/passkeys use. Good middle ground when you want size/speed without adopting codegen — but you lose readability *and* still have no schema, which is often the worst of both.

### Avro
Schema-first like Protobuf, but the schema travels with the data (or lives in a registry) and it's row-oriented for big batches. **Dominant in the Kafka/Hadoop world** — if you interview anywhere data-platform-adjacent, know it exists and that Confluent Schema Registry is how schemas are governed.

### CSV
Genuinely the right answer for one job: **bulk tabular export**. 4 million payments as JSON is a 3GB array your client cannot stream sanely; as CSV it's 400MB and opens in Excel. Every serious API has a CSV export path. Its problems (no types, no nesting, quoting hell, Excel mangling long numbers and dates) are why it's *only* for export.

### YAML / TOML
Configuration formats, **not** wire formats. YAML in particular has a genuinely dangerous parser surface (arbitrary object construction in unsafe loaders) and the famous "Norway problem" (`no` → `false`, so country code NO becomes boolean). Never accept YAML from an untrusted source. Never use it as an API payload.

---

## 5. The decision table

```
Public web API, third-party developers?        → JSON.        Non-negotiable. DX wins
Browser client?                                → JSON
Internal service-to-service, high volume?      → Protobuf/gRPC
Kafka / event streaming / data platform?       → Avro (+ schema registry) or Protobuf
Bulk export of tabular data?                   → CSV (streamed) or Parquet for analytics
Banking / healthcare / gov integration?        → XML — because the counterparty says so
Need digital signatures on the document itself? → XML (XML-DSig) or JWS over JSON
IoT / constrained device / tiny packets?       → CBOR or MessagePack
Config files?                                  → YAML/TOML — and never on the wire
```

**And the meta-answer for interviews:** *"JSON at the edge, Protobuf inside."* That's what Google, Netflix and Uber all actually do, and the reason is that the two ends have different constraints: the edge optimises for developer adoption and debuggability; the interior optimises for CPU and bytes at millions of requests per second.

---

## 6. Production rules

| Rule | Why |
|---|---|
| **Money = integer minor units + explicit currency** | Floats are wrong. Not "risky" — wrong |
| **Large IDs = strings** | 53-bit float precision. Twitter learned this publicly; you don't have to |
| **Timestamps = ISO 8601 UTC with `Z`, field named `*_at`** | Only unambiguous option; sortable as text |
| **Booleans stay booleans** | `"true"`, `1`, `"Y"` are three different bugs. Never invent a fourth |
| **Enums = lowercase snake_case strings, never integers** | `"succeeded"` is self-documenting in a log; `3` requires a lookup table and breaks if you reorder |
| **`snake_case` or `camelCase` — pick one and never mix** | Mixing is the loudest possible signal of an unreviewed API. (`snake_case` for public JSON is the most common convention: Stripe, GitHub, Twilio. `camelCase` if your consumers are JS-only) |
| **Top-level response must be an object, never a bare array** | `[1,2,3]` leaves you nowhere to add `pagination` or `meta` later without a breaking change. Also: bare-array JSON responses were historically exploitable via JSON hijacking |
| **Never trust client-declared types; validate and coerce at the boundary** | `"amount": "4999"` from a form-encoded client. Parse, don't assume ([TS L15](../../TypeScript/README.md)) |
| **Reject duplicate JSON keys, or at least be aware of them** | Real smuggling vector past validating proxies |
| **Set a max payload size and a max nesting depth** | Deeply nested JSON is a parser DoS (billion-laughs' JSON cousin) |
| **Never `eval()` JSON** | Historical; still appears in old code |
| **Don't compress already-compressed content** | Pure CPU waste |

### Naming: the table you can point at in a review

| Concept | Do | Don't |
|---|---|---|
| Money | `amount_minor: 4999`, `currency: "usd"` | `amount: 49.99` |
| Large ID | `id: "pi_3Nx8"` or `"12345678901234567890"` | `id: 12345678901234567890` |
| Time | `created_at: "2026-03-14T09:30:00Z"` | `created: "14/03/2026"` |
| Duration | `timeout_seconds: 30` (unit in the name!) | `timeout: 30` |
| Boolean | `is_active`, `has_dispute` | `active: "yes"` |
| Enum | `status: "requires_capture"` | `status: 2` |
| Absence | omit the key, or `null` — **and document which** | `""`, `-1`, `0`, `"N/A"` |
| Collection | `{ "data": [...], "has_more": true }` | `[...]` |

---

## 7. Interview traps

**Q1. "Why JSON and not XML?"**
The strong answer is historical + technical, not aesthetic:
> *"JSON is the native data model of the browser, so parsing was free when AJAX took off — that's what actually decided it. It's also 30–50% smaller and has one obvious way to express a value, where XML gives you attributes and elements with no rule for choosing. But XML is a document format that was repurposed for data: it's genuinely better where you need namespaces, formal schemas, signatures or transformations, which is why banking, healthcare and SAML still run on it."*

**Q2. "What are JSON's limitations?"**
Number precision (the big one — money and large IDs), no date type, no binary, no native schema, and spec ambiguities like duplicate keys and `undefined`-vs-absent. **Lead with numbers and give the Twitter `id_str` example** — that single anecdote makes the point better than the theory.

**Q3. "How do you represent money in an API?"**
`{ "amount_minor": 4999, "currency": "usd" }`. Integer minor units plus ISO 4217 currency. Then handle the follow-ups they *will* ask:
- *"What about currencies with no minor unit?"* JPY has 0 decimals, so 4999 JPY = ¥4,999. **You need the currency to interpret the integer** — which is exactly why the two fields are inseparable.
- *"What about 3-decimal currencies?"* BHD, KWD, TND have 3. Your `minor_unit_exponent` comes from the currency table, not an assumption.
- *"What about crypto or fractional cents?"* Then minor units aren't enough; use a decimal string plus an explicit scale, and a decimal library server-side.

**Q4. "`Number.MAX_SAFE_INTEGER` — why does it matter?"**
2^53−1 = 9007199254740991. Beyond it, JS integers silently lose precision, so any 64-bit ID from a database becomes a *different* ID after a round trip. It's silent, and it corrupts.

**Q5. "Is Protobuf always better than JSON?"**
No, and saying yes fails the question. It's smaller and faster, but you lose human readability, `curl`-ability, and easy third-party adoption, and you take on codegen and (at scale) a schema registry. *"JSON at the edge, Protobuf internally"* is the answer, with reasons.

**Q6. "Should the top-level response be an array?"**
No. `{"data": [...], "has_more": true, "next_cursor": "..."}` leaves room to add metadata without breaking clients — and pagination *will* need it. Bonus: JSON hijacking history.

**Q7. "How do you handle dates across timezones?"**
Store UTC, transmit UTC (ISO 8601 with `Z`), render in the user's timezone in the UI. Store the user's IANA timezone name (`Asia/Kolkata`), **not a UTC offset**, because offsets change with DST — this is the subtlety most candidates miss. And for future recurring events (a 9am daily report), store the *local* time plus the zone, because if you store UTC the meeting moves when DST shifts.

**Q8. "Duplicate keys in JSON — valid?"**
Yes, valid per spec, behaviour undefined in practice. Then land the security point: a validating gateway and the backend can disagree about which value counts, so it's a real authorization-bypass vector.

**Q9. "gzip or Brotli?"**
Brotli compresses ~15–20% better for text at comparable decode speed, and is universally supported in modern browsers — so `br` first, `gzip` as fallback. Pre-compress static assets at build time; compress dynamic responses at level 4–5 (not 11, which is for static content and burns CPU for a few percent).

---

## 8. Build & break

### Build — a wire-safe money type
Create `scratch/money.ts`. This is the code you'll reuse in Ledger.

```ts
/** ISO 4217 exponent: how many decimal places the currency has. */
const MINOR_UNIT_EXPONENT: Record<string, number> = {
  usd: 2, eur: 2, gbp: 2, inr: 2,
  jpy: 0, krw: 0,                     // no minor unit
  bhd: 3, kwd: 3, tnd: 3,             // three
};

export type Money = {
  /** Integer amount in the currency's minor unit. 4999 usd = $49.99 */
  readonly amountMinor: number;
  /** Lowercase ISO 4217 */
  readonly currency: string;
};

export function money(amountMinor: number, currency: string): Money {
  if (!Number.isInteger(amountMinor)) throw new Error("amountMinor must be an integer");
  if (!Number.isSafeInteger(amountMinor)) throw new Error("amountMinor exceeds safe integer range");
  const c = currency.toLowerCase();
  if (!(c in MINOR_UNIT_EXPONENT)) throw new Error(`unsupported currency: ${currency}`);
  return { amountMinor, currency: c };
}

/** Arithmetic stays in integers — never convert to a float to add. */
export function add(a: Money, b: Money): Money {
  if (a.currency !== b.currency) throw new Error("currency mismatch");   // silent conversion = fraud
  return money(a.amountMinor + b.amountMinor, a.currency);
}

/** Display only. Never feed this back into arithmetic. */
export function format(m: Money, locale = "en-US"): string {
  const exp = MINOR_UNIT_EXPONENT[m.currency];
  return new Intl.NumberFormat(locale, {
    style: "currency", currency: m.currency.toUpperCase(),
    minimumFractionDigits: exp, maximumFractionDigits: exp,
  }).format(m.amountMinor / 10 ** exp);
}

/** Splitting a bill: distribute remainder, never lose a cent. */
export function split(m: Money, ways: number): Money[] {
  const base = Math.floor(m.amountMinor / ways);
  let remainder = m.amountMinor - base * ways;
  return Array.from({ length: ways }, () => {
    const extra = remainder > 0 ? 1 : 0;
    remainder -= extra;
    return money(base + extra, m.currency);
  });
}
```

Then prove it: `split(money(1000, "usd"), 3)` must give `[334, 333, 333]` and sum to exactly 1000. Do the same with floats and watch a cent evaporate. **That cent is the lesson.**

### Break — six experiments in `node -e`
```bash
# 1. Precision loss on a large ID
node -e "console.log(JSON.parse('{\"id\":12345678901234567890}').id)"

# 2. Float money
node -e "console.log(0.1+0.2, 19.99*100, (0.615).toFixed(2))"

# 3. NaN silently becomes null
node -e "console.log(JSON.stringify({x: NaN, y: Infinity, z: undefined}))"

# 4. Duplicate keys — which wins?
node -e "console.log(JSON.parse('{\"role\":\"user\",\"role\":\"admin\"}'))"

# 5. Date parsing is not what you think
node -e "console.log(new Date('2026-03-14').toISOString(), new Date('2026-03-14T00:00:00').toISOString())"

# 6. Size comparison, for real
node -e "const o={id:'pi_3Nx8',amount_minor:4999,currency:'usd',status:'succeeded',created_at:'2026-03-14T09:30:00Z',customer:'cus_9s2k',captured:true,metadata:{order:'A17'}};const j=JSON.stringify(o);const z=require('zlib').gzipSync(j).length;console.log('json',j.length,'gzip',z)"
```

### Break your own API — three data-format bugs
1. Store a price as a JS float, sum 10,000 of them, compare to the integer sum. Note the discrepancy. That's a financial report that doesn't reconcile.
2. Return a 19-digit ID as a JSON number, fetch it in the browser, `GET` it back by that ID → `404`. The ID changed in transit.
3. Return `created_at: "2026-03-14 09:30:00"` (no `Z`). Parse it in two timezones. Note you get two different instants from identical bytes.

### Explain out loud (60 seconds)
1. Why JSON beat XML — the browser reason first.
2. JSON's five limitations, leading with numbers.
3. How you represent money, and the JPY follow-up.
4. When you'd choose Protobuf, and what you'd give up.

---

## Module 1 complete — checkpoint

Before moving on, verify you can do all of these **without looking**:

- [ ] Define an API in contract terms, and name the four layers
- [ ] Narrate the twelve stages of a request, with a cost and failure mode each
- [ ] Explain which stages a warm connection skips, and why call #2 is faster
- [ ] Explain CORS: what it protects, who enforces it, what triggers a preflight
- [ ] State safe vs idempotent vs cacheable, with a method that is one but not another
- [ ] Choose between 400/401/403/404/409/412/422/429/500/503 without hesitating
- [ ] Explain `PUT` vs `PATCH` vs `POST` by URI ownership and totality
- [ ] Name the four parts of an HTTP message and what the blank line is for
- [ ] Decide cookie vs Bearer, naming the attack each defends against
- [ ] Represent money and large IDs correctly, and say why

Any unticked box: re-read that section **now**. Module 2 builds directly on all ten.

---

## What's next

You now know the machinery. Module 2 is where you're *judged*: taking a requirement and producing a design. It starts with the thing everyone claims to know and few can define — REST itself, its six constraints, and what each one actually buys you.

Next → **[Lesson 06: REST, actually — constraints & maturity](../02-rest-design/06-rest-constraints.md)**
