# TPSSO API reference

Base URL for every endpoint here: **`https://idp.valyd.work/api/auth/tpsso`**

- HTTPS only.
- Authenticated endpoints take `Authorization: Bearer YOUR_ACCESS_TOKEN`.
- `/token` and `/refresh` authenticate with `client_id` + `client_secret` **in the JSON body**.
- Prefer the SDK helpers where you can — they call these for you.

Five endpoints: `POST /token`, `POST /refresh`, `GET /userinfo`, `GET /licenses`,
`GET /verifications`.

---

## SDK methods for login sessions (v0.2.0+)

### `valyd.createLoginSession()`

Issues a one-time login session. Call **before** redirecting. Returns `{ authorizeState, marker }`.

- `authorizeState` — pass as `state` to `getAuthorizationUrl()`.
- `marker` — HMAC-signed. Store server-side (httpOnly cookie or session). Never expose to browser JS.
- TTL — 10 minutes.

### `valyd.verifyLoginSession(marker)`

Validates the marker on the callback, **before** `exchangeCode`. Returns `{ valid: boolean }`; never
throws. This is your CSRF check — do not compare the callback `state`.

---

## POST /token — exchange code for tokens

`https://idp.valyd.work/api/auth/tpsso/token` · auth: `client_id` + `client_secret` in the body ·
headers: `Content-Type: application/json`, `Accept: application/json`

Authorization codes are **bound to the client** they were issued to, **single-use**, and expire
quickly (the endpoint reference says ~2 minutes; the flow guide says 5). Exchange immediately.

| Name | Type | Required | Description |
|---|---|---|---|
| `grant_type` | string | yes | `"authorization_code"` |
| `client_id` | string | yes | from the portal |
| `client_secret` | string | yes | **server-side only** |
| `code` | string | yes | the code from your callback |
| `redirect_uri` | string | recommended | the **exact** value used at authorize. Validated when supplied; will become required. |

```bash
curl -X POST "https://idp.valyd.work/api/auth/tpsso/token" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{
    "grant_type": "authorization_code",
    "client_id": "YOUR_CLIENT_ID",
    "client_secret": "YOUR_CLIENT_SECRET",
    "code": "AUTH_CODE_FROM_CALLBACK"
  }'
```

```js
const response = await fetch("https://idp.valyd.work/api/auth/tpsso/token", {
  method: "POST",
  headers: { "Content-Type": "application/json", "Accept": "application/json" },
  body: JSON.stringify({
    grant_type: "authorization_code",
    client_id: "YOUR_CLIENT_ID",
    client_secret: "YOUR_CLIENT_SECRET",
    code: authCode,
  }),
});
const data = await response.json();
const { access_token, refresh_token } = data.data;   // note: data.data
```

**200 OK**

```json
{
  "success": true,
  "data": {
    "access_token": "eyJhbGciOi...",
    "token_type": "Bearer",
    "expires_in": 900,
    "refresh_token": "rfrsh_abc123...",
    "user": {
      "id": 123,
      "email": "user@example.com",
      "username": "john_doe",
      "name": "John Doe",
      "valyd_id": "valyd_225c7f2ac450496f97bbbc57354a5898",
      "avatar_url": null,
      "created_at": "2025-09-11T10:15:00Z"
    }
  }
}
```

Error: `{ "success": false, "error": { "code": "invalid_client", "message": "client_id/client_secret invalid" } }`

---

## POST /refresh — refresh the access token

`https://idp.valyd.work/api/auth/tpsso/refresh` · auth: `client_id` + `client_secret` (body, or HTTP
Basic)

> **Breaking change.** This endpoint now requires client credentials, and the refresh token is
> validated against the client it was issued to. Sending only a `refresh_token` is rejected with
> `401 invalid_client`. Refresh tokens are server-side credentials — never refresh from a browser or
> mobile app.

| Name | Type | Required | Description |
|---|---|---|---|
| `refresh_token` | string | yes | your current refresh token |
| `client_id` | string | yes | |
| `client_secret` | string | yes | server-side only |
| `rotate_refresh` | boolean | no | **defaults to `true`**; set `false` only if you cannot store the replacement |

### Rotation and reuse detection

Rotation is **on by default**. Each successful refresh returns a **new** `refresh_token` and
**immediately revokes the one you presented** — always persist the new one and discard the old.

Presenting an already-rotated token is treated as a stolen credential: Valyd **revokes every refresh
token for that user and client**. Both the attacker's copy and the legitimate session stop working,
and the user signs in again. In practice this only fires if you keep a stale token, so store the
replacement atomically.

```js
const data = await (await fetch("https://idp.valyd.work/api/auth/tpsso/refresh", {
  method: "POST",
  headers: { "Content-Type": "application/json", "Accept": "application/json" },
  body: JSON.stringify({
    refresh_token: refreshToken,
    client_id: process.env.VALYD_CLIENT_ID,
    client_secret: process.env.VALYD_CLIENT_SECRET,
  }),
})).json();

const { access_token, refresh_token } = data.data.tokens;   // note the extra `tokens` level
await saveTokens({ access_token, refresh_token });
```

**200 OK** — note the tokens are nested one level deeper than on `/token`:

```json
{
  "success": true,
  "data": {
    "tokens": {
      "access_token": "eyJhbGciOi...",
      "token_type": "Bearer",
      "expires_in": 900,
      "refresh_token": "rfrsh_new456..."
    }
  }
}
```

Error: `invalid_grant` — "refresh_token is invalid or expired".

---

## GET /userinfo — user profile

`https://idp.valyd.work/api/auth/tpsso/userinfo` · **scope `profile`** · `Authorization: Bearer <access_token>`

```bash
curl -X GET "https://idp.valyd.work/api/auth/tpsso/userinfo" \
  -H "Accept: application/json" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

```json
{
  "success": true,
  "data": {
    "sub": "valyd_225c7f2ac450496f97bbbc57354a5898",
    "email": "user@example.com",
    "first_name": "John",
    "last_name": "Doe",
    "full_name": "John Doe",
    "valyd_id": "valyd_225c7f2ac450496f97bbbc57354a5898",
    "id_verified": true,
    "created_at": "2025-09-10T12:00:00Z"
  }
}
```

---

## GET /licenses — professional licenses

`https://idp.valyd.work/api/auth/tpsso/licenses` · Bearer access token (no specific scope declared on
this endpoint) · returns nursing licenses, CDL endorsements, CPR/BLS certifications, Food Handler
permits and more.

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

---

## GET /verifications — identity verification status

`https://idp.valyd.work/api/auth/tpsso/verifications` · **scope `verifications`** · Bearer access token

Use alongside `/userinfo` for a complete picture.

| Field (`data.verifications`) | Type | Description |
|---|---|---|
| `human_verified` | boolean | Passed a liveness / anti-spoof check. Falls back to `id_verified` when no explicit human check exists. |
| `id_verified` | boolean | Completed KYC document verification |
| `licenses` | array | Empty when none |
| `licenses[].license_type` | string | e.g. `drivers_license`, `medical` |
| `licenses[].verified` | boolean | currently verified |
| `licenses[].verified_from` | string \| null | source verified against |
| `licenses[].expire_at` | string \| null | ISO-8601, or `null` if it does not expire |

```js
const { human_verified, id_verified, licenses } = (await res.json()).data.verifications;
if (human_verified && id_verified) { /* verified human with completed KYC */ }
```

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
