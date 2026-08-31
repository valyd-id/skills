# Choosing your verification integration + Account mode

Two independent axes. Pick both before writing code.

|  | **Hosted** | **Core APIs** |
|---|---|---|
| **Account (Managed by Valyd)** | Login with Valyd → workflow on the hosted page; steps stored on the account; reuse skips already-done steps. **Proofs only.** | REST with the user's token — license (badge on the account), face (vs their stored vector), reuse read. KYC redirects to Valyd. **Proofs only.** |
| **Non-account (Fresh)** | One-shot hosted capture, nothing retained. **Raw data.** | Per-endpoint REST capture in your own UI. **Raw data.** |

## The data-sharing rule (critical)

- **Account APIs return proofs only** — pseudonym, `id_verified`, verified license badges, age
  bands. They **never** return raw KYC (legal name, date of birth, document images). In a decision,
  the `id_verification` check reduces to `{ status, id_verified }`, and `identity` is
  `{ valyd_id, pseudonym, id_verified, age_bands, licenses, verified_at }`.
- **Raw account KYC is released only through the consent Core API** — you request specific
  attributes, the user approves in their Valyd app, and the values return end-to-end encrypted
  (X25519 sealed box) to your public key. See `consent-attributes.md`.
- **Non-account (Fresh) APIs return the captured raw data as-is** — you performed the capture and
  Valyd retains nothing.

## Hosted vs Core APIs

```text
You need an end-user KYC flow with live camera/ID + selfie capture,
and you do NOT want to build the capture UI
  -> Hosted. Redirect to a Valyd-hosted session URL; result via webhook + decision API.
     Next: create a Workflow (get workflow_id) and configure a webhook in the portal,
           then create a session.

You want individual capabilities server-to-server, your own UI and data,
and a synchronous result
  -> Core APIs. Call the per-endpoint REST API (e.g. POST /api/v2/age-verification)
     with your App API key. Next: get an App API key from the portal, then call.

Backoffice, batch processing, or a fully custom UX (no camera redirect)
  -> Core APIs.

Unsure
  -> If a human must take a live selfie / photo of their ID in a browser, choose Hosted.
     Otherwise choose Core APIs.
```

| Aspect | Hosted | Core APIs |
|---|---|---|
| UI | Valyd-hosted capture page | you build it |
| Trigger | redirect user to session URL | server-to-server REST call |
| Result delivery | webhook + decision API | synchronous response |
| Best for | end-user KYC with camera | backoffice, batch, custom UX |
| Identifier | `workflow_id` (bundle of services) | per-endpoint call |

## Account vs Non-account

- **Account** — the verified identity is stored on the user's Valyd account and **reused
  everywhere**. A returning user re-verifies with a **selfie only**, matched against their stored
  face vector; already-verified KYC and licenses are skipped. The data belongs to the user's
  account; integrators get proofs.
- **Non-account** — a one-shot capture, nothing stored, the integrator receives the raw result.

## Services

| Service | Description |
|---|---|
| `id_verification` | KYC / OCR from a government ID |
| `liveness` | Passive liveness check |
| `face_match` | Selfie vs ID portrait |
| `age` | Age band checks from DOB |
| `credential` | Professional license lookup |
| `location` | Geolocation fix / geofence verdict |

A **Workflow** bundles services and exposes a stable `workflow_id`, created in the Developer Portal.

---

## Account × Hosted — the flow

1. Register your app in the portal → `client_id` / `client_secret`; in the **same** portal create a
   Verify project → API key (`vrf_…`, shown once) + `workflow_id`. One console, all credentials.
2. Log the user in with Valyd (OAuth2/TPSSO); exchange the code → `valyd_access_token` + identity.
3. If KYC is required and not done, **redirect the user to Valyd** to complete it. Raw KYC is stored
   under the user's per-user key, so it can't be a plain API write.
4. Create a session: `POST https://idp.valyd.work/api/v2/session` with `workflow_id`,
   `valyd_access_token`, `vendor_data`, and (for redirect) `redirect_url` + `callback`.
5. Redirect the user to the returned hosted `url`. Reuse skips already-completed steps.
6. Read the result via the signed webhook and/or `GET /api/v2/session/{id}/decision` — **proofs
   only**, `origin: "managed"`.

SDK shape (note `env` — see the gotchas file):

```js
import { Valyd } from "@valyd/sdk";
const valyd = new Valyd({ clientId, clientSecret, apiKey, webhookSecret, env: "development" });

// after login
const { accessToken, user } = await valyd.auth.exchangeCode(req.query.code);

// KYC gate — read the account's status; if not verified, redirect to Valyd (there is no KYC API)
const v = await valyd.auth.getVerifications(accessToken);
if (valyd.verify.kyc.isRequired(v))
  return res.redirect(valyd.verify.kyc.redirectUrl({ returnTo: "https://app/next" }));

const session = await valyd.verify.sessions.create({
  workflowId: WORKFLOW_ID,
  valydAccessToken: accessToken,        // <- identifies the person, unlocks reuse
  vendorData: user.valyd_id,
  metadata: { expected_lat: home.lat, expected_lng: home.lng },
});
res.json({ url: session.url });          // returning verified users -> just a face scan
```

## Account × Core APIs

- **License / credential** — matched against the account's real name, the verified badge is stored
  on the account, and a proof is returned. You do **not** pass a name.
- **Face match** — a selfie is matched against the account's stored face vector; only the selfie
  leaves your server. **Never** ask the user for an ID/reference image in this mode.
- **Reuse read / revoke** — `GET /api/v2/identity?valyd_id=…` (proofs only) and
  `DELETE /api/v2/identity/{valyd_id}`.
- **KYC** — redirect the user to Valyd; raw KYC needs the per-user encryption key to store.

```js
// license — provider auto-resolved from state + type; name comes from the account
await valyd.verify.standalone.credentialVerification({
  licenseState: "CO", licenseType: "MD", licenseNumber: "TL.0011377",
});

// face — selfie only, matched to the stored vector
const face = await valyd.verify.standalone.faceMatch({
  valydAccessToken: accessToken,
  selfie,
});
```

## Consent Core API — raw KYC with user approval

Two ways to get raw account attributes; full guide in `consent-attributes.md`.

- **After login (use this today)** — `attribute-request`; the user approves in their Valyd app.
- **At login** — *currently disabled*; the consent screen is login-only.

```bash
# 1) Request specific attributes, sending your X25519 public key
curl -X POST https://idp.valyd.work/api/auth/attribute-request \
  -H "Authorization: Bearer $CLIENT_TOKEN" -H "Content-Type: application/json" \
  -d '{ "client_id": "$CLIENT_ID",
        "attributes": ["legal_name","dob","id_verified"],
        "requester_public_key": "<base64 X25519 pubkey>" }'
# -> { data: { request_id, status: "pending" } }

# 2) The USER approves in their Valyd app. Then:
curl https://idp.valyd.work/api/auth/attribute-request/<request_id>/result \
  -H "Authorization: Bearer $CLIENT_TOKEN"
# -> { data: { status: "approved", sealed_box: "<base64>" } }   # decrypt with your X25519 secret key
```

A second consent surface, `credential-share`, releases a specific vault credential and gates the
release with a face scan as the user's consent.

## Verify once, reuse forever

Because Account mode stores the identity, the **first** visit does full KYC + license and **every
visit after is just a face + location check**. Licenses are re-checked against the board
automatically. Your app stores proofs, not documents.

Practical consequence: **do not put a KYC or license button on every visit.** KYC and license are
one-time onboarding steps; the recurring action is face match (+ location, if relevant).
