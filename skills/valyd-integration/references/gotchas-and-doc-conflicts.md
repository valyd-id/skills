# Gotchas and places the docs contradict themselves

Real contradictions found across the published Valyd docs, plus surprising-but-consistent behaviour.
When one of these matters, **read the actual API response** (or the OpenAPI spec) rather than
trusting either page. Do not silently "fix" a working integration to match a doc page.

Specs: `https://docs.valyd.work/openapi/valyd-id.json` · `https://docs.valyd.work/openapi/valyd-verify.json`

---

## Contradictions

### 1. Authorization code TTL: 2 minutes or 5?

- `docs/endpoints` and the raw-HTTP section of `docs/authentication`: **~2 minutes**.
- The step-by-step flow in `docs/authentication`, and `docs/errors` under `invalid_grant`:
  **5 minutes**.

**Do:** exchange the code the instant your callback fires. Then it never matters. Never design a
flow that parks a code for later.

### 2. Token response nesting differs between `/token` and `/refresh`

- `POST /token` → `data.access_token`
- `POST /refresh` → `data.tokens.access_token`

Not a doc bug — the two endpoints genuinely differ. Code for each specifically.

### 3. Credential-verification response shape

- `verifications/standalone` (Core APIs): the standard envelope —
  `{ success, data: { session_id, status: "passed", check: { type, status, score, data } } }`.
- `verify/verify-license` (the recipe page): a flat object —
  `{ verification_id, status: "APPROVED", checks: [ { type, status, data, error } ] }`.

These are incompatible. The Core APIs page is the reference for the endpoint; the recipe may be
describing an older or a wrapper shape. **Log the actual response once and code to what you get.**

### 4. Credential discovery response fields

- `verifications/standalone`:
  `states: [{ state_name, state_code }]`, `providers: [{ provider_code, provider_display_name, credential_name, required_fields }]`
- `verify/verify-license`:
  `states: [{ state, name, provider_count }]`, `providers: [{ provider_id, name, license_types }]`

Same disagreement. Read the response.

### 5. SDK `credentials.states()` return type

`verifications/sdk` types it as `Promise<CredentialState[]>` but every example destructures
`const { states } = await verify.credentials.states()`. **Follow the examples** — that's the usage
the docs actually exercise.

### 6. Manual session override — which path?

- `verifications/api-reference`: `PATCH /api/v2/session/{id}/status`
- `verifications/hosted`: "POST/PATCH `/api/v2/session/{id}`", explicitly marked as *inferred from
  the SDK method names*.

**Use `PATCH /api/v2/session/{id}/status`** — it's the spelled-out one. Or just call
`verify.sessions.updateStatus(id, "APPROVED")`.

### 7. Are workflows creatable over the API?

- `verifications/api-reference`: "There is no REST endpoint to create a workflow… workflows are
  defined in the Developer Portal."
- `verifications/hosted` and `verifications/sdk`: document `verify.workflows.create/list/retrieve/
  update/remove` mapped to `/api/v2/workflows`.

**Try the SDK; fall back to the portal if it 404s.** Either way you end up with a `workflow_id`.

### 8. Webhook event type `verification.completed`

`evv.mdx` shows `"type": "verification.completed"`, which is **not** in the canonical list
(`verification.approved` / `.declined` / `.in_review` / `.abandoned` / `.expired`). Per the
versioning policy new event types can be added at any time — so **handle unknown types gracefully**
rather than trusting either list exhaustively.

### 9. Webhook `decision` field: string or object?

- `verifications/webhooks`: `"decision": "approved"` — a lowercase string.
- `verifications/hosted`: `"decision": { /* same shape as the decision API */ }` — an object.

Doesn't matter if you follow the rule: **the webhook is a notification; read the real result from
`GET /api/v2/session/{id}/decision`.**

### 10. `GET /licenses` scope

`docs/endpoints` says "Bearer access token required (no specific scope declared on this endpoint in
the source)", while `docs/scopes` associates license data with `doctor_license`. If you get a 403,
request `doctor_license`.

### 11. Access token lifetime: TPSSO vs OIDC

- TPSSO: `expires_in: 900` — **15 minutes**.
- OIDC: ID token 15 min, **access token 1 hour**, refresh token 30 days.

Different products, genuinely different lifetimes. Don't apply one to the other.

---

## Surprising but consistent

### The `state` parameter is not echoed

The single biggest trap. See `login-sessions-csrf.md`.

### `redirect_url` vs `redirect_uri`

Authorize takes `redirect_url`. The token body takes `redirect_uri`. Same value, two spellings.

### OIDC discovery has a non-standard `/api/` prefix

`https://idp.valyd.work/api/.well-known/openid-configuration`, not
`https://idp.valyd.work/.well-known/openid-configuration`. Some strict OIDC libraries construct the
standard path from the issuer and will fail — configure the discovery URL explicitly, or enter the
endpoints manually.

### The SDK defaults to production, but the docs are all development

`new Valyd({...})` with no `env` targets `valyd.id`. Every documented host (`idp.valyd.work`,
`dev.valyd.work`) is **development**. Pass `env: "development"` with dev credentials, or OAuth fails
with `client_id/redirect_uri not allowed`.

### The Members API is on a different host

`https://dev.valyd.work/api/sdk/*` — the portal host — not `idp.valyd.work`. And it authenticates
with `X-Client-Id` / `X-Client-Secret`, not a Bearer token or the API key.

### Refresh token rotation is on by default and reuse is punished

Every refresh returns a new token and revokes the old one. Replaying a rotated-away token revokes
**every** refresh token for that user and client. Store the replacement atomically.

### A single-image anti-spoof score is capped at 85

`POST /api/v2/antispoof` with one `image` can never exceed `human_score: 85` (`assurance: "upload"`).
For a real verdict send `frames[]` (burst), or use the hosted flow for `assurance: "captured"`.

### Liveness is a hard equality, not a threshold

`check.status === "passed"` only when `live_score === 1`. `0` is spoof; `< 0` means no face detected.

### Location is never advisory

A real GPS fix is mandatory. Blocked permission or missing coordinates is a hard `failed` — never a
pass, never a review. With an expected point **and** `radius_m`, the status **is** the geofence
verdict.

### Credential lookups need first/last name even when `required_fields` omits it

The registry always needs a name. Collect it regardless of what the provider metadata says.

### At-login attribute consent is disabled right now

The consent screen is login-only. `attr_code` never arrives. Use `requestAttributes` (after login).

### `libsodium-wrappers` is not bundled

The self-custody consent methods lazy-load it. `npm i libsodium-wrappers` or you get
`No such module libsodium-wrappers` the first time you decrypt.

### There is no browser SDK

Hosted verification is a plain redirect to `session.url`. Older EVV snippets reference browser
helpers like `captureVisit()` / `captureLocation()` / `connectButton()` alongside comments saying no
browser SDK exists. **Build capture with `navigator.mediaDevices` + `navigator.geolocation`** and
POST to your own server.

### Relying parties now get the real legal name

A changelog entry states every relying party receives the user's **real legal name**, not the
pseudonym — while the Account-mode "proofs only" rule still lists a pseudonym among the returned
proofs. Don't assume you'll get a pseudonym; don't assume you'll get raw KYC either. Inspect what
comes back.

### Public demo credentials exist in the docs

The anti-spoof page publishes a real `client_id` / `client_secret` / app key / `workflow_id` on a
balance-capped demo account. They're deliberately public and exist for the demo. **Never copy them
into a real integration**, and never treat them as an example of safe secret handling.
