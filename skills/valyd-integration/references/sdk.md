# `@valyd/sdk` — the Node SDK

One package, both products: **Login with Valyd** on `valyd.auth`, **Verification APIs** on
`valyd.verify`, plus the workforce Members API. One app credential authenticates both; a project API
key works for verification-only use. Everything routes through one host — there is no separate
verify service.

Zero-dependency, dual ESM + CJS, fully typed TypeScript. **Node 18+** (built-in `fetch` / `crypto`).

> **Server-side only.** Your API key must never reach the browser. There is **no browser SDK** — the
> hosted flow is a plain redirect to `session.url`.

```bash
npm i @valyd/sdk
node --version   # must be v18.x or higher
```

## Three entry points

| Class | Use for | Key options |
|---|---|---|
| `Valyd` | both products from one object (`valyd.auth`, `valyd.verify`) | `clientId`, `clientSecret`, `apiKey`, `webhookSecret`, **`env`** |
| `ValydClient` | login only, plus Members API | `clientId`, `clientSecret`, `redirectUri`, `baseUrl` |
| `VerifyClient` | verification only | `apiKey`, `baseUrl`, `webhookSecret`, `timeoutMs`, `fetch` |

### The `env` trap

`new Valyd({ ... })` **defaults to PRODUCTION** (`valyd.id`). Every URL in these docs —
`idp.valyd.work`, `dev.valyd.work` — is the **development** environment. Against dev credentials,
omitting `env` fails with `client_id/redirect_uri not allowed`.

```js
new Valyd({ clientId, clientSecret, apiKey, webhookSecret, env: "development" })
```

One `env` switch sets IdP + Verify + KYC. `ValydClient` and `VerifyClient` use `baseUrl` instead;
`VerifyClient` defaults to `https://idp.valyd.work`.

## `VerifyClient` constructor options

| Option | Type | Default | Description |
|---|---|---|---|
| `apiKey` | string | — | **Required.** Sent as `X-API-Key` on every request. |
| `baseUrl` | string | `https://idp.valyd.work` | Override only for staging/self-hosted. |
| `webhookSecret` | string | — | Optional. When set, `webhooks.constructEvent` / `verify` need no explicit secret. |
| `timeoutMs` | number | `15000` | Per-request timeout. **`credentialVerification` and `kycCredential` automatically use at least 60 s**; raise this only to lift the floor for all calls. |
| `fetch` | typeof fetch | — | Custom fetch (proxies, instrumentation, tests). |

No network call is made on construction. A missing `apiKey` surfaces later as `ValydVerifyError`
with code `config_error`.

## Authentication, precisely

- **`apiKey` (`vrf_…`)** — authenticates all `verify.*` calls. This is the only credential the verify
  side needs.
- **`webhookSecret` (`whsec_…`)** — **not** an outbound auth credential. Used solely to verify the
  HMAC on **incoming** webhooks.
- **`client_id` / `client_secret`** — belong to Login with Valyd (and the Members API). They do
  **not** authenticate verification calls.

There is no constructor form that authenticates verify calls without `apiKey`.

---

## `valyd.auth` / `ValydClient` — login

| Method | Returns | Notes |
|---|---|---|
| `createLoginSession()` | `{ authorizeState, marker }` | Call before redirecting. Marker TTL 10 min. |
| `verifyLoginSession(marker)` | `{ valid: boolean }` | Your CSRF check. Never throws. |
| `getAuthorizationUrl({ state, scope, productName, ... })` | string | `scope` is an array |
| `parseCallback(url)` | `{ code, error, attrCode? }` | |
| `exchangeCode(code)` | tokens (`accessToken`, …) + `user` | Server-side, immediately |
| `refreshToken(stored)` | `{ accessToken, refreshToken }` | 1.6.0+. Rotation on by default — **persist both**. |
| `getUserInfo(token)` | profile | scope `profile` |
| `getVerifications(token)` | verification proofs | scope `verifications` |
| `getLicenses(token)` | licenses | |
| `getDoctorLicense(token)` / `getCprLicense(token)` | license details | scope `doctor_license` |

Consent / attribute release (see `consent-attributes.md`):

| Method | Notes |
|---|---|
| `ValydClient.generateRequesterKeypair()` | static; X25519 `{ publicKey, secretKey }` |
| `requestAttributes({ valydId, attributes, requesterPublicKey, purpose, managedPrivateKey? })` | → `{ requestId }` |
| `getAttributeResult(requestId, { secretKey })` | poll; local decryption when `secretKey` given |
| `getConsentedAttributes(attrCode, { secretKey })` | at-login path — currently disabled |
| `openSealedPayload(...)` | manual sealed-box open |

The self-custody methods lazy-load **`libsodium-wrappers`**, which the SDK does not bundle:
`npm i libsodium-wrappers`.

Workforce Members (see `organizations-members.md`): `getMembers`, `addMembers`, `resolveMember`,
`deactivateMember`, `reactivateMember`, `removeMember`, `resendMemberInvite`, `getBilling`.

---

## `valyd.verify` / `VerifyClient` — verification

### `verify.sessions`

| Method | Description |
|---|---|
| `create(params)` | Create a hosted session. Returns `.url` and `.sessionId`. |
| `retrieve(id)` | Fetch a session by id |
| `list({ status?, vendorData?, limit? })` | List, filterable |
| `decision(id)` | **Authoritative result** with `.checks[]` — call this after the webhook |
| `updateStatus(id, "APPROVED" \| "DECLINED")` | Manual override; only before terminal |

`create` params: `workflowId`, `redirectUrl`, `callback`, `vendorData`, `ttlSeconds`, `metadata`,
and — for Account mode — `valydAccessToken`.

### `verify.workflows`

`create({ name, features, settings? })`, `list()`, `retrieve(id)`, `update(id, patch)`, `remove(id)`.

`features` is e.g. `["id_verification","liveness","face_match","credential","location"]`.

### `verify.standalone`

| Method | Description |
|---|---|
| `idVerification({ frontImage, backImage? })` | OCR + authenticity from a government ID |
| `liveness({ image })` | Passive liveness on a selfie |
| `faceMatch({ idImage, selfie })` | 1:1 face match. Account mode: `{ valydAccessToken, selfie }` |
| `ageVerification({ dob, bands? })` | e.g. `bands: ["is_18_plus"]` |
| `credentialVerification({ ...name, licenseState, licenseNumber, licenseType \| providerCode, npi? })` | Professional license lookup |
| `kycCredential({ frontImage, selfie, backImage?, providerCode, licenseState, licenseNumber, npi? })` | ID + liveness + face match + license, matched against the OCR'd name |
| `locationMatch({ latitude, longitude, accuracy, expectedLatitude, expectedLongitude, radiusM })` | → `POST /api/v2/location` |

For `credentialVerification`, give the holder's **name** as `firstName` + `lastName` **or**
`fullName`; identify the **license** with `licenseType` (Valyd resolves the provider board for you —
no `providerCode` needed) **or** pass `providerCode` directly. In Account mode the name comes from
the connected account; don't pass one.

Field details: `verify-core-apis.md`.

### `verify.credentials`

`states()` and `providers(state)` — the latter returns each provider's `required_fields`. Both are
used in examples as `const { states } = await verify.credentials.states()`.

### `verify.kyc`

`isRequired(verifications)` and `redirectUrl({ returnTo })` — KYC is **not** an API. If the account
needs KYC, redirect the user to Valyd; they complete it there and come back.

### `verify.webhooks`

| Method | Description |
|---|---|
| `constructEvent(rawBody, headers, secret?, { toleranceSeconds? })` | Verifies HMAC, returns the parsed event. Throws `ValydVerifyError` code `invalid_signature`. |
| `verify(rawBody, headers, secret?, { toleranceSeconds? })` | Boolean check — no parse, no throw |

Also exported as top-level `constructEvent` / `verify`. When `webhookSecret` is set on the client,
the `secret` argument is optional.

---

## Helpers & types

```ts
import { readImage, type ImageInput } from "@valyd/sdk";

// ImageInput: Buffer | Uint8Array | base64 string | data-URL string
const fromFile    : ImageInput = readImage("./id_front.jpg");   // reads to base64
const fromBuf     : ImageInput = await fs.promises.readFile("./selfie.jpg");
const fromDataUrl : ImageInput = "data:image/jpeg;base64,/9j/4AAQ...";
```

Public API uses `camelCase`; wire payloads stay `snake_case`.

```ts
import type {
  Session, SessionSummary, Decision, Check, CheckEnvelope,
  KycCredentialResult, Workflow, CredentialState, CredentialProvider, WebhookEvent,
} from "@valyd/sdk";
```

## Error handling

Every failure throws `ValydVerifyError` with `{ code, status?, data? }`. `code` is either an API code
(`API_KEY_INVALID`, `VALIDATION_ERROR`, …) or an SDK code:

| SDK code | Meaning |
|---|---|
| `network_error` | DNS/socket failure |
| `timeout` | exceeded `timeoutMs` |
| `invalid_signature` | webhook HMAC mismatch or stale timestamp |
| `config_error` | missing `apiKey` / `webhookSecret` |

```js
try {
  const { check } = await verify.standalone.credentialVerification({ /* ... */ });
} catch (err) {
  if (err instanceof ValydVerifyError) {
    console.error(err.code, err.status, err.message, err.data);
    if (err.code === "API_KEY_INVALID") { /* rotate / refetch */ }
  } else throw err;
}
```

## Quickstarts

**Hosted**

```js
const verify = new VerifyClient({
  apiKey:        process.env.VALYD_API_KEY,
  webhookSecret: process.env.VALYD_WEBHOOK_SECRET,
});

const session = await verify.sessions.create({
  workflowId:  process.env.VALYD_WORKFLOW_ID,
  redirectUrl: "https://app.example.com/verify/callback",
  callback:    "https://api.example.com/webhooks/valyd",
  vendorData:  "user_123",
});
// res.redirect(session.url)

const event    = verify.webhooks.constructEvent(rawBody, headers);   // throws on bad signature
const decision = await verify.sessions.decision(event.sessionId);
```

**Core APIs**

```js
const verify = new VerifyClient({ apiKey: process.env.VALYD_API_KEY });

const { states }    = await verify.credentials.states();
const { providers } = await verify.credentials.providers("CA");

const result = await verify.standalone.kycCredential({
  frontImage:    readImage("./id_front.jpg"),
  selfie:        readImage("./selfie.jpg"),
  providerCode:  "MD",
  licenseState:  "CA",
  licenseNumber: "A12345",
});
// result.status === "passed" only when ALL checks pass
```

## Verifying your setup

```bash
npm ls @valyd/sdk         # -> @valyd/sdk@x.y.z
```

```js
const verify = new VerifyClient({ apiKey: process.env.VALYD_API_KEY });
const { states } = await verify.credentials.states();
console.log(states.length);   // > 0 means the API key works
```

A throw of `ValydVerifyError` with `API_KEY_INVALID` means the key is wrong or missing.

## Common errors

**`config_error`** — `apiKey` (or `webhookSecret` for webhook calls) not passed to the constructor.

**`invalid_signature` in the webhook handler** — the body was parsed/re-serialized before
verification. Mount with `express.raw({ type: "application/json" })` and confirm the secret matches.

**`timeout`** — exceeded `timeoutMs` (default 15 s). Credential lookups take 10–60 s; use
`timeoutMs: 90_000`.

## Version history that matters

| Version | Change |
|---|---|
| v0.1.0 | `ValydClient`: `getAuthorizationUrl`, `parseCallback`, `exchangeCode`, `refreshToken`; resource helpers `getUserInfo`, `getLicenses`, `getCprLicense`, `getDoctorLicense`, `getVerifications` |
| v0.2.0 | `createLoginSession()` / `verifyLoginSession()`. **The state-equality CSRF pattern was removed.** |
| v1.5.1 | One unified `@valyd/sdk` (`valyd.auth` + `valyd.verify`); Workforce Members API |
| v1.6.0+ | `valyd.auth.refreshToken` |
| v1.8.0 | `resolveMember`, `reactivateMember`. **At-login attribute release disabled** — consent screen is login-only. |
| latest | Core API docs for `antispoof`, `antispoof/identity`, `face-uniqueness`, `location`; relying parties now receive the user's **real legal name**, not the pseudonym |
