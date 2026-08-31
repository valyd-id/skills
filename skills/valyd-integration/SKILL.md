---
name: valyd-integration
description: Integrate Valyd identity — Login with Valyd (OAuth2/TPSSO and OIDC), the Verification APIs (KYC, liveness, anti-spoof, face match, age bands, professional licenses, geolocation), the @valyd/sdk Node package, signed webhooks, the workforce Members API, consent-based attribute release, and the Valyd MCP server. Use whenever a task mentions Valyd, valyd_id, TPSSO, "Login with Valyd", @valyd/sdk, ValydClient, VerifyClient, idp.valyd.work, dev.valyd.work, mcp.valyd.work, workflow_id, X-API-Key verification sessions, vrf_ or whsec_ keys, or verifying a person's identity or license through Valyd.
---

# Valyd integration

Valyd is one identity platform with three surfaces, all issued from **one app** in **one console**:

| Surface | What it does | Credential |
|---|---|---|
| **Login with Valyd** (TPSSO / OIDC) | Users sign in with an already-verified Valyd account | `client_id` + `client_secret` |
| **Verification APIs** | KYC, liveness, anti-spoof, face match, age, license, location | App API key (`vrf_…`) |
| **MCP server** | Agents ask a human to approve by face; run web tasks | User OAuth 2.1 token, scope `mcp` |

One npm package covers the first two: `@valyd/sdk` — `valyd.auth.*` for login, `valyd.verify.*` for verification.

---

## Read this before writing any code

These eleven facts cause almost every Valyd integration bug. Internalise them; the reference files assume you know them.

### 1. The `state` parameter is NOT echoed back — standard OAuth CSRF is broken here

Valyd replaces your `state` with **its own opaque session id** on the callback. `sentState === callbackState` is always false. Comparing them rejects every legitimate login.

Use login sessions instead:

```js
// on /login
const session = await valyd.createLoginSession();      // { authorizeState, marker }
res.cookie("valyd_login", session.marker, { httpOnly: true, maxAge: 600_000 });
res.redirect(valyd.getAuthorizationUrl({ state: session.authorizeState, scope: [...] }));

// on /callback — BEFORE exchangeCode
const { valid } = await valyd.verifyLoginSession(req.cookies.valyd_login);
if (!valid) return res.status(400).send("Invalid login session");
```

Marker TTL is **10 minutes**, it is HMAC-signed, it must be stored server-side only, and `verifyLoginSession` returns `{ valid: boolean }` and never throws. Full detail: `references/login-sessions-csrf.md`.

### 2. Credentials can never be created by an API — a human must use the portal

Signing up, creating an app, copying the `client_secret`, creating the API key, creating a workflow, setting the webhook URL: all human browser steps at `https://dev.valyd.work`. There is **no** password login (magic link or face only) and **no** CI-automatable portal sign-in.

**So: when an integration needs credentials you don't have, stop and ask the human for them.** Do not invent them, and do not go looking for an endpoint that mints them.

`client_secret`, the API key (`vrf_…`) and the webhook secret (`whsec_…`) are each shown **once**. If lost, rotate.

### 3. The SDK defaults to PRODUCTION, but every URL in the docs is DEVELOPMENT

`new Valyd({ ... })` with no `env` targets production (`valyd.id`). Every host documented here and in the references — `idp.valyd.work`, `dev.valyd.work` — is the **development** environment. Against dev credentials, omitting `env` fails with `client_id/redirect_uri not allowed`.

```js
new Valyd({ clientId, clientSecret, apiKey, webhookSecret, env: "development" })
```

One `env` switch sets IdP + Verify + KYC. `ValydClient` / `VerifyClient` take `baseUrl` instead (`VerifyClient` defaults to `https://idp.valyd.work`).

### 4. `redirect_url` on authorize, `redirect_uri` on token

The authorize URL takes **`redirect_url`**. The `POST /token` body takes **`redirect_uri`**. Different spellings, same value — and it must match the portal registration **character for character, with no trailing slash**.

### 5. Verify the webhook HMAC over the RAW body, or it never matches

Signature = `HMAC_SHA256("{timestamp}.{rawBody}", webhookSecret)`, lowercase hex, header `X-Valyd-Signature`. If your framework parses and re-serialises the JSON, the bytes differ and verification always fails. Mount with `express.raw({ type: "application/json" })`.

Also: reject timestamps older than **5 minutes**, compare in constant time, and **deduplicate on `X-Valyd-Event-Id`** — delivery is at-least-once (up to 10 retries over ~2.5 hours, and manual resends reuse the same id).

### 6. The webhook is a notification, not the result

Always call `GET /api/v2/session/{id}/decision` for the real outcome and per-check data. Equally: the `?status=` query param on your redirect-back URL is a **UX hint only** — a user can edit it. Never gate access on it.

### 7. Account mode returns proofs; Non-account mode returns raw data

- **Account (Managed by Valyd)** — you pass the user's `valyd_access_token`. Results are **proofs only**: pseudonym, `id_verified`, license badges, age bands. **Never** raw KYC.
- **Non-account (Fresh)** — no token. You did the capture, nothing is retained, and you get the **raw** extracted data (`fields`, `dob`, portrait, OCR).

Raw account attributes are released **only** through the consent API, sealed to your X25519 key.

### 8. At-login attribute consent is currently DISABLED

The consent screen is login-only right now, so `attr_code` never arrives on the callback. Use the **after-login** path — `requestAttributes` → the user approves in their Valyd app → poll `getAttributeResult`. Anything you read about `attributes` on the authorize URL is retained for when it is re-enabled; do not build on it today.

### 9. Two different auth headers — do not cross them

| Calling | Header |
|---|---|
| `/api/v2/*` (Verification) | `X-API-Key: <App API key>` (or `Authorization: Bearer <apiKey>`) |
| `/api/auth/tpsso/userinfo`, `/licenses`, `/verifications` | `Authorization: Bearer <user access_token>` |
| `/api/sdk/*` (Members, on **dev.valyd.work**) | `X-Client-Id` + `X-Client-Secret` |
| `POST /token`, `POST /refresh` | credentials **in the JSON body** |

The webhook secret is never an outbound auth credential — it only verifies incoming signatures.

### 10. Credential (license) lookups take 10–60 seconds

Set the HTTP timeout to **at least 90 s** (`timeoutMs: 90_000`, `--max-time 90`). The SDK's default is 15 s; it auto-raises to ≥60 s for `credentialVerification` / `kycCredential`, but raw `fetch` and `curl` do not.

### 11. Everything is server-side

`client_secret`, the App API key, the webhook secret, the login marker, and the X25519 secret key all stay on your server. There is **no browser SDK** — the hosted flow is a plain redirect to `session.url`. The only things that reach the browser are that URL and the `session_token`.

---

## The 2×2: pick the integration shape first

Two independent axes. Decide both before writing code.

|  | **Hosted** (Valyd renders capture) | **Core APIs** (you build the UI) |
|---|---|---|
| **Account** (Managed by Valyd) | Login with Valyd → workflow on the hosted page. Steps stored on the account; reuse skips completed steps. **Proofs only.** | REST with the user's token — license badge, face vs stored vector, reuse read. KYC redirects to Valyd. **Proofs only.** |
| **Non-account** (Fresh) | One-shot hosted capture, nothing retained. **Raw data.** | Per-endpoint REST capture in your own UI. **Raw data.** |

```text
Does a human need to take a live selfie / photograph an ID in a browser,
and you don't want to build that camera UI?
  YES -> Hosted     (create a session, redirect to session.url, read the decision)
  NO  -> Core APIs  (POST the images/fields yourself, synchronous JSON back)

Should the verification be stored and reused on later visits/logins?
  YES -> Account      (log the user in first; pass valyd_access_token)
  NO  -> Non-account  (no token)
```

Backoffice, batch, or fully custom UX → Core APIs. Returning-user flows where re-doing KYC is wasteful → Account.

---

## Hosts and base URLs

| Host | Purpose |
|---|---|
| `https://idp.valyd.work` | Everything at runtime: TPSSO, OIDC, and all `/api/v2/*` verification |
| `https://dev.valyd.work` | Developer Portal (human) **and** the Members API (`/api/sdk/*`) |
| `https://mcp.valyd.work` | MCP server (`/verification/mcp`) |
| `https://docs.valyd.work` | Docs, `llms.txt`, OpenAPI specs |

Paths on `idp.valyd.work`:

- Authorize (TPSSO): `/auth`
- TPSSO API: `/api/auth/tpsso/{token,refresh,userinfo,licenses,verifications}`
- OIDC discovery: `/api/.well-known/openid-configuration` — note the **non-standard `/api/` prefix**
- OIDC: `/api/auth/oidc/{authorize,token,userinfo,jwks.json}`
- Verification: `/api/v2/…`
- Consent: `/api/auth/attribute-request`

**Response envelope** — every REST response, both products:

```json
{ "success": true, "data": {}, "error": { "code": "string", "message": "string" } }
```

`error` is present only when `success` is `false`. Two exceptions worth remembering: `POST /token` returns tokens at `data.access_token`, but `POST /refresh` nests them at `data.tokens.access_token`.

---

## Where to look next

Load only the file the task needs. Each one is self-contained.

| Task | File |
|---|---|
| Get credentials; portal steps; passwordless dev sign-in | `references/portal-and-accounts.md` |
| Add "Login with Valyd"; authorize URL; token exchange; refresh | `references/login-oauth.md` |
| The CSRF mechanism in depth | `references/login-sessions-csrf.md` |
| Which scope to request; what each returns; 403s | `references/scopes.md` |
| The five TPSSO endpoints, request/response shapes | `references/tpsso-endpoints.md` |
| Enterprise SSO — Mendix, discovery, JWKS, claim mapping | `references/oidc.md` |
| Get a user's raw legal name / DOB with consent | `references/consent-attributes.md` |
| Choosing Hosted vs Core, Account vs Non-account; reuse | `references/verify-modes-and-account.md` |
| Hosted sessions: create → redirect → webhook → decision | `references/verify-hosted.md` |
| Every `/api/v2/*` check endpoint and its fields | `references/verify-core-apis.md` |
| Signature verification, retries, dedupe, delivery log | `references/webhooks.md` |
| Session and check status values, and how to act on each | `references/statuses.md` |
| `@valyd/sdk` — every class, method, option, type | `references/sdk.md` |
| Every error code, both products, with fixes | `references/errors.md` |
| Workforce members, roles, orgs, billing, private apps | `references/organizations-members.md` |
| Connect an agent over MCP; the three tools | `references/mcp.md` |
| Worked end-to-end builds (KYC, license, EVV, anti-spoof) | `references/recipes.md` |
| Places the published docs contradict themselves | `references/gotchas-and-doc-conflicts.md` |

---

## Minimal working shapes

Three snippets that are correct as written. Expand from the reference files.

**Login with Valyd** — `references/login-oauth.md`

```js
import { ValydClient } from "@valyd/sdk";
const valyd = new ValydClient({
  clientId: process.env.VALYD_CLIENT_ID,
  clientSecret: process.env.VALYD_CLIENT_SECRET,   // server-side only
  redirectUri: process.env.VALYD_REDIRECT_URI,     // exact portal match, no trailing slash
});

app.get("/login", async (req, res) => {
  const s = await valyd.createLoginSession();
  res.cookie("valyd_login", s.marker, { httpOnly: true, sameSite: "lax", maxAge: 600_000 });
  res.redirect(valyd.getAuthorizationUrl({ state: s.authorizeState, scope: ["profile", "verifications"] }));
});

app.get("/callback", async (req, res) => {
  const { code, error } = valyd.parseCallback(req.url);
  if (error || !code) return res.status(400).send(error ?? "missing code");

  const { valid } = await valyd.verifyLoginSession(req.cookies.valyd_login);  // NOT a state compare
  if (!valid) return res.status(400).send("Invalid login session");

  const tokens = await valyd.exchangeCode(code);      // exchange immediately — the code expires fast
  const profile = await valyd.getUserInfo(tokens.accessToken);
  res.clearCookie("valyd_login");
  // ...set your own session
});
```

**Hosted verification** — `references/verify-hosted.md`

```js
import { VerifyClient, ValydVerifyError } from "@valyd/sdk";
const verify = new VerifyClient({
  apiKey: process.env.VALYD_API_KEY,
  webhookSecret: process.env.VALYD_WEBHOOK_SECRET,
});

const session = await verify.sessions.create({
  workflowId: process.env.VALYD_WORKFLOW_ID,
  redirectUrl: `${APP_URL}/verify/callback`,
  callback: `${APP_URL}/webhooks/valyd`,
  vendorData: user.id,              // echoed back on the webhook
});
res.redirect(session.url);

app.post("/webhooks/valyd", express.raw({ type: "application/json" }), async (req, res) => {
  let event;
  try {
    event = verify.webhooks.constructEvent(req.body, req.headers);   // raw Buffer, not parsed JSON
  } catch (err) {
    if (err instanceof ValydVerifyError && err.code === "invalid_signature")
      return res.status(400).send("bad signature");
    throw err;
  }
  if (await alreadyProcessed(event.eventId)) return res.json({ ok: true });   // at-least-once
  const decision = await verify.sessions.decision(event.sessionId);           // the real result
  await persist(event.vendorData, decision);
  res.json({ ok: true });
});
```

**A single Core check** — `references/verify-core-apis.md`

```js
const verify = new VerifyClient({ apiKey: process.env.VALYD_API_KEY, timeoutMs: 90_000 });

const { check } = await verify.standalone.credentialVerification({
  fullName: "Jane Smith",
  licenseState: "CA",
  licenseType: "MD",        // the board is resolved for you
  licenseNumber: "G12345",
});
// check.status: "passed" | "failed" | "review"
// check.data.license.status: "active" | "expired" | ...
```

---

## Before you hand the work back

- [ ] Every secret read from env; nothing hard-coded, nothing in browser-reachable code.
- [ ] The CSRF check is `verifyLoginSession(marker)` — no `state` comparison anywhere.
- [ ] `redirect_url` / `redirect_uri` matches the portal exactly, no trailing slash.
- [ ] The webhook route uses the raw body, checks the timestamp, and dedupes on the event id.
- [ ] The decision comes from `sessions.decision(id)`, never from `?status=`.
- [ ] Credential-lookup timeouts are ≥90 s.
- [ ] `env` (or `baseUrl`) matches the environment the credentials came from.
- [ ] The human has been asked for any credential that requires a portal visit.
