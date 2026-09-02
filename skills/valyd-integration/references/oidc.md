# Using your own OIDC library

Connect with Valyd is **standard OpenID Connect**, so any OIDC-capable framework or platform works
— Mendix, Salesforce, Auth0-style modules, `openid-client`, Spring Security, and so on. Point it at
discovery and it configures itself.

If you're writing Node and don't need a specific platform's module, prefer `@valyd/sdk` — see
[`connect-oidc.md`](connect-oidc.md). It handles state, nonce, PKCE and ID-token validation for you.

## Discovery

```http
GET https://idp.valyd.id/api/.well-known/openid-configuration
```

> Note the **`/api/` prefix**. The standard `/.well-known/openid-configuration` path also works,
> but a strict library that builds the path from the issuer may need the URL given explicitly.

```json
{
  "issuer": "https://idp.valyd.id",
  "authorization_endpoint": "https://idp.valyd.id/api/auth/oidc/authorize",
  "token_endpoint": "https://idp.valyd.id/api/auth/oidc/token",
  "userinfo_endpoint": "https://idp.valyd.id/api/auth/oidc/userinfo",
  "jwks_uri": "https://idp.valyd.id/api/auth/oidc/jwks.json",
  "response_types_supported": ["code", "id_token", "token id_token"],
  "grant_types_supported": ["authorization_code", "refresh_token"],
  "subject_types_supported": ["public"],
  "id_token_signing_alg_values_supported": ["RS256"],
  "scopes_supported": ["openid", "profile", "email", "verifications", "zkp"],
  "token_endpoint_auth_methods_supported": ["client_secret_post", "client_secret_basic"],
  "claims_supported": [
    "sub", "iss", "aud", "exp", "iat", "name", "email",
    "preferred_username", "first_name", "last_name"
  ]
}
```

Logout is advertised as `end_session_endpoint` — `/api/auth/oidc/logout`. See [`tokens.md`](tokens.md).

## Manual configuration

For platforms without auto-discovery:

| Setting | Value |
| --- | --- |
| Issuer | `https://idp.valyd.id` |
| Authorization | `https://idp.valyd.id/api/auth/oidc/authorize` |
| Token | `https://idp.valyd.id/api/auth/oidc/token` |
| Userinfo | `https://idp.valyd.id/api/auth/oidc/userinfo` |
| JWKS | `https://idp.valyd.id/api/auth/oidc/jwks.json` |
| Auth method | `client_secret_post` / `client_secret_basic` |
| ID token algorithm | `RS256` |

```text
Platform supports auto-discovery
  -> point it at https://idp.valyd.id/api/.well-known/openid-configuration
Platform does not
  -> enter the manual values above
Unsure
  -> curl -s https://idp.valyd.id/api/.well-known/openid-configuration
     HTTP 200 with an "issuer" key means discovery works.
```

## Registering the client

The same portal flow as any app (see [`portal-and-accounts.md`](portal-and-accounts.md)): create
the app, get `client_id` and a one-time `client_secret`, register your redirect URI. It must match
**exactly** — no trailing slash, correct protocol.

If you use RP-initiated logout, register your `post_logout_redirect_uri` as an **additional
redirect URI**.

## Claim mapping

| Platform field | OIDC claim |
| --- | --- |
| Username | `preferred_username` or `sub` |
| Email | `email` |
| Name | `name` |
| First name | `first_name` |
| Last name | `last_name` |

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

Key the user by **`sub`** — the stable `valyd_…` id. `id_verified` tells you the account passed
identity verification.

## Mendix

**Studio Pro** — issuer `https://idp.valyd.id`; endpoints `/api/auth/oidc/authorize`,
`/api/auth/oidc/token`, `/api/auth/oidc/userinfo`; scopes `openid profile email`; enter
`client_id` + `client_secret`; redirect URI `https://your-app.mendixcloud.com/oidc/callback`.

**Mendix Cloud Portal** — app → Environment → Security → enable SSO → "OpenID Connect" → issuer
`https://idp.valyd.id` → credentials → save and restart.

### Custom module constants

```js
const OIDC_ISSUER        = "https://idp.valyd.id";
const OIDC_CLIENT_ID     = "your-client-id";
const OIDC_CLIENT_SECRET = "your-client-secret";
const OIDC_REDIRECT_URI  = "https://your-app.mendixcloud.com/oidc/callback";
const OIDC_SCOPES        = "openid profile email";
```

### Complete SSO config

```json
{
  "sso": {
    "enabled": true,
    "provider": "openid_connect",
    "config": {
      "issuer": "https://idp.valyd.id",
      "client_id": "mendix-app-123",
      "client_secret": "your-secret-here",
      "authorization_endpoint": "https://idp.valyd.id/api/auth/oidc/authorize",
      "token_endpoint": "https://idp.valyd.id/api/auth/oidc/token",
      "userinfo_endpoint": "https://idp.valyd.id/api/auth/oidc/userinfo",
      "jwks_uri": "https://idp.valyd.id/api/auth/oidc/jwks.json",
      "scopes": ["openid", "email", "profile"],
      "redirect_uri": "https://your-app.mendixcloud.com/oidc/callback",
      "token_endpoint_auth_method": "client_secret_post",
      "id_token_signing_alg": "RS256",
      "user_mapping": {
        "username": "preferred_username",
        "email": "email",
        "name": "name",
        "first_name": "first_name",
        "last_name": "last_name"
      }
    }
  }
}
```

## Verification

```bash
curl -s https://idp.valyd.id/api/.well-known/openid-configuration   # 200, issuer = https://idp.valyd.id
curl -s https://idp.valyd.id/api/auth/oidc/jwks.json                # 200, body has a "keys" array
```

Then run a real login and confirm userinfo returns `sub`, `preferred_username`, `email`, `name`.

## Common errors

| Error | Cause | Fix |
| --- | --- | --- |
| Invalid `redirect_uri` | Doesn't match the registration | Match exactly — no trailing slash, right protocol |
| Invalid client credentials | Wrong id/secret, or wrong environment | Re-check in the portal for that environment |
| User mapping failed | Expected claims missing | Include `profile` and `email` in the scopes |
| ID token validation failed | Signature or expiry | Sync server time; confirm the JWKS endpoint is reachable; **never accept `alg: "none"`** |
| Discovery failed | Can't reach `.well-known` | Check connectivity; try the `/api/` prefixed URL |
| `410 Gone` | You configured a legacy `/api/auth/tpsso/*` endpoint | Use `/api/auth/oidc/*` |

## Security practice

Keep `client_secret` server-side only. HTTPS everywhere. Register exact redirect URIs, never
wildcards, in production. Rotate secrets on a schedule — and remember rotation invalidates the old
secret immediately, so deploy the new one first.
