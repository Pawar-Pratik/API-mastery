# Lesson 23 — Real-time: WebSockets, SSE & webhooks

> **Why this lesson exists:** every API eventually needs to tell someone that something happened, and the naïve answer — "the client polls" — is wrong often enough to be a real interview filter. This lesson covers the five options and when each is correct, then designs Ledger's **webhook system in full**, because *"design a webhook delivery system"* is one of the best API system-design questions there is: it forces signing, retries, ordering, idempotency, SSRF, and dead-lettering into one answer.

**Time:** ~100 minutes · **Prereq:** Lessons 02, 12, 18

---

## 1. The idea in one sentence

> **Choose your real-time mechanism by asking two questions — *who initiates?* and *is the recipient a browser or a server?* — because those two answers eliminate three of the five options immediately.**

---

## 2. The five options

| Mechanism | Direction | Recipient | Connection | Use when |
|---|---|---|---|---|
| **Short polling** | Client asks repeatedly | Any | New each time | Simple, low-frequency, tolerant of delay |
| **Long polling** | Client asks, server holds | Any | Held until data or timeout | You need push but can't use SSE/WS |
| **SSE** | Server → client | **Browser** | One held HTTP response | Server-push to a browser, text only |
| **WebSocket** | Both ways | Browser or server | One upgraded TCP connection | Genuinely bidirectional, low latency |
| **Webhooks** | Server → **server** | A server with a public URL | New request per event | Notifying *other companies'* systems |

### The decision tree

```
Who needs to be told?
├─ Another company's SERVER          → WEBHOOK  (they can't hold a connection to you)
└─ A browser/app you control
   ├─ Do they need to SEND too, at low latency?
   │  ├─ Yes (chat, cursors, games, collaboration)  → WEBSOCKET
   │  └─ No  (notifications, live feed, progress, LLM tokens) → SSE
   └─ Is a few seconds of delay fine, or is the volume tiny?
      └─ POLLING — and don't apologise for it
```

**Polling is underrated and you should be willing to defend it.** It's stateless, trivially load-balanced, works through every proxy and firewall, has no reconnection logic, degrades gracefully, and is easy to debug. With `ETag`/`304` ([Lesson 17](../04-production/17-caching-and-performance.md)) a poll that finds nothing new costs ~150 bytes. *"I'd poll every 5 seconds with conditional requests, and only move to SSE if we measure that it isn't enough"* is a perfectly strong answer — and choosing the simplest sufficient mechanism is a senior instinct.

The real argument against polling is arithmetic: 10,000 clients polling every 2 seconds is **5,000 req/s** of mostly-empty responses. That's when push earns its complexity.

---

## 3. Server-Sent Events

Plain HTTP, one long-lived response, a trivially simple text format. **The most under-used tool in this list.**

```http
GET /v1/events/stream HTTP/1.1
Accept: text/event-stream

HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive
X-Accel-Buffering: no                  ← tells nginx NOT to buffer. Essential.

id: 01HQ8ZK3
event: payment.succeeded
data: {"id":"pi_3Nx8","amount_minor":4999}

id: 01HQ8ZK4
event: payment.failed
data: {"id":"pi_3Nx9","code":"card_declined"}

: heartbeat                            ← a comment line keeps proxies from timing out
```

```ts
export function paymentStream(req: Request, res: Response) {
  res.writeHead(200, {
    "Content-Type": "text/event-stream",
    "Cache-Control": "no-cache, no-transform",
    "Connection": "keep-alive",
    "X-Accel-Buffering": "no",
  });

  // Resume from where the client left off — the killer SSE feature.
  const lastId = req.header("last-event-id");
  const send = (e: Event) => {
    res.write(`id: ${e.id}\nevent: ${e.type}\ndata: ${JSON.stringify(e.data)}\n\n`);
  };

  // 1. Replay anything missed while disconnected.
  if (lastId) for (const e of eventsAfter(req.merchantId, lastId)) send(e);

  // 2. Then subscribe to live events.
  const unsubscribe = bus.subscribe(req.merchantId, send);

  // 3. Heartbeat, or an idle proxy will close the connection at 30–60s.
  const hb = setInterval(() => res.write(": heartbeat\n\n"), 15_000);

  // 4. Always clean up — otherwise you leak a subscription per dropped client.
  req.on("close", () => { clearInterval(hb); unsubscribe(); });
}
```
```ts
// Client — and note how little code this is
const es = new EventSource("/v1/events/stream");
es.addEventListener("payment.succeeded", e => update(JSON.parse(e.data)));
// Reconnection with Last-Event-ID is AUTOMATIC. You write nothing.
```

**Why SSE is the right default for one-way push:**
- **Automatic reconnection with resume.** The browser reconnects and sends `Last-Event-ID` itself. To match this over WebSocket you must write reconnection, backoff and a resume protocol by hand.
- It's **just HTTP**: your existing auth (cookies!), logging, tracing, load balancers and CORS all work unchanged.
- Trivial to implement server-side — no library needed.

**The four real limitations:**
1. **Text only.** Binary must be base64'd (+33%).
2. **One-way.** Client→server goes over normal requests, which is fine for most apps.
3. **HTTP/1.1 connection limit.** 6 connections per origin per browser, and an SSE stream holds one *permanently* — so 2 tabs with 3 streams each and the app deadlocks. **Over HTTP/2 this disappears** (100+ streams per connection), so **SSE requires HTTP/2 in practice.** This is the #1 gotcha and the thing to mention.
4. **Proxies buffer.** nginx buffers responses by default and your events arrive in clumps or never. Hence `X-Accel-Buffering: no` plus `proxy_buffering off`.

---

## 4. WebSockets

```http
GET /ws HTTP/1.1
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13

HTTP/1.1 101 Switching Protocols       ← the only 101 you'll ever meet
Upgrade: websocket
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

After the `101`, HTTP is gone. It's a bidirectional frame-based protocol over the same TCP connection.

### The four hard problems (this is what interviews probe)

**1. Authentication.** The browser's `WebSocket` constructor **cannot set headers** — so no `Authorization: Bearer`. Your options:

| Approach | Verdict |
|---|---|
| Token in the query string (`wss://…?token=…`) | ❌ **Logged everywhere** ([Lesson 02](../01-foundations/02-journey-of-a-request.md)) |
| Cookie (sent automatically on the handshake) | ✅ Works well for first-party; needs CSRF thought — see below |
| **First message after connect is an auth frame** | ✅ **The standard answer.** Connect unauthenticated, authenticate immediately, close if it doesn't arrive within ~5s |
| A short-lived single-use ticket from a REST call | ✅ Best of both: `POST /ws-tickets` → a 30-second one-time token → pass it in the query string |

> **The `Origin` check matters here.** WebSocket handshakes are **not** subject to CORS, and cookies *are* sent — so a malicious page can open an authenticated socket to your server ("cross-site WebSocket hijacking"). **You must validate the `Origin` header on the handshake yourself.** That's a great detail to volunteer.

**2. Scaling.** A WebSocket is stateful — it's pinned to one server instance. So:
- You need sticky routing or connection-aware load balancing.
- **A message for user X must reach whichever instance holds X's socket** → a pub/sub backbone (Redis pub/sub, NATS, Kafka) that every instance subscribes to.
- Deploys drop every connection at once; the resulting **reconnect thundering herd** needs jittered backoff on the client ([Lesson 18](../04-production/18-reliability-and-idempotency.md)).
- Each connection costs memory (a few KB) plus a file descriptor; 100k connections per instance is achievable but needs tuning (`ulimit`, socket buffers).

```
       ┌── instance A ── sockets for users 1,4,7
LB ────┼── instance B ── sockets for users 2,5,8      all subscribe to Redis pub/sub
       └── instance C ── sockets for users 3,6,9
```

**3. Heartbeats.** Intermediaries kill idle connections silently, and TCP won't tell you for minutes. **Ping/pong every 30 seconds and terminate on a missed pong** — otherwise you accumulate "zombie" connections that consume memory and never receive anything.

```ts
const wss = new WebSocketServer({ noServer: true });

// Origin validation on upgrade — CORS does NOT protect you here.
server.on("upgrade", (req, socket, head) => {
  if (!ALLOWED_ORIGINS.includes(req.headers.origin ?? "")) {
    socket.write("HTTP/1.1 403 Forbidden\r\n\r\n"); socket.destroy(); return;
  }
  wss.handleUpgrade(req, socket, head, ws => wss.emit("connection", ws, req));
});

wss.on("connection", (ws, req) => {
  let principal: Principal | null = null;
  ws.isAlive = true;
  ws.on("pong", () => { ws.isAlive = true; });

  // Auth-by-first-message, with a hard deadline.
  const authTimer = setTimeout(() => ws.close(4001, "auth timeout"), 5_000);

  ws.on("message", async raw => {
    if (raw.length > 64 * 1024) return ws.close(1009, "message too large");   // bound it
    const msg = safeJson(raw);
    if (!principal) {
      principal = await authenticateToken(msg?.token);
      if (!principal) return ws.close(4003, "unauthorized");
      clearTimeout(authTimer);
      subscribe(principal.merchantId, ws);
      return ws.send(JSON.stringify({ type: "ready" }));
    }
    await handle(principal, msg, ws);
  });

  ws.on("close", () => { clearTimeout(authTimer); unsubscribe(ws); });
});

// Reap zombies.
setInterval(() => {
  for (const ws of wss.clients) {
    if (!ws.isAlive) { ws.terminate(); continue; }
    ws.isAlive = false; ws.ping();
  }
}, 30_000);
```

**4. Backpressure.** If you `send()` faster than a client can receive, messages queue **in your server's memory**. A slow mobile client can OOM your process.
```ts
if (ws.bufferedAmount > 1_000_000) {         // 1 MB backed up
  ws.close(1013, "client too slow");          // drop them; don't let them take you down
  return;
}
ws.send(payload);
```
**`bufferedAmount` is the check nobody writes and everybody needs.**

---

## 5. Webhooks: the full production design

This is the section to know cold. Webhooks are how Ledger tells merchants that money moved, and it's a complete distributed-systems problem in miniature.

### The architecture

```
payment succeeds
  → INSERT into outbox, in the SAME transaction as the payment   (Lesson 18)
  → dispatcher picks it up
  → for each matching webhook_endpoint:
       create a delivery row (status=pending)
  → worker pool sends HTTP POST with an HMAC signature
       → 2xx  → mark delivered
       → else → schedule a retry with exponential backoff
       → after N attempts → dead-letter, alert the merchant, auto-disable the endpoint
```

### The payload, and the thin-vs-fat decision

```http
POST https://merchant.example.com/webhooks/ledger
Content-Type: application/json
Ledger-Signature: t=1773480600,v1=5257a869e7ec...,v1=8b2f01c4a9de...
Ledger-Event-Id: evt_01HQ8ZK3
Ledger-Event-Type: payment.succeeded
Ledger-Delivery-Attempt: 1
User-Agent: Ledger-Webhooks/1.0

{
  "id": "evt_01HQ8ZK3",
  "type": "payment.succeeded",
  "created_at": "2026-03-14T09:30:00Z",
  "api_version": "2026-03-14",
  "livemode": true,
  "data": {
    "object": {
      "id": "pi_3Nx8", "object": "payment",
      "amount_minor": 4999, "currency": "usd", "status": "succeeded"
    },
    "previous_attributes": { "status": "requires_capture" }
  }
}
```

| | **Thin** (`{id, type, object_id}`) | **Fat** (the full object) |
|---|---|---|
| Receiver must fetch | Yes — one extra API call | No |
| Data freshness | **Always current** (they fetch now) | A snapshot; may be stale on arrival |
| Ordering problems | **Mostly avoided** — they read current state | Real: an older event can arrive after a newer one |
| Payload size / PII exposure | Minimal | Larger, and sends data to a third-party URL |
| Load on your API | Higher | Lower |

**The right answer is usually fat with a thin escape hatch**: send the object (so simple receivers need no API call), include `id` and `type` so sophisticated receivers can re-fetch current state, and **document that events may arrive out of order** so anyone doing state transitions knows to verify. Stripe does exactly this.

### Signing — the receiver's only defence

```ts
const signed = `${timestamp}.${rawBody}`;
const v1 = createHmac("sha256", endpoint.secret).update(signed).digest("hex");
// Send ALL current secrets during rotation:
const header = `t=${timestamp},` + secrets.map(s => `v1=${hmac(signed, s)}`).join(",");
```

Four rules, all covered in [Lesson 12](../03-security/12-authentication-landscape.md) and all load-bearing:
1. **Sign `timestamp + raw body`**, so the signature covers *when* and *what*.
2. **Receiver rejects timestamps older than ~5 minutes** — otherwise a captured request replays forever.
3. **Constant-time comparison.**
4. **Support two live secrets** so rotation has no downtime.

And the receiver-side gotcha worth repeating because it burns everyone: **verify against the raw bytes, before JSON parsing.** Re-serialising changes key order and whitespace, and the signature will never match.

### Retries

```ts
// Backoff schedule — total window ~3 days. Publish it; merchants build alerting on it.
const SCHEDULE_SEC = [0, 5, 30, 120, 600, 3_600, 21_600, 86_400, 172_800];

function nextAttemptAt(attempt: number): Date | null {
  if (attempt >= SCHEDULE_SEC.length) return null;             // dead-letter
  const base = SCHEDULE_SEC[attempt];
  const jitter = Math.random() * base * 0.2;                    // ±20% — Lesson 18
  return new Date(Date.now() + (base + jitter) * 1000);
}
```

| Rule | Why |
|---|---|
| **Retry on timeout, connection error, and 5xx** | Transient |
| **Do NOT retry on 4xx (except 429)** | Their endpoint rejected it deliberately; retrying won't help |
| **`410 Gone` → disable the endpoint immediately** | They've told you it's retired |
| **Short timeout (5–10 s) per attempt** | A slow receiver must not consume your worker pool |
| **Jitter every delay** | 10,000 endpoints failing together would retry in lockstep |
| **Cap total attempts (~8, over ~3 days), then dead-letter** | Infinite retries are a resource leak and a DoS on the receiver |
| **Auto-disable after sustained failure, and email the merchant** | An abandoned endpoint shouldn't cost you delivery capacity forever |
| **Cap concurrency per endpoint** | One slow merchant must not starve the others (a bulkhead — Lesson 18) |

### Ordering — say the honest thing

**You cannot guarantee ordered delivery over independent HTTP requests with retries.** Event A fails and retries at +30s; event B succeeds immediately. B arrives first.

Your options, and the trade-off is the answer:
1. **Don't guarantee it. Document it.** Include a monotonic `sequence` and `created_at` so receivers can detect out-of-order events and re-fetch current state. **This is what everyone does, and it's correct.**
2. **Per-endpoint serial delivery** — one in-flight delivery per endpoint, ordered. Guarantees order, but **head-of-line blocking**: one stuck event delays everything behind it for days.
3. **Per-entity ordering** — serialise per `payment_id` while parallelising across payments. A good middle ground, and more work.

> **The interview answer:** *"I wouldn't promise ordering. I'd include a sequence number and the resource ID, document that events can arrive out of order, and tell receivers to treat the webhook as a signal to read current state rather than as a state transition. Guaranteeing order means serial delivery per endpoint, which converts one stuck event into a multi-day backlog."*

### Idempotency, from the receiver's side
At-least-once delivery means **duplicates are guaranteed** — a receiver returns 200 but their response is lost, so you retry. So the contract must say: *"deduplicate on `Ledger-Event-Id`; the same event may be delivered more than once."* That's the mirror image of [Lesson 18](../04-production/18-reliability-and-idempotency.md), and it's why every serious webhook doc includes that sentence.

### The endpoint registration problem (SSRF)
A merchant-supplied URL is attacker-controlled input ([Lesson 16](../03-security/16-owasp-and-hardening.md)):

```ts
export async function registerEndpoint(url: string, merchantId: string) {
  await assertSafeUrl(url);                    // https only, public IP only, no metadata ranges

  // Prove they control the endpoint before sending it real events.
  const challenge = randomUUID();
  const res = await safeFetch(url, {
    method: "POST",
    body: JSON.stringify({ type: "endpoint.verification", challenge }),
    headers: { "Ledger-Signature": sign(...) },
    signal: AbortSignal.timeout(5_000),
  });
  if (res.status < 200 || res.status >= 300)
    throw ApiError.badRequest("endpoint_verification_failed",
      "The endpoint must return 2xx to the verification request");
  // ...
}
```
Two extra rules: **re-validate the URL on every delivery**, not just at registration (DNS records change, and a hostname that resolved publicly at registration can later point at `10.0.0.5`), and **require HTTPS** — you're sending payment data to it.

### The data model

```sql
CREATE TABLE webhook_endpoints (
  id            text PRIMARY KEY,
  merchant_id   uuid NOT NULL,
  url           text NOT NULL,
  secret        text NOT NULL,                 -- encrypted at rest
  previous_secret text,                        -- rotation window
  enabled_events text[] NOT NULL,              -- ['payment.*','refund.succeeded']
  status        text NOT NULL DEFAULT 'active',-- active | disabled | failing
  api_version   text NOT NULL,                 -- pinned! Lesson 11
  created_at    timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE webhook_deliveries (
  id            text PRIMARY KEY,
  endpoint_id   text NOT NULL REFERENCES webhook_endpoints(id),
  event_id      text NOT NULL,
  attempt       int  NOT NULL DEFAULT 0,
  status        text NOT NULL,                 -- pending|delivered|failed|dead
  next_attempt_at timestamptz,
  response_status int,
  response_body   text,                        -- TRUNCATED to ~2KB
  duration_ms     int,
  UNIQUE (endpoint_id, event_id)                -- ← one delivery record per event per endpoint
);
CREATE INDEX ON webhook_deliveries (status, next_attempt_at) WHERE status = 'pending';
```

Three details that make this production-grade:
- **`api_version` pinned per endpoint**, so an API version change doesn't alter the payload shape a merchant's parser expects ([Lesson 11](../02-rest-design/11-versioning-and-evolution.md)).
- **`UNIQUE (endpoint_id, event_id)`** so a dispatcher bug can't create duplicate delivery *records*.
- **Store the (truncated) response** — it's the entire content of the merchant's support ticket. *"You returned 500 with this body at 14:03"* ends the conversation.

### The merchant-facing features that make it usable
Not optional extras — these are what separate a webhook system people trust from one they complain about:
- **A delivery log** in the dashboard: every attempt, status, response body, duration.
- **Manual retry** of a specific delivery (`POST /webhook-endpoints/{id}/deliveries/{did}/retry`).
- **An event log** they can poll as a fallback (`GET /v1/events?after=evt_x`) — **this is the most important one**: it means a merchant who was down for six hours can reconcile without you doing anything.
- **A test-event sender**, so they can develop against it.
- **Secret rotation** with an overlap window.

---

## 6. Production rules

| Rule | Why |
|---|---|
| **Poll with `ETag`/304 before reaching for push** | Simplest sufficient mechanism; a no-op poll costs ~150 bytes |
| **SSE for one-way browser push; require HTTP/2** | Auto-reconnect with resume for free; HTTP/1.1's 6-connection limit deadlocks tabs |
| **SSE: heartbeat + `X-Accel-Buffering: no`** | Proxies close idle connections and buffer streams |
| **WebSocket: validate `Origin` on the handshake** | CORS does not apply; cookies do — cross-site hijacking otherwise |
| **WebSocket: auth-by-first-message or a short-lived ticket, never a token in the URL** | URLs are logged everywhere |
| **WebSocket: ping/pong + terminate on missed pong** | Zombie connections leak memory |
| **WebSocket: check `bufferedAmount` and drop slow clients** | A slow client will OOM your server |
| **WebSocket at scale: pub/sub backbone + sticky routing + jittered client reconnect** | Sockets are pinned to instances; deploys drop everything at once |
| **Webhooks: outbox in the same transaction as the state change** | Otherwise events are silently lost on a crash |
| **Webhooks: HMAC over `timestamp.rawBody`, two live secrets, ±5 min tolerance** | Origin + integrity + replay protection + rotation |
| **Webhooks: exponential backoff with jitter, ~8 attempts over ~3 days, then dead-letter + notify** | Bounded effort, and merchants can build alerting on a published schedule |
| **Webhooks: short per-attempt timeout, per-endpoint concurrency cap** | One slow receiver must not starve the rest |
| **Webhooks: don't promise ordering; ship `sequence` + `created_at` and document it** | Ordered delivery means serial delivery means head-of-line blocking |
| **Webhooks: tell receivers to dedupe on event ID** | At-least-once means duplicates are certain |
| **Webhooks: validate the URL at registration *and* on every delivery; HTTPS only** | SSRF, and DNS changes after registration |
| **Webhooks: pin the API version per endpoint** | A payload-shape change would break their parser |
| **Webhooks: expose a pollable event log** | Lets merchants self-serve any recovery |

---

## 7. Interview traps

**Q1. "How do you notify a client that a long-running job finished?"**
Ask who the client is. **Browser** → SSE (or poll a job resource). **Another company's server** → webhook. **Your own service** → a queue. Then give the REST shape: `POST /jobs` → `202` + `Location` → the client polls or receives a callback ([Lesson 03](../01-foundations/03-http-methods-and-status.md)).

**Q2. "SSE vs WebSocket?"**
SSE: one-way, plain HTTP, **automatic reconnect with `Last-Event-ID` resume**, works with existing auth/proxies/tracing, text only, needs HTTP/2 to avoid the 6-connection limit. WebSocket: bidirectional, binary, lower per-message overhead, but you hand-write reconnection, auth, heartbeats and backpressure, and you need sticky routing plus pub/sub to scale. **Default to SSE unless you genuinely need client→server at low latency.**

**Q3. "Design a webhook system."**
Structure your answer: outbox for reliability → dispatcher fan-out per endpoint → worker pool with short timeouts → HMAC signing with timestamp → exponential backoff with jitter and a capped attempt count → dead-letter with merchant notification and auto-disable → a delivery log and manual retry → **a pollable event log as the fallback** → and the honest statements about ordering (not guaranteed) and duplicates (guaranteed). Hitting all of that is a complete answer.

**Q4. "How does a receiver know a webhook really came from you?"**
HMAC-SHA256 over `timestamp.rawBody` with a shared secret, verified in constant time, with a ±5-minute window and multiple accepted signatures during rotation. Then the gotcha: **verify the raw bytes before parsing.**

**Q5. "Can you guarantee webhooks arrive in order?"**
No — retries reorder them by construction. Give the three options and recommend #1 (document it, ship a sequence number, tell receivers to read current state), naming head-of-line blocking as the cost of the alternative.

**Q6. "A merchant's endpoint is down for 6 hours. What happens?"**
Retries follow the published backoff schedule (~3 days of attempts). After the cap, deliveries dead-letter, the endpoint is marked failing/disabled, and the merchant is emailed. **Crucially, no events are lost** — they're all in the event log, so the merchant can reconcile via `GET /v1/events?after=…` and/or replay specific deliveries. That last part is the difference between a good and a bad answer.

**Q7. "How do you authenticate a WebSocket?"**
Not with a header (the browser API can't set them). Auth-by-first-message with a close-on-timeout, or a short-lived single-use ticket obtained over REST. Plus **validate `Origin`**, because CORS doesn't apply and cookies are sent.

**Q8. "You have 500,000 concurrent WebSocket connections. What breaks?"**
Memory and file descriptors per connection; sockets pinned to instances so a deploy drops all of them at once and the reconnect storm can self-DDoS (fix: jittered backoff); message fan-out needs pub/sub, and a naïve "publish to all" pattern makes every instance do all the work; slow clients back up in server memory (`bufferedAmount`); and health checks must not count a busy socket pump as unhealthy. Mention that this is exactly why managed services (Pusher, Ably, API Gateway WebSockets) exist.

**Q9. "Isn't polling bad?"**
No — it's often correct. Stateless, easy to scale and debug, works everywhere, and with `ETag`/304 an empty poll is tiny. The argument against it is arithmetic (10k clients × 0.5 Hz = 5k req/s), so measure before adding push. Being willing to defend the simple option is a positive signal.

**Q10. "A merchant complains they never got an event."**
Answer with the workflow, which proves you'd built for it: look up the delivery record by event ID → see attempts, response statuses and bodies → three cases: (a) you delivered a 200 and they lost it internally (show them the log), (b) their endpoint 500'd (show them their own response body), (c) the event was never generated (a bug on your side — check the outbox). **Then offer a manual retry.** That triage is only possible because you stored the response.

---

## 8. Build & break

### Build — Ledger's webhook system
1. `outbox` table + a dispatcher that fans events out to matching endpoints.
2. A worker: HMAC signing, 8-second timeout, backoff schedule with jitter, per-endpoint concurrency cap of 5.
3. `webhook_deliveries` with attempt history and truncated response bodies.
4. Endpoint registration with `assertSafeUrl` + a verification challenge.
5. Auto-disable after 8 consecutive failures + a notification.
6. `GET /v1/events?after=` as the pollable fallback, and `POST …/deliveries/{id}/retry`.
7. Secret rotation: two live secrets, both signatures sent.

### Build — a correct receiver (do both sides; it's the fastest way to learn)
```ts
app.post("/webhooks/ledger",
  express.raw({ type: "application/json", limit: "1mb" }),      // RAW body — before parsing
  async (req, res) => {
    const raw = req.body.toString("utf8");
    if (!verify(req.header("ledger-signature") ?? "", raw, [CURRENT, PREVIOUS]))
      return res.status(400).json({ error: "invalid_signature" });

    const event = JSON.parse(raw);

    // Idempotency: at-least-once means duplicates WILL happen.
    const inserted = await db.query(
      `INSERT INTO processed_events (event_id) VALUES ($1)
        ON CONFLICT DO NOTHING RETURNING event_id`, [event.id]);
    if (inserted.rowCount === 0) return res.status(200).json({ status: "duplicate_ignored" });

    // ★ ACK FAST, process async. A slow receiver gets retried and duplicated.
    await queue.add("ledger-event", event);
    res.status(200).json({ received: true });
  });
```
**That "ack fast, process async" pattern is the single most important thing a webhook receiver does.** If you process synchronously and take 30 seconds, the sender times out, retries, and you process twice — while holding a connection.

### Build — an SSE feed for the Console
`GET /v1/events/stream` with `Last-Event-ID` replay, 15-second heartbeats, and cleanup on close. Then verify: open it, kill your network for 30 seconds, restore, and confirm the missed events arrive — **without writing any client reconnection code.** That demonstration is why SSE deserves your default.

### Break — six experiments
1. **Sign the parsed body** instead of the raw bytes on the receiver. Send a payload with reordered keys. Watch a legitimate event fail verification.
2. **Remove the timestamp check.** Capture a valid webhook with `ngrok`/`webhook.site`, replay it an hour later, and watch it be accepted. That's replay.
3. **Process synchronously for 30 seconds.** Watch the sender time out and retry; count how many times you processed the same event. Then add the fast ack + dedupe.
4. **Remove jitter** and point 200 endpoints at a service that's down for 60 seconds. Graph delivery attempts per second — clean spikes. Add jitter and watch it smooth.
5. **Remove the per-endpoint concurrency cap** and make one endpoint sleep 10 seconds. Watch it consume your whole worker pool while every other merchant's events sit queued.
6. **Open 7 SSE streams over HTTP/1.1** from one browser origin. Watch request #7 (and all normal API calls) hang forever. Then serve over HTTP/2 and watch it work.

### Explain out loud (2.5 minutes)
1. The two questions that pick your mechanism, and the decision tree.
2. Why SSE is the default for one-way push, and its one hard prerequisite.
3. The four hard problems of WebSockets.
4. The full webhook architecture, in order.
5. Why you won't promise ordering, and what you give receivers instead.

---

## What's next

You now know all five paradigms. The final Module 5 lesson is the synthesis: the decision framework for *"REST vs GraphQL vs gRPC vs…"*, what the industry actually uses where, and how to answer the architecture question without sounding like you're picking a favourite.

Next → **[Lesson 24: Choosing the right API style](24-choosing-an-api-style.md)**
