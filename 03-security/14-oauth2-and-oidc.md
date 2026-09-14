# Lesson 14 — OAuth 2.1 & OpenID Connect

> **Why this lesson exists:** OAuth is the most confusing widely-used spec in web engineering, mostly because people learn it as a sequence of redirects rather than as an answer to a specific question. Learn the question first and the flows become obvious. This is also near-guaranteed interview material — *"draw the authorization code flow"* and *"what does PKCE prevent?"* are standard, and the second one filters hard.

**Time:** ~90 minutes · **Prereq:** Lesson 13

---

## 1. The idea in one sentence

> **OAuth 2 exists so a user can grant a third-party app *limited* access to their data on another service, without ever giving that app their password — and every part of the spec is machinery for doing that safely across a browser redirect.**

**OAuth is authoriz*ation* (delegated access), not authentic*ation*.** "Log in with Google" is OIDC — a thin, standardised authentication layer built *on top* of OAuth. Confusing the two is the single most common OAuth interview mistake, and §6 makes the distinction concrete.

---

## 2. The problem, before the solution

You're building a bookkeeping app. Your user wants it to read their Ledger payments.

**The pre-OAuth approach ("the password anti-pattern"):** the user types their Ledger password into your app, and you log in as them.

Everything wrong with that:
1. **You have their password.** They must trust your storage, your employees, your breach history.
2. **You have *everything*.** Read payments? You can also issue refunds, change bank details, delete the account.
3. **No revocation.** The only way to cut you off is to change the password — which breaks every *other* integration too.
4. **MFA is impossible.** You can't complete a second factor on the user's behalf.
5. **Ledger can't tell you apart from the user.** No per-app audit trail, no per-app rate limits.

**OAuth fixes all five at once:** the user authenticates *at Ledger*, approves *specific scopes*, and your app receives a **token** that is limited, revocable, attributable, and never the password.

> That five-item list is the best possible answer to *"why does OAuth exist?"* — it shows you understand the problem, so the flows read as consequences rather than ceremony.

---

## 3. The four roles

Get these names right; the spec and every error message uses them.

| Role | Who, in our example |
|---|---|
| **Resource Owner** | The user who owns the payments |
| **Client** | The bookkeeping app requesting access |
| **Authorization Server** | Ledger's auth service — authenticates the user, issues tokens |
| **Resource Server** | Ledger's API — accepts tokens, serves data |

Client types, because the flow depends on it:

| Type | Can it keep a secret? | Examples |
|---|---|---|
| **Confidential** | Yes | A server-side web app with a backend |
| **Public** | **No** | SPAs (source is visible), mobile apps (binaries are decompilable), desktop apps, CLIs |

**"Public client" means the client secret is not secret.** Anything shipped to a user's device can be extracted. That single fact is why PKCE exists.

---

## 4. The authorization code flow + PKCE — draw this from memory

This is **the** flow. OAuth 2.1 makes it the only one for user-facing clients, with PKCE **mandatory for all clients**, public and confidential alike.

```
┌────────┐                  ┌─────────┐                ┌──────────────┐        ┌──────────┐
│  User  │                  │ Client  │                │ Auth Server  │        │   API    │
└───┬────┘                  └────┬────┘                └──────┬───────┘        └────┬─────┘
    │  "connect Ledger"          │                            │                     │
    │───────────────────────────▶│                            │                     │
    │                            │  1. generate:                                    │
    │                            │     code_verifier = random(43-128 chars)          │
    │                            │     code_challenge = BASE64URL(SHA256(verifier))  │
    │                            │     state = random  (CSRF защита)                 │
    │                            │                            │                     │
    │  2. 302 to /authorize?client_id&redirect_uri&response_type=code               │
    │     &scope=payments:read&state=xyz&code_challenge=abc&code_challenge_method=S256
    │◀───────────────────────────│                            │                     │
    │                                                          │                     │
    │  3. user authenticates AT THE AUTH SERVER (password + MFA)                     │
    │─────────────────────────────────────────────────────────▶│                     │
    │  4. consent screen: "Bookkeeping App wants to read your payments"  [Allow]     │
    │─────────────────────────────────────────────────────────▶│                     │
    │                                                          │                     │
    │  5. 302 to redirect_uri?code=SHORT_LIVED_CODE&state=xyz  │                     │
    │◀─────────────────────────────────────────────────────────│                     │
    │                            │                            │                     │
    │  6. browser delivers code  │                            │                     │
    │───────────────────────────▶│                            │                     │
    │                            │  7. POST /token            │                     │
    │                            │     grant_type=authorization_code                │
    │                            │     code=...&code_verifier=ORIGINAL_RANDOM        │
    │                            │     client_id (+ secret if confidential)          │
    │                            │───────────────────────────▶│                     │
    │                            │                            │ verify:             │
    │                            │                            │  SHA256(verifier)   │
    │                            │                            │   == challenge?     │
    │                            │  8. { access_token, refresh_token, expires_in }   │
    │                            │◀───────────────────────────│                     │
    │                            │                            │                     │
    │                            │  9. GET /v1/payments  Authorization: Bearer ...   │
    │                            │─────────────────────────────────────────────────▶│
    │                            │  10. data (scoped to payments:read)              │
    │                            │◀─────────────────────────────────────────────────│
```

### The three questions this flow's design answers

**Q: Why a `code` first, instead of just returning the token?**
Because step 5 travels **through the browser** — in a URL. URLs land in browser history, `Referer` headers, server access logs, and any extension or proxy in the way (Lesson 02, §1). So OAuth sends a **short-lived, single-use, useless-on-its-own code** through the browser, and the actual token is exchanged over a **direct back-channel POST** (step 7) that never touches the URL bar.

**Q: What does `state` do?**
CSRF protection for the redirect. The client generates a random `state`, stores it in the session, and verifies the value that comes back matches. Without it, an attacker can craft a redirect containing *their own* authorization code and trick a logged-in victim's browser into completing the flow — **linking the attacker's account to the victim's app session** (a "login CSRF" / account-injection attack). **Always generate, always verify, always single-use.**

**Q: What does PKCE actually prevent?** — the question that filters candidates.

> **Authorization code interception.** For a public client (mobile/SPA) there's no client secret, so the code alone is enough to get a token. On mobile, the redirect goes to a custom URI scheme (`myapp://callback`) which — on older platforms — **any other installed app could also register**. A malicious app intercepts the redirect, steals the code, and redeems it.
>
> PKCE binds the code to the specific client instance that started the flow. The client invents a random `code_verifier`, sends only its SHA-256 hash (`code_challenge`) up front, and must present the original `code_verifier` to redeem the code. **A stolen code is useless without the verifier, which never left the device.**

The word to use: PKCE makes it a **proof of possession**. Being able to say *"a stolen code is worthless because the attacker doesn't have the verifier, and the verifier never travelled"* is the complete answer.

PKCE is **mandatory in OAuth 2.1 even for confidential clients**, because it also defends against code injection in server-side flows, at essentially zero cost.

---

## 5. The other grants — what's alive, what's dead, and why

| Grant | Status | Use it when |
|---|---|---|
| **Authorization code + PKCE** | ✅ **The default for anything user-facing** | Web apps, SPAs, mobile, desktop |
| **Client credentials** | ✅ Alive | **No user involved** — service-to-service, a cron job accessing its own data |
| **Device authorization** | ✅ Alive | Input-constrained devices: TVs, CLIs, printers. User visits a URL on their phone and types a code |
| **Refresh token** | ✅ Alive | Getting a new access token without re-prompting ([Lesson 13](13-sessions-and-jwt.md)) |
| **Implicit** (`response_type=token`) | ❌ **Removed in 2.1** | Never |
| **Resource owner password credentials** (ROPC) | ❌ **Removed in 2.1** | Never |

### Why implicit died — a proper answer
It returned the access token **directly in the redirect URL fragment**, skipping the code exchange. Its original justification was that browsers couldn't do a cross-origin POST to the token endpoint before CORS was widespread. Once CORS existed, that justification evaporated, and the problems remained:
- The **token itself** goes through the browser URL — history, extensions, `Referer` leakage.
- **No client authentication** at all, and no PKCE, so token injection was possible.
- **No refresh tokens** were issued, so implementations used very long-lived access tokens — exactly the wrong trade.

**Auth code + PKCE gives SPAs everything implicit did, safely.** That's the sentence.

### Why ROPC died
The client collects the user's actual username and password and posts them to the token endpoint. That's the password anti-pattern with extra steps: no MFA, no consent screen, no federated login, and the client handles credentials. It survived only as a migration crutch.

### Client credentials, concretely
```http
POST /oauth/token
Content-Type: application/x-www-form-urlencoded
Authorization: Basic base64(client_id:client_secret)

grant_type=client_credentials&scope=payments:read
```
No user, no redirect, no consent — the app is acting as *itself*. This is the OAuth-flavoured alternative to an API key, and it's worth knowing why you'd choose it: standardised, short-lived tokens with automatic expiry rather than a long-lived static secret.

> Note the `Content-Type`: **the token endpoint requires `application/x-www-form-urlencoded`, not JSON.** A spec mandate that catches people out constantly (Lesson 04, §6).

---

## 6. OIDC — authentication on top of OAuth

OAuth gives you an **access token**: a key to an API. It deliberately says *nothing* about who the user is. So people did the wrong thing — they treated "I got a valid access token" as proof of identity, which breaks in a specific and exploitable way (see Q6 below).

**OpenID Connect (OIDC)** standardises the missing piece:

| | OAuth 2 access token | OIDC ID token |
|---|---|---|
| Purpose | **Authorization** — call an API | **Authentication** — prove who the user is |
| Format | Opaque *or* JWT; the client **must not** parse it | **Always a JWT**, meant for the client to read |
| Audience (`aud`) | The **resource server** (API) | The **client** |
| Contains | Scopes, maybe nothing readable | `sub`, `email`, `name`, `iss`, `aud`, `exp`, `nonce`, `auth_time` |
| Who validates | The API | **The client** |

```http
GET /authorize?...&scope=openid%20profile%20email&nonce=n-0S6
                     ↑ the `openid` scope is what turns OAuth into OIDC
```
```json
{
  "access_token": "...",
  "id_token": "eyJhbGciOiJSUzI1NiJ9...",     // ← the new part
  "token_type": "Bearer",
  "expires_in": 900
}
```

**Validating an ID token — the checklist:** verify the signature against the issuer's JWKS, check `iss`, check `aud === your client_id`, check `exp`, and check **`nonce`** matches the one you sent (this binds the ID token to *your* authorization request, preventing token replay/injection). Same rigour as [Lesson 13 §3](13-sessions-and-jwt.md).

**Discovery** — the reason OIDC is pleasant to implement:
```
GET https://auth.ledger.dev/.well-known/openid-configuration
→ { "authorization_endpoint": ..., "token_endpoint": ..., "jwks_uri": ...,
    "scopes_supported": [...], "response_types_supported": [...] }
```
One well-known URL and your library configures itself. This is why "Log in with Google/Microsoft/Okta" is a config change rather than a project.

> **The rule that keeps you out of trouble:** the **access token is opaque to the client** — never parse it, never make decisions from it, just send it. The **ID token is for the client** — parse and validate it. The **API validates access tokens** and should generally ignore ID tokens entirely.

---

## 7. Scopes: designing them well

Scopes are the *limited* in "limited access", and designing them badly is the most common own-goal.

```
payments:read      payments:write
refunds:write      customers:read
payouts:write      webhooks:manage
```

| Rule | Why |
|---|---|
| **Resource:action shape** | Predictable and readable on a consent screen |
| **Separate read from write** | Most integrations only need read; don't force them to ask for write |
| **Granular enough to be meaningful, coarse enough to be understood** | A consent screen with 40 checkboxes gets blind-approved, defeating the purpose |
| **Never a `*` or `admin` scope for third parties** | It's the password anti-pattern with a token |
| **Enforce on every request, at the resource server** | A scope the API doesn't check is decoration |
| **Return `403 insufficient_scope` and name the required scope** | So the developer can fix it in one try |

```ts
export const requireScope = (needed: string) =>
  (req: Request, _res: Response, next: NextFunction) => {
    const scopes = req.principal.kind === "api_key" || req.principal.kind === "oauth"
      ? req.principal.scopes : ["*"];                  // first-party sessions bypass scopes
    if (!scopes.includes("*") && !scopes.includes(needed)) {
      throw ApiError.forbidden("insufficient_scope",
        `This token lacks the '${needed}' scope`,
        { required_scope: needed, granted_scopes: scopes });
    }
    next();
  };

v1.post("/payments/:id/refunds", requireScope("refunds:write"), createRefund);
```

**Scopes are not a substitute for authorization.** `payments:read` says *"this app may read payments"* — it does **not** say *which* payments. The tenancy check is separate and always required ([Lesson 15](15-authorization-and-multitenancy.md)). Conflating the two is how multi-tenant breaches happen.

---

## 8. The security rules (and the mistakes they prevent)

| Rule | Attack it prevents |
|---|---|
| **Exact-match `redirect_uri` against a pre-registered allowlist** | **Open redirect → code theft.** Never allow wildcards, never match by prefix. `https://app.com/cb` must not match `https://app.com/cb/../../evil` or `https://app.com.evil.com/cb`. This is the #1 OAuth implementation bug |
| **`state` on every request, verified, single-use** | Login CSRF / account injection |
| **PKCE always, `S256` only** (reject `plain`) | Code interception |
| **Authorization codes: single-use, ≤60s, bound to client + redirect_uri + verifier** | Code replay |
| **`nonce` on OIDC requests, verified in the ID token** | ID token replay/injection |
| **Verify `aud` on the API side** | A token minted for another service being replayed at yours |
| **HTTPS on every endpoint and every redirect URI** | Interception (loopback `http://127.0.0.1` is the one allowed exception, for native apps) |
| **Refresh tokens: rotate, detect reuse, revoke the family** | Long-term theft ([Lesson 13](13-sessions-and-jwt.md)) |
| **Show a consent screen naming the app and the scopes; let users revoke per app** | The whole point of OAuth |
| **Rate-limit `/authorize` and `/token`** | Brute force, code guessing |
| **For native apps, use the system browser (ASWebAuthenticationSession / Custom Tabs), never an embedded WebView** | A WebView lets the *app* read the user's password keystrokes — it defeats the entire purpose |

That last one is frequently missed and is a great thing to raise unprompted for a mobile-flavoured question.

---

## 9. Interview traps

**Q1. "What problem does OAuth solve?"**
The five-item list from §2. Lead with *"it lets a user delegate limited, revocable, attributable access to a third party without sharing their password"* and then enumerate what the password anti-pattern costs.

**Q2. "Draw the authorization code flow."**
Draw §4 and narrate the **three design questions**: code-then-token because the redirect is a URL, `state` for CSRF, PKCE for code interception. Drawing boxes is table stakes; explaining *why each hop exists* is the differentiator.

**Q3. "What does PKCE prevent, exactly?"**
Authorization code interception by a malicious app registered for the same custom URI scheme (or any actor who can read the redirect). The verifier never leaves the device, so a stolen code can't be redeemed. **Proof of possession.** Mandatory in 2.1 for all clients.

**Q4. "Why was implicit deprecated?"**
Token in the URL fragment (history/`Referer`/extension leakage), no client authentication or PKCE (token injection), no refresh tokens (so long-lived access tokens). CORS removed the original justification. Auth code + PKCE supersedes it.

**Q5. "OAuth vs OIDC?"**
OAuth = authorization (access token, for an API, opaque to the client). OIDC = authentication layered on top (ID token, a JWT for the client, validated by the client). Trigger: the `openid` scope.

**Q6. "Why can't you use an access token to authenticate a user?"**
The subtle one, and a real historical vulnerability class. An access token only proves *"someone granted this app access to some account"* — it doesn't say **which** account, or that the *current* user is that account. So an attacker can take an access token issued to a *different* app for *their own* account and present it to your login endpoint; if you just call `/userinfo` with it and log in whoever comes back, you've authenticated the attacker as that user. This is the **"confused deputy" / access-token-injection** problem, and it's why OIDC's ID token is bound to your `client_id` via `aud` and to your request via `nonce`. Getting this question right marks you as someone who has read the spec, not just a tutorial.

**Q7. "Where do you store OAuth tokens in a SPA?"**
Same answer as [Lesson 13](13-sessions-and-jwt.md): access token in memory, refresh token in an `HttpOnly` cookie — which, for a third-party SPA, means the **BFF (backend-for-frontend) pattern**: a thin server-side component holds the tokens and the browser only ever holds a session cookie. That's the current OAuth-for-browser-apps recommendation, and naming the BFF pattern is a strong signal.

**Q8. "How do you revoke a third party's access?"**
Per-app: delete the grant, revoke its refresh tokens (and the token family), and let the short access-token lifetime expire the rest. The user-facing requirement is a "connected apps" page with per-app revoke — and it's a legal requirement in some jurisdictions.

**Q9. "Client credentials vs API key?"**
Both authenticate an app with no user. API key: simpler, long-lived, static — you must rotate it manually. Client credentials: standardised, yields **short-lived** tokens with automatic expiry, integrates with existing OAuth infrastructure and scope enforcement. Choose client credentials if you already run an authorization server; API keys if you want the simplest possible developer experience.

**Q10. "A partner says the OAuth flow fails with `redirect_uri_mismatch`. What's wrong?"**
Their registered URI doesn't **exactly** match what they sent — usually a trailing slash, `http` vs `https`, a different port, or an added query parameter. Then say the important part: *"and I would not 'fix' it by relaxing the matching, because exact matching is what prevents code theft via open redirect."*

---

## 10. Build & break

### Build — be the client (the fastest way to make this concrete)
Register an OAuth app with GitHub (free, 2 minutes) and implement the flow by hand — no OAuth library.

```ts
import { createHash, randomBytes } from "node:crypto";
import express from "express";

const app = express();
const CLIENT_ID = process.env.GH_CLIENT_ID!;
const CLIENT_SECRET = process.env.GH_CLIENT_SECRET!;
const REDIRECT_URI = "http://127.0.0.1:3000/callback";
const pending = new Map<string, { verifier: string; createdAt: number }>();

const b64url = (b: Buffer) => b.toString("base64url");

app.get("/login", (req, res) => {
  const verifier = b64url(randomBytes(32));
  const challenge = b64url(createHash("sha256").update(verifier).digest());
  const state = b64url(randomBytes(16));
  pending.set(state, { verifier, createdAt: Date.now() });     // in prod: the session, not a Map

  const url = new URL("https://github.com/login/oauth/authorize");
  url.searchParams.set("client_id", CLIENT_ID);
  url.searchParams.set("redirect_uri", REDIRECT_URI);
  url.searchParams.set("scope", "read:user");
  url.searchParams.set("state", state);
  url.searchParams.set("code_challenge", challenge);
  url.searchParams.set("code_challenge_method", "S256");
  res.redirect(url.toString());
});

app.get("/callback", async (req, res) => {
  const { code, state } = req.query as Record<string, string>;

  // 1. Verify state — single use, and time-bounded.
  const entry = state ? pending.get(state) : undefined;
  pending.delete(state);
  if (!entry) return res.status(400).send("invalid state — possible CSRF");
  if (Date.now() - entry.createdAt > 10 * 60_000) return res.status(400).send("state expired");

  // 2. Exchange the code — back-channel POST, FORM-ENCODED, never JSON.
  const tokenRes = await fetch("https://github.com/login/oauth/access_token", {
    method: "POST",
    headers: { "content-type": "application/x-www-form-urlencoded", accept: "application/json" },
    body: new URLSearchParams({
      grant_type: "authorization_code",
      client_id: CLIENT_ID,
      client_secret: CLIENT_SECRET,
      code,
      redirect_uri: REDIRECT_URI,
      code_verifier: entry.verifier,
    }),
  });
  const { access_token } = await tokenRes.json();

  // 3. Use it. The token is OPAQUE to us — we never parse it.
  const me = await fetch("https://api.github.com/user", {
    headers: { authorization: `Bearer ${access_token}`, accept: "application/vnd.github+json" },
  }).then(r => r.json());

  res.json({ login: me.login, scopes_we_asked_for: "read:user" });
});

app.listen(3000);
```

Doing this by hand once is worth more than reading the spec twice. Then, in your notes, answer: *which step would break if I dropped `state`? Which if I dropped `code_verifier`?*

### Build — be the authorization server (the harder, more valuable half)
For Ledger, implement a minimal `/oauth/authorize` + `/oauth/token`:
- Registered clients with **exact** redirect URIs and a type (public/confidential)
- Authorization codes: 60-second TTL, single-use, bound to `client_id` + `redirect_uri` + `code_challenge`
- `/token`: verifies `SHA256(code_verifier) === code_challenge`, marks the code used, issues access (15 min, with `aud`) + refresh (rotating)
- A consent screen that lists the app name and requested scopes
- A "connected apps" page with per-app revoke

### Break — five attacks on your own server
1. **Replay a code.** Redeem the same authorization code twice. The second must fail. If it succeeds, an intercepted code is reusable forever.
2. **Drop the verifier.** Redeem a code with a *wrong* `code_verifier`. Must fail. Then remove the check server-side and see the code alone suffice — that's the pre-PKCE world.
3. **Skip `state`.** Remove state verification, then craft a callback URL containing your own code and open it in a browser logged into another account. Watch the accounts get linked. That's login CSRF.
4. **Loosen the redirect URI.** Change exact matching to `startsWith`, then register `https://app.com/cb` and send `redirect_uri=https://app.com/cb.evil.com/`. Watch the code get delivered to the wrong host.
5. **Ignore `aud`.** Mint a token with `aud: "https://other-service.dev"` and present it to Ledger's API. If it's accepted, any sibling service's token now works on yours.

Each of these is a real, published vulnerability class. Doing them yourself is how you stop implementing them by accident.

### Explain out loud (2 minutes)
1. Why OAuth exists — the five costs of the password anti-pattern.
2. The auth code + PKCE flow, and why the code hop exists at all.
3. What PKCE prevents, in one sentence, using "proof of possession".
4. OAuth vs OIDC, and why an access token can't authenticate a user.
5. Two rules that prevent code theft.

---

## What's next

You now know who's calling and what they were granted. The next lesson is where APIs *actually* get breached — not authentication, but **authorization**: proving that this caller may touch *this specific object*, in a multi-tenant system, on every single query.

Next → **[Lesson 15: Authorization, multi-tenancy & BOLA](15-authorization-and-multitenancy.md)**
