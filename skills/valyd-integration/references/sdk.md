# `@valyd/sdk` — the Node SDK

One package covers both products: **Connect with Valyd** on `valyd.auth`, **verification** on
`verify.sessions`, plus the Organization Members API. Zero-dependency, dual ESM + CJS, fully typed;
**Node 18+** (built-in `fetch` / `crypto`).

```bash
npm i @valyd/sdk        # unversioned — always pulls the latest published release
node --version          # must be v18.x or higher
```

> **Server-side only.** Your API key must never reach the browser. There is **no browser SDK** —
> the verification flow is a redirect to `session.url`, and the only front end is the
> [drop-in button](connect-oidc.md).

## Entry points

| Class | For | Key options |
| --- | --- | --- |
| `ValydClient` | Connect with Valyd (OIDC) + Members API | `clientId`, `clientSecret`, `redirectUri`, `baseUrl` |
| `VerifyClient` | Verification sessions | `apiKey`, `baseUrl`, `webhookSecret`, `timeoutMs`, `fetch` |

### `VerifyClient` constructor options

| Option | Type | Default | Description |
| --- | --- | --- | --- |
| `apiKey` | string | — | **Required.** The App API key sent on every request. |
| `baseUrl` | string | `https://idp.valyd.work` | The API host **for your environment**. |
| `webhookSecret` | string | — | When set, `webhooks.constructEvent` / `verify` need no explicit secret. |
| `timeoutMs` | number | `15000` | Per-request timeout. |
| `fetch` | typeof fetch | — | Custom fetch (proxies, instrumentation, tests). |

No network call is made on construction. A missing `apiKey` surfaces later as `ValydVerifyError`
with code `config_error`.

Set `baseUrl` from `VALYD_IDP_URL` so the same code deploys to every environment — see
[`environments.md`](environments.md).

## Authentication

- **`apiKey` (`vrf_…`)** — authenticates all `verify.*` calls. The only credential the verify side
  needs.
- **`webhookSecret` (`whsec_…`)** — **not** an outbound credential. Used solely to verify the HMAC
  on **incoming** webhooks.
- **`client_id` / `client_secret`** — belong to Connect with Valyd (and the Members API). They do
  **not** authenticate verification calls.

A full Reusable Verification build holds both families, used independently.

---

## `valyd.auth` — Connect with Valyd

| Method | Notes |
| --- | --- |
| `createAuthorizationRequest({ scope, redirectUri? })` | **Recommended entry point.** Generates strong `state`, `nonce`, and an S256 PKCE verifier/challenge together. Store the transaction server-side; redirect to `transaction.url`. |
| `getAuthorizationUrl({ state, nonce, codeChallenge, scope, redirectUri? })` | Low-level URL builder. `state` is required. Prefer `createAuthorizationRequest()` so PKCE and nonce cannot be forgotten. |
| `handleCallback(url, { transaction })` | **One call:** compares state, sends the PKCE verifier, exchanges the code, verifies RS256/JWKS + issuer/audience/expiry/nonce, and fetches UserInfo. |
| `exchangeCode(code)` | Exchanges at `POST /api/auth/oidc/token`; verifies the ID token before returning `{ accessToken, refreshToken, idToken, claims, expiresIn, scope, tokenType }`. |
| `refreshToken(refreshToken)` | Refreshes with `grant_type: "refresh_token"`. **Rotation is on — persist the returned `refreshToken` every time.** |
| `getUserInfo(token)` | Profile |
| `getVerifications(token)` | Proofs and badges |
| `getLicenses(token)` | Professional licenses |

The SDK adds the required `openid` scope automatically.

### Deprecated no-ops — never use

`createLoginSession()` and `verifyLoginSession()` still compile but **do nothing**. A CSRF check
built on them silently passes for everyone. Use `createAuthorizationRequest()` +
`handleCallback()`. See [`oidc-session-security.md`](oidc-session-security.md).

### Consent / attribute release

| Method | Notes |
| --- | --- |
| `ValydClient.generateRequesterKeypair()` | static; X25519 `{ publicKey, secretKey }` |
| `requestAttributes({ valydId, attributes, requesterPublicKey, purpose, managedPrivateKey? })` | → `{ requestId }` |
| `getAttributeResult(requestId, { secretKey })` | poll; local decryption when `secretKey` is given |
| `getConsentedAttributes(attrCode, { secretKey })` | at-login path — **currently disabled** |
| `openSealedPayload(...)` | manual sealed-box open |

These lazy-load **`libsodium-wrappers`**, which the SDK does not bundle: `npm i libsodium-wrappers`.
See [`consent-attributes.md`](consent-attributes.md).

### Organization Members

`getMembers`, `addMembers`, `resolveMember`, `deactivateMember`, `reactivateMember`,
`removeMember`, `resendMemberInvite`, `getBilling` — see
[`organizations-members.md`](organizations-members.md).

---

## `verify.sessions` — the whole verification surface

| Method | Description |
| --- | --- |
| `create(params)` | Create a verification session. Returns `.url` and `.sessionId`. |
| `retrieve(id)` | Fetch a session by id |
| `list({ status?, vendorData?, limit? })` | List, filterable by status / vendorData |
| `decision(id)` | **Authoritative result** with `.checks[]` — call after the webhook |
| `updateStatus(id, "APPROVED" \| "DECLINED")` | Manual override; non-terminal sessions only |

`create` params: `workflowId`, `redirectUrl`, `callback`, `vendorData`, `ttlSeconds`, `metadata`,
`idempotencyKey`, and — for Reusable Verification — `valydAccessToken`.

**Every billable operation accepts `idempotencyKey`**, sent as the `Idempotency-Key` header so a
network retry can never double-charge or double-run. Use a UUID per logical operation and reuse it
across retries.

> **This is also the Unique Human API.** Same surface, no user token: create a session for a
> workflow containing the anti-spoof and/or face-uniqueness checks, redirect to `session.url`, read
> `decision()`.

`faceUniquenessUnlink(valydUuid)` — GDPR erasure: forget this project's link to a face id.

### What was removed

| Removed | When | Use instead |
| --- | --- | --- |
| `verify.workflows.*` CRUD | v1.10.4 | Compose in the Developer Portal; pass the `workflowId` |
| `verify.standalone.evvPresence` | v1.10.4 | The `/evv-presence` endpoint never existed server-side. Compose presence from face match + location checks in a workflow. |
| `verify.standalone.idVerification`, `faceMatch`, `locationMatch`, `ageVerification`, `credential`, `kycCredential` | v1.10.5 | Run them as **workflow checks** in a session |
| `verify.kyc.redirectUrl()` | v1.10.5 | Run KYC as a workflow check |

The public surface is now **the hosted flow only**: `valyd.auth`, `verify.sessions.*`, and the
Unique Human anti-spoof calls (`verify.standalone.antispoof` / `antispoofIdentity`). Any remaining
`verify.standalone.*` methods are compatibility leftovers and **not part of the public products**.

## `verify.credentials`

- `states()` — supported states
- `providers(state)` — providers (license types) in a state, with `required_fields`
- `types(state?, provider?)` — credential/license types: whole catalog, per-state, or
  per-provider-in-a-state

Examples destructure the result: `const { states } = await verify.credentials.states()`.

## `verify.webhooks`

| Method | Description |
| --- | --- |
| `constructEvent(rawBody, headers, secret?, { toleranceSeconds? })` | Verifies the HMAC, returns the parsed event. Throws `ValydVerifyError` code `invalid_signature`. |
| `verify(rawBody, headers, secret?, { toleranceSeconds? })` | Boolean check — no parse, no throw |

Also exported as top-level `constructEvent` / `verify`.

---

## Types

Public API uses `camelCase`; wire payloads stay `snake_case`.

```typescript
import type {
  Session, SessionSummary, Decision, Check, CheckEnvelope,
  Workflow, CredentialState, CredentialProvider, WebhookEvent,
} from "@valyd/sdk";
```

`readImage` and `ImageInput` are still exported, but with direct per-check calls removed there is
normally nothing to pass them to — Valyd's verification page performs the capture.

## Error handling

Every failure throws `ValydVerifyError` with `{ code, status?, data? }` — either an API code
(`API_KEY_INVALID`, `VALIDATION_ERROR`, …) or an SDK code:

| SDK code | Meaning |
| --- | --- |
| `network_error` | DNS / socket failure |
| `timeout` | exceeded `timeoutMs` |
| `invalid_signature` | webhook HMAC mismatch or stale timestamp |
| `config_error` | missing `apiKey` / `webhookSecret` |

```javascript
import { VerifyClient, ValydVerifyError } from "@valyd/sdk";

const verify = new VerifyClient({ apiKey: process.env.VALYD_API_KEY });

try {
  const session = await verify.sessions.create({
    workflowId:  process.env.VALYD_WORKFLOW_ID,
    redirectUrl: "https://yourapp.com/checked",
  });
} catch (err) {
  if (err instanceof ValydVerifyError) {
    console.error(err.code, err.status, err.message, err.data);
    if (err.code === "API_KEY_INVALID") { /* rotate / refetch */ }
  } else throw err;
}
```

## Verifying your setup

```bash
npm ls @valyd/sdk
```

```javascript
const verify = new VerifyClient({ apiKey: process.env.VALYD_API_KEY });
const { states } = await verify.credentials.states();
console.log(states.length);   // > 0 means the API key works
```

A `ValydVerifyError` with `API_KEY_INVALID` means the key is wrong, missing, or from another
environment.

## Common errors

**`config_error`** — `apiKey` (or `webhookSecret` for webhook calls) wasn't passed to the
constructor.

**`invalid_signature` in the webhook handler** — the body was parsed/re-serialized before
verification. Mount with `express.raw({ type: "application/json" })` and confirm the secret.

**`timeout`** — exceeded `timeoutMs` (default 15 s). Sessions whose workflow includes a credential
check can involve 10–60 s registry lookups; raise the timeout if you poll synchronously.

**A method that no longer exists** — check the removals table above. The SDK moved to a hosted-flow
-only surface in v1.10.5.

## Version history that matters

| Version | Change |
| --- | --- |
| v1.10.1 | **Standard OIDC end to end.** `createAuthorizationRequest()` + `handleCallback()`. **The IdP now echoes `state`** — the standard comparison is the required CSRF check; the marker pattern is deprecated. |
| v1.10.2 | Anti-spoof as first-class SDK methods; `antispoofChallenge()`; optional `idempotencyKey` on billable checks |
| v1.10.3 | `verify.credentials.types()` |
| v1.10.4 | **`verify.workflows.*` CRUD removed**; `evvPresence` removed |
| v1.10.5 | **Standalone direct checks hidden** — public surface is the hosted flow only |
