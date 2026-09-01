# Account API — reading a connected user

> **Auth:** the user's Bearer `valyd_access_token` · **Free** — reads never cost a check ·
> **Proofs and public data only, never PII**

Once the user has [connected](connect-oidc.md), three reads cover everything their account holds.
**The Account API never runs a check** — it reads what previous checks already proved.

Base: `https://idp.valyd.work/api/auth/oidc`

| Endpoint | One call gets you |
| --- | --- |
| `GET /userinfo` | Who they are — legal name, username, country, `id_verified` |
| `GET /licenses` | Professional licenses already verified on their account |
| `GET /verifications` | Every proof and badge — KYC done, age bands, license badges |

## Read before you verify

This is the habit that makes Valyd cheap and fast:

```typescript
const proofs = await valyd.auth.getVerifications(accessToken);
if (proofs.id_verified) {
  // KYC already done — you're finished. No check, no cost, no PII stored.
}
```

The user verified once — maybe in another app entirely. You check the status, you don't re-run it,
and you never store personal data. Only when something is missing do you
[run a workflow](verification-sessions.md).

Reads are only half of it: with the same token you can also **re-prove who they are, right now** —
match their face before a sensitive action, confirm their location, or re-check a live license.

---

## GET /userinfo

**Scope:** `profile` · `Authorization: Bearer YOUR_ACCESS_TOKEN`

Standard OIDC userinfo — returns **top-level claims**, no envelope.

```bash
curl -X GET "https://idp.valyd.work/api/auth/oidc/userinfo" \
  -H "Accept: application/json" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

```json
{
  "sub": "valyd_225c7f2ac450496f97bbbc57354a5898",
  "valyd_id": "valyd_225c7f2ac450496f97bbbc57354a5898",
  "preferred_username": "johndoe",
  "email": "user@example.com",
  "name": "John Doe",
  "id_verified": true
}
```

Use `sub` (identical to `valyd_id`) as your primary key.

---

## GET /licenses

**Auth:** Bearer access token. No specific scope is declared on this endpoint in the source; if you
get a 403, request `doctor_license`.

Returns a snapshot of professional licenses as verified by Valyd — nursing licenses, CDL
endorsements, CPR/BLS certifications, Food Handler permits, and more.

```json
{
  "success": true,
  "data": {
    "licenses": [
      {
        "type": "nurse_licenses",
        "number": "RN-123456",
        "status": "Active",
        "expires_on": "2027-06-30",
        "issuer": "CA Board of Nursing"
      },
      {
        "type": "cpr_certification",
        "number": "CPR-998877",
        "status": "Active",
        "expires_on": "2026-05-15",
        "issuer": "American Heart Association"
      }
    ]
  }
}
```

Note the `{ success, data }` envelope here — unlike `/userinfo`.

---

## GET /verifications

**Scope:** `verifications` · Bearer access token

The user's verification status: whether they passed a human (liveness) check, whether they
completed KYC, and any professional licenses linked to their identity.

| Field (`data.verifications`) | Type | Description |
| --- | --- | --- |
| `human_verified` | boolean | Passed a liveness / anti-spoof check. Falls back to `id_verified` when no explicit human check is on file. |
| `id_verified` | boolean | Completed KYC document verification |
| `licenses` | array | Empty when none |
| `licenses[].license_type` | string | e.g. `drivers_license`, `medical` |
| `licenses[].verified` | boolean | currently verified |
| `licenses[].verified_from` | string \| null | source verified against |
| `licenses[].expire_at` | string \| null | ISO-8601, or `null` if it does not expire |

```json
{
  "success": true,
  "data": {
    "verifications": {
      "human_verified": true,
      "id_verified": true,
      "licenses": [
        {
          "license_type": "drivers_license",
          "verified": true,
          "verified_from": "kyc",
          "expire_at": "2027-03-01T00:00:00+00:00"
        }
      ]
    }
  }
}
```

---

## What you will never get here

**Raw identity data** — date of birth, document number, address, document images — is never
returned by these endpoints. It requires the user's explicit approval through the
[consent flow](consent-attributes.md), and comes back end-to-end encrypted.

The account's `identity` object carries a `verified_at` timestamp so you can judge freshness, and a
license badge carries the registry's own `status` and `expires_at`. Re-run a check when your policy
needs a fresher answer.

## General notes

- Every response carries an **`X-Request-Id`** header. Log it; quote it to support. Never send
  support your API keys, tokens, or identity data.
- HTTPS only.
- SDK helpers: `valyd.auth.getUserInfo(token)`, `getVerifications(token)`, `getLicenses(token)`.

## Related

- [`scopes.md`](scopes.md) — which scope unlocks which endpoint
- [`verification-sessions.md`](verification-sessions.md) — running a check for what's missing
- [`consent-attributes.md`](consent-attributes.md) — getting raw attributes with consent
