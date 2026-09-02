---
name: valyd-integration
description: Integrate Valyd identity — Connect with Valyd (OpenID Connect sign-in), Reusable Verification (workflow sessions for KYC, liveness, face match, age, professional licenses, location), the Unique Human API (liveness and face uniqueness with no user login), the @valyd/sdk Node package, signed webhooks, the Organization Members API, consent-based attribute release, and the Valyd MCP server. Use whenever a task mentions Valyd, valyd_id, "Connect with Valyd", @valyd/sdk, ValydClient, VerifyClient, idp.valyd.id, dev.valyd.id, mcp.valyd.id, workflowId, verification sessions, vrf_ or whsec_ keys, or verifying a person's identity or license through Valyd.
---

# Valyd integration

Valyd is an identity platform with **two products** and one npm package, `@valyd/sdk`.

| Product | What it answers | Auth | Result |
| --- | --- | --- | --- |
| **Reusable Verification** | Who is this person, and what have they already proven? | `client_id`/`client_secret` for sign-in + App API key for checks | Proofs saved to the user's Valyd ID and reused |
| **Unique Human API** | Is this a live, unique human? | App API key only, no user login | Returned to you; nothing saved to an account |

**Connect with Valyd** is the OIDC sign-in that starts Reusable Verification. It can double as your
app's login button.

---

## Read this before writing any code

Twelve facts that cause almost every Valyd bug. The reference files assume you know them.

### 1. Pick the product first — there is no decision tree beyond this

```text
Do you need to know only whether someone is a live, unique human,
with no account and nothing stored?
  YES -> Unique Human API   (App API key + a liveness/uniqueness workflow)
  NO  -> Reusable Verification (Connect the user, read their proofs, run a workflow for what's missing)
```

The old Hosted-vs-Core / Account-vs-Non-account 2×2 is **retired**. If you find it in older
material, ignore it.

### 2. Every check runs inside a workflow session — there are no per-check REST calls

ID/KYC, face match, liveness, age, professional license and location run **only** as checks inside
a workflow session. There is no `POST /api/v2/id-verification`, no `verify.standalone.faceMatch()`
in the public surface, and you never upload images from your backend. The one flow is always:

```js
const session = await verify.sessions.create({ workflowId, /* ... */ });
// redirect the person to session.url — Valyd runs the camera, capture, retries
const decision = await verify.sessions.decision(session.sessionId);
```

The SDK's remaining `verify.standalone.*` methods are compatibility leftovers and **not part of the
public products** — the only supported ones are the Unique Human anti-spoof calls.

### 3. Workflows are built in the portal — the SDK has no workflow CRUD

`verify.workflows.*` was removed. Compose checks in the Developer Portal → Workflows, copy the
`workflowId`, and pass it to `sessions.create()`. A session runs the workflow as it was when the
session was created.

### 4. `state` IS echoed back — compare it. This reversed in August 2026.

Connect with Valyd is now **standard OIDC**. The IdP echoes your `state` unchanged, and comparing
it is the correct, required CSRF check.

**`createLoginSession()` and `verifyLoginSession()` are deprecated no-ops.** They still compile.
A CSRF check built on them silently passes for everyone — that is a security bug, not a stale API.
Delete any code using them.

```js
// on /login
const transaction = valyd.auth.createAuthorizationRequest({ scope: ["profile", "verifications"] });
req.session.valydOidc = transaction;        // server-side; holds state, nonce, PKCE verifier
res.redirect(transaction.url);

// on /callback
const transaction = req.session.valydOidc;
delete req.session.valydOidc;               // consume once
if (!transaction) return res.status(400).send("Login expired");
const result = await valyd.auth.handleCallback(req.originalUrl, { transaction });
```

`handleCallback()` compares state, sends the PKCE verifier, exchanges the code, verifies the ID
token (RS256/JWKS, issuer, audience, expiry, nonce), and fetches UserInfo — all in one call.

### 5. The legacy TPSSO endpoints are gone — `410 Gone`

`/api/auth/tpsso/token`, `/refresh` and `/tpsso/authorize` were **removed** on 2026-08-18. Every
endpoint now lives under `/api/auth/oidc/*`. Removed endpoints return an explicit `410` with a
pointer, never a silent 404.

### 6. Three tokens, three jobs — never cross them

| Token | Lifetime | Job |
| --- | --- | --- |
| **Access token** | ~15 min (`expires_in` ≈ 900) | Call Valyd APIs as the user |
| **ID token** | validated once at login | Prove *who* logged in, to *your* backend |
| **Refresh token** | 30 days, rotates every use | Mint new access tokens |

**Never send the ID token to an API** — an ID token accepted as an API credential is a security
bug. Treat the access token as **opaque**; don't parse it. The refresh token is an opaque
`rfrsh_…` string, not a JWT.

### 7. Refresh rotation is on, and replay is treated as theft

Every refresh revokes the token you sent and returns a new one — **persist the new value every
time, atomically**. Replaying a rotated-away token revokes the user's entire refresh-token family
for your client.

### 8. Credentials are human-only and environment-scoped

No API mints a `client_id`, `client_secret`, App API key, or `workflowId` — a person creates them
at `https://dev.valyd.id`. **Ask the human rather than inventing them.**

These docs describe the **development** environment (`*.valyd.id`). Production and testing mirror
the layout on their own domains with their **own credentials**. Never mix a key from one
environment with the host of another. Set the host per environment via `VALYD_IDP_URL` (or the
SDK's `baseUrl`).

### 9. Verify the webhook HMAC over the RAW body, and deduplicate

Signature = `HMAC_SHA256("{timestamp}.{rawBody}", webhookSecret)`, lowercase hex, header
`X-Valyd-Signature`. If your framework parses and re-serialises the JSON the bytes differ and
verification fails every time. Mount with `express.raw({ type: "application/json" })`, reject
timestamps older than **5 minutes**, compare in constant time, and **dedupe on `X-Valyd-Event-Id`**
— delivery is at-least-once (up to 10 retries over ~2.5 h; manual resends reuse the id).

### 10. The webhook is a notification; `?status=` is a hint. The decision call is authority.

Always read `verify.sessions.decision(id)` for the real outcome and per-check breakdown. Never gate
access on the `?status=` query param on your redirect URL — a user can edit it.

### 11. You get proofs, not PII

Reusable Verification returns a pseudonym, `id_verified`, license badges and age bands. Documents,
DOB and face data stay encrypted with Valyd. Raw attributes are released **only** through the
consent flow, sealed end-to-end to your X25519 key.

**The at-login consent path is currently disabled** — the consent screen is login-only, so
`attr_code` never arrives. Use the after-login `requestAttributes` flow.

Biometrics are **irreversible vectors, never images**. Valyd does not store or return face images,
and the template is never exposed through any API.

### 12. Read before you verify

The habit that makes Valyd cheap: check what the account already holds (free, instant) and only run
a workflow for what's missing.

```js
const proofs = await valyd.auth.getVerifications(accessToken);
if (proofs.id_verified) { /* done — no check, no cost */ }
```

Everything is server-side: `client_secret`, the App API key, the webhook secret, the OIDC
transaction and the X25519 secret key. There is **no browser SDK** — the flow is a redirect to
`session.url`. The only front end is the drop-in button.

---

## Hosts and base URLs

| Host | Purpose |
| --- | --- |
| `https://idp.valyd.id` | The API — OIDC sign-in, the Account API, and verification |
| `https://dev.valyd.id` | Developer Portal (human) **and** the Organization Members API (`/api/sdk/*`) |
| `https://mcp.valyd.id` | MCP server (`/verification/mcp`) |
| `https://docs.valyd.id` | Docs, API Playground, `llms.txt`, OpenAPI specs |

One API namespace: authorize, token, JWKS, UserInfo, licenses and verifications all sit under
`https://idp.valyd.id/api/auth/oidc`. Discovery is at `/.well-known/openid-configuration` (the
`/api/.well-known/…` alias also works).

**Response envelopes differ by endpoint.** The OIDC token endpoint returns **standard top-level
token JSON with no wrapper**. `/userinfo` returns top-level claims. `/licenses` and
`/verifications` use `{ success, data }`. Read what you get; don't assume.

Every response carries an **`X-Request-Id`** header — log it, and quote it to support. Never send
support your keys, tokens, or identity data.

---

## Where to look next

Load only the file the task needs. Each is self-contained.

| Task | File |
| --- | --- |
| Which product, and the mental model | `references/products.md` |
| Hosts, credential formats, per-environment setup | `references/environments.md` |
| Get credentials; portal steps; passwordless sign-in | `references/portal-and-accounts.md` |
| Add Connect with Valyd — button, flow, token exchange | `references/connect-oidc.md` |
| state / nonce / PKCE, and the deprecated marker pattern | `references/oidc-session-security.md` |
| Access vs ID vs refresh token; validation; logout | `references/tokens.md` |
| Reading a connected user's profile, licenses, proofs | `references/account-api.md` |
| Which scope returns what; 403s | `references/scopes.md` |
| Getting raw legal name / DOB with consent | `references/consent-attributes.md` |
| Create a session → redirect → webhook → decision | `references/verification-sessions.md` |
| Composing workflows in the portal; presets; reuse | `references/workflows.md` |
| Every check, what it returns, where it's available | `references/checks-reference.md` |
| Liveness + uniqueness with no user login | `references/unique-human-api.md` |
| Signatures, retries, dedupe, delivery log | `references/webhooks.md` |
| Session lifecycle, statuses, acting on each | `references/statuses.md` |
| `@valyd/sdk` — classes, methods, options, types | `references/sdk.md` |
| Every error code with fixes | `references/errors.md` |
| Organizations, roles, workforce, billing | `references/organizations-members.md` |
| Enterprise SSO with your own OIDC library | `references/oidc.md` |
| Connecting an agent over MCP | `references/mcp.md` |
| Worked end-to-end builds | `references/recipes.md` |
| Stale patterns, removals, and doc conflicts | `references/gotchas-and-doc-conflicts.md` |

---

## Minimal working shapes

Correct as written. Expand from the reference files.

**Connect with Valyd** — `references/connect-oidc.md`

```js
import { ValydClient } from "@valyd/sdk";
const valyd = new ValydClient({
  clientId:    process.env.VALYD_CLIENT_ID,
  clientSecret: process.env.VALYD_CLIENT_SECRET,   // server-side only
  redirectUri: process.env.VALYD_REDIRECT_URI,     // exact registered match, no trailing slash
});

app.get("/login", (req, res) => {
  const transaction = valyd.auth.createAuthorizationRequest({
    scope: ["profile", "verifications"],
  });
  req.session.valydOidc = transaction;             // state + nonce + PKCE, server-side
  res.redirect(transaction.url);
});

app.get("/auth/valyd/callback", async (req, res) => {
  const transaction = req.session.valydOidc;
  delete req.session.valydOidc;                    // consume once
  if (!transaction) return res.status(400).send("Login expired");

  const { user, accessToken } = await valyd.auth.handleCallback(req.originalUrl, { transaction });
  req.session.user = user;                         // user.valyd_id is your primary key
  res.redirect("/account");
});
```

**Run a verification** — `references/verification-sessions.md`

```js
import { VerifyClient, ValydVerifyError } from "@valyd/sdk";
const verify = new VerifyClient({
  apiKey:        process.env.VALYD_API_KEY,
  webhookSecret: process.env.VALYD_WEBHOOK_SECRET,
});

const session = await verify.sessions.create({
  workflowId:       process.env.VALYD_WORKFLOW_ID,
  valydAccessToken: accessToken,   // omit for the Unique Human API — then nothing is saved
  redirectUrl:      `${APP_URL}/verify/callback`,
  callback:         `${APP_URL}/webhooks/valyd`,
  vendorData:       user.id,       // echoed back on the webhook
  idempotencyKey:   crypto.randomUUID(),
});
res.redirect(session.url);

app.post("/webhooks/valyd", express.raw({ type: "application/json" }), async (req, res) => {
  let event;
  try {
    event = verify.webhooks.constructEvent(req.body, req.headers);   // raw Buffer
  } catch (err) {
    if (err instanceof ValydVerifyError && err.code === "invalid_signature")
      return res.status(400).send("bad signature");
    throw err;
  }
  if (await alreadyProcessed(event.eventId)) return res.json({ ok: true });
  const decision = await verify.sessions.decision(event.sessionId);   // the real result
  await persist(event.vendorData, decision);
  res.json({ ok: true });
});
```

**Read before verifying** — `references/account-api.md`

```js
const proofs = await valyd.auth.getVerifications(accessToken);
if (!proofs.id_verified) {
  // only now create a session for a workflow containing id_verification
}
```

---

## Before you hand the work back

- [ ] No `createLoginSession` / `verifyLoginSession` anywhere — they are no-ops.
- [ ] CSRF is a strict `state` comparison against the stored transaction; nonce is checked too.
- [ ] No `/api/auth/tpsso/*` calls — that namespace returns 410.
- [ ] No direct per-check REST calls or `verify.standalone.*` outside Unique Human anti-spoof.
- [ ] The ID token is validated and never sent to an API.
- [ ] Each rotated refresh token is persisted atomically.
- [ ] The webhook route uses the raw body, checks the timestamp, and dedupes on the event id.
- [ ] The outcome comes from `sessions.decision(id)`, never `?status=`.
- [ ] Host and credentials come from the same environment.
- [ ] The account was read for existing proofs before paying for a check.
- [ ] The human was asked for any credential that requires a portal visit.
