# Lesson 22 — gRPC & Protobuf

> **Why this lesson exists:** gRPC is what large companies actually use *inside* their systems, so if you interview anywhere with more than a handful of services you'll be asked about it. It's also the cleanest possible demonstration of a real engineering trade-off: you give up human readability and browser support, and get 5× smaller payloads, mandatory schemas, generated clients in ten languages, and streaming that HTTP/1.1 can't do.

**Time:** ~80 minutes · **Prereq:** Lessons 02, 05

---

## 1. The idea in one sentence

> **gRPC is typed function calls over HTTP/2 with a mandatory schema — it optimises for machine-to-machine efficiency and contract safety, at the cost of everything that makes an API pleasant to explore by hand.**

---

## 2. Why it exists: the internal-traffic problem

At any real scale, most of your traffic is service-to-service, and REST+JSON starts costing measurably:

| Cost | Detail |
|---|---|
| **Bytes** | JSON re-sends every field *name* on every message. `{"amount_minor":4999}` — the key is 12 of the 20 bytes |
| **CPU** | JSON parsing is text parsing. At 500k messages/sec it's a real fraction of your compute bill |
| **No contract** | Every consumer hand-writes types and hopes; a renamed field is discovered in production |
| **No streaming** | HTTP/1.1 gives you request/response. Long-lived bidirectional flows need a different mechanism |
| **Boilerplate** | Every service writes its own client, retry logic, deadline handling and error mapping |

Google's internal answer (Stubby, then gRPC in 2015) attacks all five: a schema-first IDL, a compact binary encoding, HTTP/2 multiplexing and streaming, and generated clients that come with deadlines, retries and interceptors built in.

The industry outcome, worth stating plainly: **REST/JSON at the edge, gRPC inside.** Netflix, Uber, Square, Dropbox, and essentially the whole Kubernetes ecosystem (`etcd`, the CRI, CSI and CNI interfaces) run gRPC internally.

---

## 3. Protobuf: the schema is the contract

```proto
syntax = "proto3";
package ledger.v1;

import "google/protobuf/timestamp.proto";

service PaymentService {
  rpc CreatePayment (CreatePaymentRequest) returns (Payment);
  rpc GetPayment    (GetPaymentRequest)    returns (Payment);
  rpc ListPayments  (ListPaymentsRequest)  returns (ListPaymentsResponse);
  rpc WatchPayments (WatchPaymentsRequest) returns (stream Payment);   // server stream
}

message Payment {
  string id                              = 1;
  string merchant_id                     = 2;
  int64  amount_minor                    = 3;   // int64, not double. Money is integers
  string currency                        = 4;
  PaymentStatus status                   = 5;
  optional string customer_id            = 6;   // proto3 `optional` → presence is tracked
  google.protobuf.Timestamp created_at   = 7;
  map<string, string> metadata           = 8;
}

enum PaymentStatus {
  PAYMENT_STATUS_UNSPECIFIED = 0;   // ★ field 0 MUST exist and mean "unset"
  PAYMENT_STATUS_PENDING     = 1;
  PAYMENT_STATUS_SUCCEEDED   = 2;
  PAYMENT_STATUS_FAILED      = 3;
}

message CreatePaymentRequest {
  int64  amount_minor   = 1;
  string currency       = 2;
  optional string customer_id = 3;
  string idempotency_key = 4;      // Lesson 18 — still your job, gRPC doesn't provide it
}
```

### The wire format, and why it's small

A message is encoded as a sequence of `(field_number, wire_type, value)` triples. **Field *names* never travel.**

```
Field 3 (amount_minor), varint, 4999  →  0x18 0x87 0x27      (3 bytes)
JSON equivalent: {"amount_minor":4999}                        (21 bytes)
```

Two mechanisms produce the size win:
- **Numbers instead of names.** Field 1 costs one byte of tag.
- **Varint encoding.** Small integers take one byte; only large ones take more.

This is why **field numbers are the real contract, and names are cosmetic.**

### The evolution rules — the most important table in this lesson

| Change | Safe? | Why |
|---|---|---|
| **Add a new field with a new number** | ✅ | Old readers skip unknown fields (a tolerant reader, by design) |
| **Rename a field** (same number) | ✅ **on the wire** | The name never travels. But it breaks *generated code* — a source change, not a wire change |
| **Remove a field** | ⚠️ | Only with `reserved`; see below |
| **Change a field's number** | ❌ **Catastrophic** | The old number now means nothing, and the new number may collide with another field's history |
| **Change a field's type** | ❌ mostly | Some pairs are compatible (`int32`↔`int64`, `bytes`↔`string` for UTF-8 data); most silently corrupt data |
| **Add an enum value** | ⚠️ | Old clients see it as the unknown value — which is exactly why `UNSPECIFIED = 0` exists |
| **Change `optional` ↔ `repeated`** | ❌ | Different wire semantics |
| **Reuse a deleted field's number** | ❌ **Data corruption** | An old client sends field 6 meaning `customer_id`; you now read it as `refund_reason` |

```proto
message Payment {
  reserved 6, 9 to 11;                    // ← numbers permanently retired
  reserved "customer_id", "old_status";   // ← names too, so nobody re-adds them
}
```

> **`reserved` is the single most important Protobuf habit.** Without it, someone will eventually reuse field 6, and an old client's `customer_id` will be parsed as whatever field 6 means now — **silently, with no error, corrupting data.** This is the Protobuf equivalent of "never repurpose a field" from [Lesson 11](../02-rest-design/11-versioning-and-evolution.md), and it's a favourite interview question.

### The proto3 presence problem
In proto3, scalar fields historically had no way to distinguish "set to 0" from "not set" — both encode as absent. That's the same `undefined`-vs-`null` problem as JSON PATCH ([Lesson 09](../02-rest-design/09-writes-patch-and-bulk.md)), and it matters enormously for partial updates.

Fixes, in order of preference: `optional` (restored in proto 3.15+, which tracks presence properly), wrapper types (`google.protobuf.Int64Value`), or a `google.protobuf.FieldMask` listing which fields the client intends to change — the gRPC-standard way to express PATCH semantics.

---

## 4. The four call types

This is gRPC's other big differentiator, and it's what HTTP/2 buys you.

```proto
rpc GetPayment    (GetPaymentRequest)         returns (Payment);                 // 1. unary
rpc WatchPayments (WatchRequest)              returns (stream Payment);          // 2. server stream
rpc UploadEvents  (stream Event)              returns (UploadSummary);           // 3. client stream
rpc SyncPayments  (stream PaymentSyncRequest) returns (stream PaymentSyncEvent); // 4. bidirectional
```

| Type | Shape | Real use |
|---|---|---|
| **Unary** | One in, one out | 95% of calls. The REST equivalent |
| **Server streaming** | One in, many out | Live payment feed, log tailing, large result sets without pagination |
| **Client streaming** | Many in, one out | Bulk ingest, telemetry upload, file chunks |
| **Bidirectional** | Many both ways, independently | Chat, live collaboration, long-lived sync sessions |

**Server streaming is genuinely valuable** and often overlooked: instead of paginating 4 million rows, you stream them, and the client processes as they arrive with bounded memory on both sides. It's flow-controlled by HTTP/2, so a slow consumer naturally applies backpressure — which a REST paginated loop cannot express.

---

## 5. gRPC over HTTP/2 — the parts you must know

```
POST /ledger.v1.PaymentService/GetPayment HTTP/2
content-type: application/grpc+proto
grpc-timeout: 5S                     ← the deadline, on the wire
te: trailers

[compressed-flag:1][length:4][protobuf bytes]

# Response
HTTP/2 200
content-type: application/grpc+proto
[frame]
--- trailers ---                      ← the STATUS arrives in TRAILERS, after the body
grpc-status: 0
grpc-message:
```

Three consequences of that layout:

1. **The status is in HTTP trailers**, not the status line — because a streaming response can only know its final status after all the data. This is precisely why **browsers cannot speak gRPC natively**: the Fetch API gives JS no access to trailers, and no way to control HTTP/2 framing.
2. **Deadlines are first-class.** `grpc-timeout` travels with the call, and gRPC propagates the *remaining* deadline to downstream calls automatically. That's the deadline propagation you had to hand-roll in [Lesson 18](../04-production/18-reliability-and-idempotency.md) — here it's built in, and it's one of gRPC's genuinely best features.
3. **One connection, many streams.** HTTP/2 multiplexing means no per-request connection setup and no head-of-line blocking at the HTTP layer.

### Status codes
gRPC has its own 17-code set. Know the mapping:

| gRPC | Code | HTTP analogue |
|---|---|---|
| `OK` | 0 | 200 |
| `INVALID_ARGUMENT` | 3 | 400 |
| `DEADLINE_EXCEEDED` | 4 | 504 |
| `NOT_FOUND` | 5 | 404 |
| `ALREADY_EXISTS` | 6 | 409 |
| `PERMISSION_DENIED` | 7 | 403 |
| `RESOURCE_EXHAUSTED` | 8 | 429 |
| `FAILED_PRECONDITION` | 9 | 400/409 |
| `ABORTED` | 10 | 409 |
| `UNIMPLEMENTED` | 12 | 501 |
| `INTERNAL` | 13 | 500 |
| `UNAVAILABLE` | 14 | 503 |
| `UNAUTHENTICATED` | 16 | 401 |

The distinction worth knowing, because it's about retry safety: **`FAILED_PRECONDITION` means don't retry until the system state changes; `ABORTED` means a concurrency conflict, so retry at a higher level; `UNAVAILABLE` means retry with backoff.** That's more retry-relevant information than HTTP status codes convey, and it's a nice detail to raise.

---

## 6. The browser problem

**You cannot call gRPC from browser JavaScript.** Trailers and framing control aren't available. Three workarounds:

| Option | How | Trade-off |
|---|---|---|
| **grpc-web** | A proxy (Envoy) translates gRPC-Web ↔ gRPC | **No client or bidirectional streaming.** Extra infrastructure hop |
| **Connect (connectrpc.com)** | A protocol that speaks gRPC, gRPC-Web *and* a plain HTTP/JSON mode from the same handlers | Best current option for "I want gRPC internally and something curl-able at the edge" |
| **A REST/GraphQL gateway** | Hand-written, or generated via `grpc-gateway` annotations | The most common real architecture |

`grpc-gateway` deserves a mention because it's genuinely elegant: annotate your proto and get a REST façade generated from the same contract.

```proto
rpc GetPayment (GetPaymentRequest) returns (Payment) {
  option (google.api.http) = { get: "/v1/payments/{id}" };
}
```
**One schema, two protocols** — REST/JSON for external consumers, gRPC internally. That's the architecture to describe when asked "how do you serve both?"

---

## 7. gRPC in practice

```ts
// Server
import { Server, ServerCredentials, status } from "@grpc/grpc-js";

const paymentService = {
  async getPayment(call, callback) {
    // Deadline handling: don't start work you can't finish.
    if (call.getDeadline() < Date.now() + 50) {
      return callback({ code: status.DEADLINE_EXCEEDED, message: "insufficient time budget" });
    }
    // Tenancy from metadata (the credential), never from the request message.
    const merchantId = await authenticate(call.metadata.get("authorization")[0]);

    const payment = await repos(merchantId).payments.findById(call.request.id);
    if (!payment) return callback({ code: status.NOT_FOUND, message: "payment not found" });

    callback(null, toProto(payment));
  },

  // Server streaming: flow control is handled by HTTP/2 — respect backpressure.
  async watchPayments(call) {
    const cursor = repos(await authenticate(call.metadata.get("authorization")[0]))
      .payments.streamSince(call.request.since);
    for await (const p of cursor) {
      if (call.cancelled) break;                 // the client hung up; stop working
      const ok = call.write(toProto(p));
      if (!ok) await once(call, "drain");        // ← the backpressure line people forget
    }
    call.end();
  },
};
```

```ts
// Client — note what comes free: deadlines, retries, connection pooling
const client = new PaymentServiceClient("payments.internal:50051", credentials, {
  "grpc.service_config": JSON.stringify({
    methodConfig: [{
      name: [{ service: "ledger.v1.PaymentService" }],
      timeout: "5s",
      retryPolicy: {
        maxAttempts: 3,
        initialBackoff: "0.2s", maxBackoff: "4s", backoffMultiplier: 2,
        retryableStatusCodes: ["UNAVAILABLE", "DEADLINE_EXCEEDED"],   // ← only safe ones
      },
    }],
  }),
  "grpc.keepalive_time_ms": 30_000,
});
```

**That `service_config` is a highlight worth naming in an interview:** retry policy, backoff, deadlines and hedging are **declarative configuration**, not code you write per call site. Combined with a service mesh, you get mTLS, load balancing, circuit breaking and retries without touching application code — which is the operational argument for gRPC as much as the performance one.

> Note the retry policy lists only `UNAVAILABLE` and `DEADLINE_EXCEEDED`. gRPC won't retry `INVALID_ARGUMENT` for you, and **it can't know whether your `CreatePayment` is idempotent** — so an idempotency key is still your responsibility ([Lesson 18](../04-production/18-reliability-and-idempotency.md)). gRPC gives you the retry machinery; it does not give you retry safety.

---

## 8. gRPC vs REST — the honest comparison

| | REST + JSON | gRPC + Protobuf |
|---|---|---|
| Payload size | 1× | **~0.3×** |
| Parse speed | 1× | **~3–5×** |
| Schema | Optional (OpenAPI, bolted on) | **Mandatory** |
| Generated clients | Possible | **Excellent, 10+ languages** |
| Browser support | **Native** | Needs a proxy |
| Human-readable / `curl`-able | **Yes** | No (needs `grpcurl`) |
| Streaming | SSE/WebSocket (bolted on) | **Native, four modes** |
| HTTP caching | **Yes, free** | No |
| Deadlines/retries | Roll your own | **Built in, declarative** |
| Load balancing | Easy (L7, per request) | Harder — long-lived connections pin to one backend, so you need **client-side LB or a mesh** |
| Debugging | Easy | Harder; needs tooling |
| Learning curve | Low | Moderate (codegen, build integration) |
| Best for | Public APIs, browsers, third parties | Internal services, polyglot teams, high volume, streaming |

That load-balancing row is a real operational gotcha worth raising: because gRPC multiplexes many requests over **one long-lived TCP connection**, a naïve L4 load balancer sends all of a client's traffic to a single backend, so adding replicas doesn't help. The fixes are client-side load balancing (via DNS or a resolver), a service mesh sidecar (Envoy/Linkerd), or an L7 proxy that understands HTTP/2 streams. Knowing this marks you as someone who has actually deployed it.

---

## 9. Production rules

| Rule | Why |
|---|---|
| **Never change or reuse a field number; always `reserved` deletions** | Reused numbers silently corrupt old clients' data |
| **Every enum has an `UNSPECIFIED = 0`** | Distinguishes unset from a real value; makes new values safe |
| **`int64` for money, never `double`** | Same reason as JSON (Lesson 05) — plus note `int64` becomes a **string** in proto3 JSON mapping, precisely because of JS's 53-bit limit |
| **Use `optional` (or `FieldMask`) for partial updates** | Otherwise "set to 0" and "unset" are indistinguishable |
| **Version the *package*, not the endpoint** (`ledger.v1`) | Two versions can coexist in one binary |
| **Set a deadline on every call; propagate the remaining budget** | gRPC does this for you — use it |
| **Configure retries declaratively, only for safe status codes** | And still send an idempotency key for writes |
| **Keep `.proto` files in a central repo with CI-enforced compatibility checks** (`buf breaking`) | Backwards-incompatible changes should fail the build, not production |
| **Client-side LB or a mesh — never a naïve L4 balancer** | Long-lived connections pin traffic to one backend |
| **Enable keepalives** | Idle HTTP/2 connections get silently dropped by NATs and LBs |
| **mTLS between services** | The standard for internal traffic ([Lesson 12](../03-security/12-authentication-landscape.md)) |
| **Tenancy from metadata (the credential), never the request message** | Same rule as everywhere |
| **Respect `call.cancelled` and write backpressure in streams** | Otherwise you do work for hung-up clients and OOM on slow ones |

---

## 10. Interview traps

**Q1. "Why is Protobuf smaller and faster than JSON?"**
Field *numbers* instead of names on the wire, plus varint encoding for small integers, plus no text parsing (no string scanning, no number-from-text conversion). Roughly 3× smaller, 3–5× faster to parse. **And the schema is mandatory, so the parser knows the shape in advance** — that's a large part of the speed.

**Q2. "What happens if you change a field number?"**
Data corruption, silently. The old number is unrecognised (skipped as unknown) and the new number may collide with what another field used to mean. Always `reserved` removed numbers *and* names.

**Q3. "Why can't browsers use gRPC?"**
The final status arrives in **HTTP trailers**, and the Fetch API exposes neither trailers nor HTTP/2 framing control. Hence grpc-web (with a proxy, and no client streaming), Connect, or a REST gateway.

**Q4. "When would you choose gRPC over REST?"**
Internal service-to-service, high request volume, polyglot teams needing generated clients, streaming requirements, and where you want declarative deadlines/retries. **When not to:** public APIs, browser clients, anything where `curl`-ability and HTTP caching matter, or a small system where codegen is overhead you don't need.

**Q5. "How do you evolve a gRPC API without breaking clients?"**
Add fields with new numbers (old readers skip unknowns); never change numbers or types; `reserved` on removal; `UNSPECIFIED = 0` on enums so new values degrade gracefully; version the package for genuinely breaking changes; and **enforce it in CI with `buf breaking`** so an incompatible change fails the build.

**Q6. "Does gRPC give you idempotency and retries?"**
Retries yes — declaratively, via `service_config`, for configured status codes. **Idempotency no.** gRPC cannot know whether your method is safe to repeat, so a write still needs an idempotency key. Making that distinction is the answer.

**Q7. "How do you load balance gRPC?"**
The gotcha. Long-lived multiplexed connections defeat L4 balancing, so all requests pin to one backend. Use client-side load balancing (resolve all backends and distribute per request), a service mesh sidecar, or an L7 proxy that balances individual HTTP/2 streams.

**Q8. "How would you serve both external REST and internal gRPC from one service?"**
One `.proto` as the source of truth, with `google.api.http` annotations, and `grpc-gateway` (or Connect) generating the REST façade. One contract, two protocols — no duplicated validation or drift.

**Q9. "Protobuf vs Avro vs Thrift?"**
Protobuf: the default; strong tooling; schema needed at compile time. Avro: the schema travels with the data or lives in a registry, which suits evolving batch/streaming data — dominant in Kafka/Hadoop. Thrift: Facebook's near-equivalent to Protobuf+gRPC, still used but less momentum. Being able to say *why* Avro wins in Kafka (registry-managed, reader/writer schema resolution) is the differentiator.

**Q10. "What's the hardest part of adopting gRPC?"**
Not the protocol — the **tooling and organisational plumbing**: build integration for codegen, a central proto repo with compatibility CI, load balancing that actually works, and the loss of casual debuggability (your team can no longer `curl` a service). Naming the operational cost rather than the technical one is the senior answer.

---

## 11. Build & break

### Build — one service, two protocols
Define `payment_service.proto` for Ledger with `CreatePayment`, `GetPayment`, `ListPayments` (cursor-paginated) and `WatchPayments` (server streaming). Then:
1. Generate TS types and a server stub (`@grpc/proto-loader` or `ts-proto`).
2. Implement the handlers on top of your **existing** service layer — same business logic, second transport. That reuse is the point.
3. Write a client with a 5s deadline and a retry policy for `UNAVAILABLE` only.
4. Compare the same `GetPayment` over REST and gRPC: measure serialized bytes and p50 latency. Write both numbers down.

### Build — prove the evolution rules
1. Add a field with a new number. Old client, new server → the field is ignored. New client, old server → the field is absent. **No errors either way.** That's tolerant reading by design.
2. Rename a field, keeping the number. Wire compatibility is preserved; **generated code breaks**. Observe that the failure is a compile error, not a runtime surprise — which is the good outcome.
3. **The dangerous one:** delete field 6, then add a *different* field with number 6. Have an old client send the original field 6. Watch the new server read it as the new field's type. **Silent corruption.** Now add `reserved 6;` and watch the compiler refuse.
4. Add an enum value and read it with an old client. Confirm it lands on the unknown/`UNSPECIFIED` path rather than throwing.

### Break — three operational failures
1. **Naïve load balancing.** Run 3 server instances behind a round-robin L4 proxy, send 1,000 requests from one client, and count per-instance hits. All on one instance. Switch to client-side LB and re-count.
2. **No deadline.** Call a handler with a 30-second sleep and no deadline. Watch the client hang and the server hold resources. Add `grpc-timeout` and see `DEADLINE_EXCEEDED`.
3. **Streaming without backpressure.** Stream 1M rows to a deliberately slow consumer, ignoring `call.write`'s return value. Watch server memory climb. Add the `drain` wait.

### Explain out loud (2 minutes)
1. Why Protobuf is smaller and faster — two mechanisms.
2. The evolution rules, and why `reserved` matters.
3. Why browsers can't speak gRPC.
4. What gRPC gives you free that you hand-rolled in Lesson 18 — and what it still doesn't.
5. The load-balancing gotcha.

---

## What's next

Both REST and gRPC assume a request that gets an answer. Next: the interaction patterns where the *server* initiates — webhooks, SSE and WebSockets — including the full production design of the webhook system Ledger needs, which is one of the best system-design questions in the API space.

Next → **[Lesson 23: Real-time — WebSockets, SSE & webhooks](23-realtime-and-webhooks.md)**
