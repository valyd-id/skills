# Tokens

Every successful [Authorization Code](connect-oidc.md) exchange returns **three tokens** in one
top-level JSON from `POST /api/auth/oidc/token`. Each has exactly one job — **most integration bugs
come from using one token for another's job.**

| Token | Lifetime | Job |
| --- | --- | --- |
| Access token | ~15 minutes (`expires_in` ≈ 900) | Call Valyd resource APIs as the user |
| ID token | validated once at login (`exp` ≈ 15 min) | Prove *who* logged in, to *your* backend |
| Refresh token | 30 days, **rotates on every refresh** | Mint new access tokens without the user |

## Login sessions

Together these three tokens **are** a login session — there is no separate session object to
create. Your backend keeps the access token (to call APIs) and the rotating refresh token (to renew
quietly), usually mirrored by your own app session cookie. The login lasts as long as you keep
refreshing — up to 30 days per rotating refresh token — and ends at logout, on refresh-token theft
detection, or after 30 days of silence.

> **One word, two things.** A *login session* (this page) is unrelated to a *verification session*
> (one person's run through a check, ending in a decision). "Session expired" from a resource API
> means **refresh the access token**; `EXPIRED` from the decision API means **create a new
> [verification session](verification-sessions.md)**. A dead login session never invalidates a
> verification result, and a finished verification session never logs anyone in.

---

## Access token

Sent as `Authorization: Bearer …` to `/userinfo`, `/licenses`, `/verifications`, and to the
Verification API as `valydAccessToken` for account-connected sessions. It is **scope-gated**: it can
only reach what the user approved on the consent screen.

```json
{
  "iss": "https://idp.valyd.id",
  "sub": "valyd_f895da61d5174b81b8dd6a4e3b417339",
  "aud": "YOUR_CLIENT_ID",
  "iat": 1755600000,
  "exp": 1755600900,
  "scope": "openid profile verifications"
}
```

**Use it for:** calling Valyd APIs on the user's behalf; attaching to a verification session so the
proof saves to their account.

**Never use it for:** identifying the user in your app — that's the ID token's job — or storing
long-term; it dies in ~15 minutes.

> **Treat it as opaque.** Its internal format is Valyd's to change. Don't parse it, don't build
> logic on its claims — pass it in the `Authorization` header and let the API validate it.

---

## ID token

An **RS256-signed JWT** — the login receipt. Your backend validates it once at login and uses its
claims to create your own session.

```json
{
  "iss": "https://idp.valyd.id",
  "sub": "valyd_f895da61d5174b81b8dd6a4e3b417339",
  "aud": "YOUR_CLIENT_ID",
  "iat": 1755600000,
  "exp": 1755600900,
  "nonce": "RANDOM_NONCE_FROM_AUTHORIZE",
  "name": "John Doe",
  "preferred_username": "john.doe",
  "id_verified": true
}
```

Claim notes:

- `sub` is the stable `valyd_…` id — **use it as your primary key**.
- `aud` must equal your `client_id`.
- `nonce` must equal the value you sent on `/authorize` (replay protection).
- `id_verified` tells you the account passed identity verification.

**Use it for:** establishing the login on your backend, keying the user by `sub`, and later as the
`id_token_hint` on logout.

**Never send it to an API.** It is not an access credential — Valyd endpoints reject it, and an ID
token accepted as an API credential anywhere is a security bug. It also never belongs in a URL or
in browser storage.

> **Always validate before trusting**: signature (RS256/JWKS), `iss`, `aud`, `exp`, `nonce`. An
> unvalidated ID token is just attacker-writable JSON.

---

## Refresh token

An opaque string (`rfrsh_…`) — **not a JWT, nothing to decode** — held only on your backend.

**Use it for:** minting a new access token at the token endpoint with
`grant_type: "refresh_token"`, from your backend, with your client credentials.

**Never use it for:** calling APIs, or anywhere client-side. It is the longest-lived credential in
the system — treat it like a password.

> **Rotation is on.** Every refresh revokes the token you sent and returns a new one — persist the
> new value every time, atomically. Replaying a rotated-away token is treated as **theft** and
> revokes the user's entire refresh-token family for your client.

---

## Validating tokens

**Let a library do it.** The SDK's `handleCallback()` / `exchangeCode()` verify the ID token's
RS256 signature against discovery/JWKS plus issuer, audience, expiry and nonce before returning.
Any standard OIDC library pointed at
`https://idp.valyd.id/api/.well-known/openid-configuration` does the same.

**Manually:** fetch the signing keys from `https://idp.valyd.id/api/auth/oidc/jwks.json`, verify
the RS256 signature, then check `iss === "https://idp.valyd.id"`, `aud === your client_id`, `exp`
in the future, and `nonce === the value you sent`. **Never accept `alg: "none"`** or an unexpected
algorithm.

---

## Logout

`GET https://idp.valyd.id/api/auth/oidc/logout` — RP-initiated logout, advertised in discovery as
`end_session_endpoint`.

| Parameter | Notes |
| --- | --- |
| `id_token_hint` | the `id_token` from login. An **expired one is accepted** — its signature still proves the user/client. |
| `post_logout_redirect_uri` | must **exactly match** one of your registered redirect URIs — register your post-logout URL as an additional redirect URI |
| `state` | optional, echoed back |

Revokes the user's refresh **and** access tokens for your client, then redirects.

## Related

- [`connect-oidc.md`](connect-oidc.md) — where tokens come from
- [`account-api.md`](account-api.md) — what the access token can read
- [`oidc-session-security.md`](oidc-session-security.md) — the transaction that produces the nonce
