# Stale patterns, removals & doc conflicts

Two kinds of hazard: **things that changed** (older code and older docs are now wrong), and
**things the published docs still disagree about**. When one matters, read the actual API response
or the OpenAPI spec rather than trusting any page.

Specs: `https://docs.valyd.work/openapi/valyd-id.json` ·
`https://docs.valyd.work/openapi/valyd-verify.json`

---

## Part 1 — Breaking changes you will meet in existing code

### The `state` CSRF rule reversed (2026-08-18)

Valyd's old TPSSO flow did **not** echo `state`; it substituted its own opaque session id, and the
documented workaround was a login-session "marker". **That is now backwards.** The IdP echoes
`state` unchanged and the standard comparison is the required CSRF check.

**`createLoginSession()` / `verifyLoginSession()` are deprecated no-ops.** They still compile and
still return, so a CSRF check built on them **silently passes for everyone** — a security bug, not
a stale API. Delete them. See [`oidc-session-security.md`](oidc-session-security.md).

### Legacy TPSSO endpoints removed — `410 Gone` (2026-08-18)

`POST /api/auth/tpsso/token`, `/refresh`, and `/tpsso/authorize` are **removed**. Use
`/api/auth/oidc/*`. Policy: removed endpoints return an explicit `410` with a pointer, never a
silent 404 — so a 410 from Valyd means "you're on the old namespace".

The old TPSSO base `https://idp.valyd.work/api/auth/tpsso` and the old authorize URL
`https://idp.valyd.work/auth?...&redirect_url=...` are both gone. The current authorize endpoint is
`/api/auth/oidc/authorize` and the parameter is `redirect_uri` throughout — the old
`redirect_url` / `redirect_uri` split no longer exists.

### Workflow CRUD removed from the SDK (v1.10.4)

`verify.workflows.create/list/retrieve/update/remove` are gone. Workflows are composed in the
Developer Portal; pass the resulting `workflowId` to `sessions.create()`. They return if and when
the server contract becomes a confirmed public API.

### Direct per-check calls hidden (v1.10.5)

`verify.standalone.idVerification`, `faceMatch`, `locationMatch`, `ageVerification`, `credential`,
`kycCredential`, and the `verify.kyc.redirectUrl()` helper are **no longer exposed**. Run these as
workflow checks in a session. The public surface is now the hosted flow only: `valyd.auth`,
`verify.sessions.*`, and the Unique Human anti-spoof calls.

Any `POST /api/v2/<check>` code — `/id-verification`, `/liveness`, `/face-match`,
`/age-verification`, `/credential-verification`, `/kyc-credential`, `/location` — is calling a
surface that is no longer a documented public API.

### `evvPresence` removed (v1.10.4)

`verify.standalone.evvPresence` and the `/evv-presence` endpoint **never existed server-side** —
it always 404'd. Compose presence from face-match + location checks in one workflow.

### The integration decision tree is retired

The Hosted × Core / Account × Non-account 2×2 and the "Choosing an integration" page are retired.
There are **two products** — [Reusable Verification and the Unique Human API](products.md) — and
the choice between them is the only one.

### Age check: `verified` → `satisfied` (2026-08-19)

`bands.*.verified` is a **deprecated alias**. Read `satisfied` — same value, honest name.

---

## Part 2 — Places the docs still disagree

### The SDK `env` option vs `VALYD_IDP_URL`

`environments.md` in the docs documents per-environment hosts via `VALYD_IDP_URL` (SDK: `baseUrl`).
The EVV page still shows a `Valyd({ ..., env: "development" })` constructor option, warning that
without it the SDK defaults to production. Both may work; **prefer `baseUrl` / `VALYD_IDP_URL`**,
which is the documented mechanism, and be aware the `env` form exists in older code.

### `/licenses` scope

The endpoint reference says "Bearer access token required (no specific scope declared on this
endpoint in the source)", while the scopes page associates license data with `doctor_license`. If
you get a 403, request `doctor_license`.

### Response envelopes are genuinely inconsistent

Not a doc bug — the API really varies:

| Endpoint | Shape |
| --- | --- |
| `POST /api/auth/oidc/token` | **standard top-level token JSON, no wrapper** |
| `GET /api/auth/oidc/userinfo` | **top-level OIDC claims, no wrapper** |
| `GET /api/auth/oidc/licenses` | `{ success, data }` |
| `GET /api/auth/oidc/verifications` | `{ success, data }` |
| Verification API | `{ success, data, error }` |

Read what you get; don't assume a wrapper.

### `verify.credentials.states()` return type

Typed as `Promise<CredentialState[]>` but every example destructures
`const { states } = await verify.credentials.states()`. **Follow the examples** — that is the usage
the docs exercise.

### Webhook `decision` field: string or object?

The webhooks page shows `"decision": "approved"` (a lowercase string); other material shows an
object. It doesn't matter if you follow the rule: **the webhook is a notification; read the real
result from `sessions.decision(id)`.**

### Webhook event types

The canonical list is `verification.approved` / `.declined` / `.in_review` / `.abandoned` /
`.expired`. The EVV page shows a `verification.completed` that isn't in it. New event types can be
added at any time — **handle unknown types gracefully** rather than trusting any list exhaustively.

### Older EVV code samples

The EVV page predates the v1.10.4/v1.10.5 removals and still shows
`verify.standalone.credentialVerification(...)`, `faceMatch(...)`, `locationMatch(...)` and
`verify.kyc.redirectUrl(...)` as direct calls, plus browser helpers (`captureVisit()`,
`captureLocation()`, `connectButton()`) alongside comments saying no browser SDK exists. **Treat
that page's code as illustrative of the EVV *use case*, not of the current API.** The current
shape is one workflow bundling the checks — see [`recipes.md`](recipes.md).

---

## Part 3 — Surprising but correct

**OIDC discovery has an `/api/` prefix.** `https://idp.valyd.work/api/.well-known/openid-configuration`.
The standard `/.well-known/…` path also works, but strict libraries that construct the path from
the issuer may need the URL given explicitly.

**Two hosts, two credential families.** The Members API is on `https://dev.valyd.work/api/sdk/*` —
the *portal* host — and authenticates with `X-Client-Id` / `X-Client-Secret`, not a Bearer token or
the App API key.

**Refresh rotation punishes replay.** Every refresh revokes the token you sent. Replaying a
rotated-away token revokes the user's entire refresh-token family for your client. Persist the
replacement atomically.

**A single-image anti-spoof score is capped at 85.** For a real verdict use a burst, or the hosted
capture which yields `assurance: "captured"`.

**Location is never advisory.** A real GPS fix is mandatory; blocked permission is a hard `failed`.
Where a radius is configured, the status **is** the geofence verdict.

**At-login consent is disabled.** The consent screen is login-only, so `attr_code` never arrives.
Use the after-login `requestAttributes` flow.

**`libsodium-wrappers` is not bundled.** The self-custody consent methods lazy-load it —
`npm i libsodium-wrappers`.

**There is no browser SDK.** Verification is a redirect to `session.url`. The only front end is the
drop-in `signin/client.js` button.

**Testing costs real money.** Checks run for real — no simulated results — and bill against your
wallet. New accounts get a $100 welcome credit.

**Two things called "session".** A *login session* is three OIDC tokens. A *verification session*
is one run through a workflow. "Session expired" from a resource API means refresh the access
token; `EXPIRED` from the decision API means create a new verification session.

**Public demo credentials exist in the docs.** The anti-spoof page publishes a real
`client_id`/`client_secret`/app key on a balance-capped demo account. They are deliberately public.
Never copy them into a real integration, and never treat them as an example of safe secret
handling.
