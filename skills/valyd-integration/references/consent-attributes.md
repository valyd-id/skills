# Requesting raw user data (consent)

> **Status: the at-login path is temporarily DISABLED.** The consent screen is login-only right now —
> the user just signs in, no data is released, and `attr_code` never arrives on your callback. Use
> the **after-login** path (`requestAttributes`) for all data requests today. The at-login sections
> are kept for when it is re-enabled; do not build on them.

Login and the verification APIs return **proofs** — a pseudonym, `id_verified`, license badges, age
bands. When you need a user's **raw personal attributes** (legal name, date of birth, country), you
request them explicitly, the user consents, and the values come back **end-to-end encrypted**,
sealed to a key only you hold. Valyd never sees them.

Runs on `valyd.auth` (the `ValydClient`) — **not** `valyd.verify`.

**You need:** `VALYD_CLIENT_ID`, `VALYD_CLIENT_SECRET` (server-side), the subject's `valyd_id`
(from login), and a backend to hold the X25519 secret key.

## Available attributes

Three groups, with escalating requirements. **Prefer proofs over raw fields** — request
`is_18_plus` rather than `dob` whenever it suffices.

**Proofs** — non-identifying; release on consent alone, no face or vault needed:

| Key | Value |
| --- | --- |
| `id_verified` | boolean — the user has completed KYC |
| `is_16_plus`, `is_18_plus`, `is_21_plus`, `is_30_plus`, `is_65_plus` | boolean age bands, derived, no raw DOB |
| `preferred_username` | pseudonymous username |

**Raw identity** — real PII kept server-readable; needs a face-assured (or quick in-page face) session:

| Key | Value |
| --- | --- |
| `legal_name` / `full_name` | full legal name |
| `first_name`, `last_name` | given / family name |
| `email`, `phone` | contact |
| `country` | country |

**Vault-only raw KYC** — sealed **on the user's device** from their encrypted identity vault; the
server is blind to these. The user must have their vault **unlocked on the device they consent on**;
if it isn't, the request is refused with an "unlock your vault" prompt. These are never silently
dropped:

| Key | Value |
| --- | --- |
| `dob` | date of birth (`YYYY-MM-DD`) |
| `age` | age in years, derived on-device |
| `gender` | gender / sex from the ID |
| `nationality` | nationality from the ID |
| `document_number` | ID document number |

---

## After login — the supported path

Generate a keypair, request the attributes with the user's `valyd_id`, then poll. The user approves
in their Valyd app (notification → face approval). Passing your `secretKey` to `getAttributeResult`
opens the sealed box locally, so plaintext never leaves your server.

```js
import { Valyd, ValydClient } from "@valyd/sdk";

const valyd = new Valyd({
  clientId: process.env.VALYD_CLIENT_ID,
  clientSecret: process.env.VALYD_CLIENT_SECRET,   // server-side only
});

// 1. X25519 keypair. Keep secretKey SERVER-SIDE and PERSIST it for this request —
//    a fresh keypair cannot open a payload sealed to an older key.
const { publicKey, secretKey } = await ValydClient.generateRequesterKeypair();

// 2. Request. valydId comes from the logged-in user; they are prompted in their Valyd app.
const { requestId } = await valyd.auth.requestAttributes({
  valydId,
  attributes: ["legal_name", "dob", "country"],
  requesterPublicKey: publicKey,
  purpose: "Confirm your legal name for payroll onboarding",   // shown on the consent prompt
});

// 3. Poll. secretKey makes the SDK decrypt locally; Valyd stays blind.
const result = await valyd.auth.getAttributeResult(requestId, { secretKey });
if (result.status === "approved" || result.status === "released") {
  result.attributes;   // { legal_name: "Ada Lovelace", dob: "1990-01-01", country: "GB" }
}
```

Status progression: `pending` → `approved` / `released`, or `denied` / `expired`.

### Install the crypto dependency

The self-custody methods — `generateRequesterKeypair`, `getAttributeResult` with a `secretKey`,
`openSealedPayload` — use `libsodium-wrappers`, which the SDK **lazy-loads and does not bundle**.

```bash
npm i libsodium-wrappers
```

Without it you get `No such module libsodium-wrappers` the first time you decrypt. Nothing else in
the SDK needs it.

### Without the SDK (REST)

```bash
# 1. Create the request (base64 X25519 public key)
curl -X POST https://idp.valyd.id/api/auth/attribute-request \
  -H "Authorization: Bearer $CLIENT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{ "valyd_id": "valyd_...",
        "attributes": ["legal_name","dob","country"],
        "requester_public_key": "<base64 X25519 pubkey>",
        "purpose": "Payroll onboarding" }'
# -> { "data": { "request_id": "...", "status": "pending" } }

# 2. The USER approves in their Valyd app. Then poll:
curl https://idp.valyd.id/api/auth/attribute-request/<request_id>/result \
  -H "Authorization: Bearer $CLIENT_TOKEN"
# -> { "data": { "status": "approved", "sealed_payload": "<base64>" } }
# Open sealed_payload with your X25519 secret key (libsodium sealed box).
```

## Managed custody — no crypto on your side

Hand Valyd the secret key and it opens the box for you; `attributes` comes back as plaintext. **The
trade-off is real: Valyd can read the released values**, so you give up end-to-end privacy. Fine for
testing; prefer self-custody for real personal data.

```js
const { requestId } = await valyd.auth.requestAttributes({
  valydId,
  attributes: ["legal_name", "dob"],
  requesterPublicKey: publicKey,
  managedPrivateKey: secretKey,      // Valyd decrypts
});
const r = await valyd.auth.getAttributeResult(requestId);   // no secretKey needed
r.attributes;   // plaintext, decrypted by Valyd
```

---

## At login — currently disabled

Retained for reference. When re-enabled, you put `attributes` + your X25519 public key on the
authorize URL; the consent screen renders each field as a checkbox; on Authorize the granted fields
are sealed on the user's device to your key and returned with the login as `attr_code`.

```js
const { publicKey, secretKey } = await ValydClient.generateRequesterKeypair();

const url = valyd.auth.getAuthorizationUrl({
  scope: ["openid", "profile"],
  state,
  attributes: ["legal_name", "dob", "is_18_plus"],
  requesterPublicKey: publicKey,
  purpose: "Confirm your identity",
});

// On the callback:
const { code, attrCode } = valyd.auth.parseCallback(callbackUrl);
const { user } = await valyd.auth.exchangeCode(code);
if (attrCode) {
  const result = await valyd.auth.getConsentedAttributes(attrCode, { secretKey });
  result.attributes;   // only what the user kept checked
}
```

REST equivalent for fetching the sealed result:

```bash
curl "https://idp.valyd.id/api/auth/attribute-request/$ATTR_CODE/result?client_id=$VALYD_CLIENT_ID"
# -> { "data": { "status": "released", "custody": "self", "sealed_payload": "<base64>" } }
```

A second consent surface, `credential-share`, releases a specific vault credential and gates the
release with a face scan as the user's consent.

---

## Things to know

- **Consent is remembered per app.** Once approved, the user is not re-prompted on later logins; you
  receive the field inline automatically. They can revoke an app (and its data) from *Connected
  sites* in their Valyd account, which re-asks next login.
- **At login is synchronous; after login is asynchronous.** After login you poll.
- **The user chooses what to share.** You get only the fields left checked. Fields the user has not
  verified (no KYC) are not offerable. If they unchecked something you need, ask again with
  `requestAttributes`.
- **Read it promptly.** The sealed payload is purged about **5 minutes** after approval; `attr_code`
  is one-time and short-lived.
- **Needs a backend.** Keypair generation, the `secretKey`, and opening the box are server-side only.
  A browser-only SPA cannot do self-custody safely.
- **Persist the keypair** for the life of the request. A new keypair cannot open an older payload.
- Keep `secretKey` and `clientSecret` server-side only.

## Related

- Proofs already granted at login: `valyd.auth.getUserInfo(token)`, `getLicenses(token)`,
  `getVerifications(token)`.
- Run a verification check: `valyd.verify.*` — returns a proof, not raw data.
