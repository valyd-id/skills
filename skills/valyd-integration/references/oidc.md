# OpenID Connect (OIDC)

Valyd implements OIDC on top of OAuth 2.0, for enterprise platforms and any OIDC-compatible app.

```text
USE OIDC WHEN                                  USE PLAIN TPSSO/OAUTH WHEN
- Enterprise platforms (Mendix, Salesforce)    - Custom web/mobile app
- Your framework requires discovery            - You only need access tokens for API calls
- You need signed ID tokens (JWT)              - Simpler integration
- You want automatic endpoint discovery
```

The login-session CSRF mechanism is a **TPSSO** concern. A standards-compliant OIDC client library
handles `state` and `nonce` itself against the OIDC endpoints below.

## Endpoints

**Discovery** — note the **non-standard `/api/` prefix**:

```http
GET https://idp.valyd.work/api/.well-known/openid-configuration
```

```json
{
  "issuer": "https://idp.valyd.work",
  "authorization_endpoint": "https://idp.valyd.work/api/auth/oidc/authorize",
  "token_endpoint": "https://idp.valyd.work/api/auth/oidc/token",
  "userinfo_endpoint": "https://idp.valyd.work/api/auth/oidc/userinfo",
  "jwks_uri": "https://idp.valyd.work/api/auth/oidc/jwks.json",
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

### Manual configuration (platforms without discovery)

| Setting | Value |
|---|---|
| Issuer | `https://idp.valyd.work` |
| Authorization | `https://idp.valyd.work/api/auth/oidc/authorize` |
| Token | `https://idp.valyd.work/api/auth/oidc/token` |
| Userinfo | `https://idp.valyd.work/api/auth/oidc/userinfo` |
| JWKS | `https://idp.valyd.work/api/auth/oidc/jwks.json` |
| Auth method | `client_secret_post` / `client_secret_basic` |
| ID token algorithm | `RS256` |

```text
Platform supports auto-discovery
  -> point it at https://idp.valyd.work/api/.well-known/openid-configuration
Platform does not
  -> enter the manual values above
Unsure
  -> curl -s https://idp.valyd.work/api/.well-known/openid-configuration
     HTTP 200 with an "issuer" key means discovery works.
```

## Registering the client

Same portal flow as any project (see `portal-and-accounts.md`): create the project, get
`client_id` + a one-time `client_secret`, register the redirect URI. It must match **exactly** — no
trailing slash, correct protocol. Example: `https://your-app.mendixcloud.com/oidc/callback`.

## Claim mapping

| Platform field | OIDC claim |
|---|---|
| Username | `preferred_username` or `sub` |
| Email | `email` |
| Name | `name` |
| First name | `first_name` |
| Last name | `last_name` |

Sample `GET /api/auth/oidc/userinfo` response:

```json
{
  "sub": "user_12345",
  "preferred_username": "john.doe",
  "email": "john.doe@example.com",
  "email_verified": true,
  "name": "John Doe",
  "first_name": "John",
  "last_name": "Doe",
  "picture": "https://example.com/avatar.jpg"
}
```

## Mendix

**Studio Pro** — set the issuer to `https://idp.valyd.work`; endpoints
`/api/auth/oidc/authorize`, `/api/auth/oidc/token`, `/api/auth/oidc/userinfo`; scopes
`openid profile email`; enter `client_id` + `client_secret`; redirect URI
`https://your-app.mendixcloud.com/oidc/callback`.

**Mendix Cloud Portal** — app → Environment → Security → enable SSO → "OpenID Connect" → issuer
`https://idp.valyd.work` → credentials → save and restart.

**Custom module constants**

```js
const OIDC_ISSUER        = "https://idp.valyd.work";
const OIDC_CLIENT_ID     = "your-client-id";
const OIDC_CLIENT_SECRET = "your-client-secret";
const OIDC_REDIRECT_URI  = "https://your-app.mendixcloud.com/oidc/callback";
const OIDC_SCOPES        = "openid profile email";
```

**Complete Mendix SSO config**

```json
{
  "sso": {
    "enabled": true,
    "provider": "openid_connect",
    "config": {
      "issuer": "https://idp.valyd.work",
      "client_id": "mendix-app-123",
      "client_secret": "your-secret-here",
      "authorization_endpoint": "https://idp.valyd.work/api/auth/oidc/authorize",
      "token_endpoint": "https://idp.valyd.work/api/auth/oidc/token",
      "userinfo_endpoint": "https://idp.valyd.work/api/auth/oidc/userinfo",
      "jwks_uri": "https://idp.valyd.work/api/auth/oidc/jwks.json",
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

## Token expiry (OIDC)

| Token | Expiry |
|---|---|
| ID token | 15 minutes |
| Access token | 1 hour |
| Refresh token | 30 days |

These differ from TPSSO, where the access token is 15 minutes.

## Verification

```bash
curl -s https://idp.valyd.work/api/.well-known/openid-configuration   # 200, issuer = https://idp.valyd.work
curl -s https://idp.valyd.work/api/auth/oidc/jwks.json                # 200, body has a "keys" array
```

Then run a real login and confirm `userinfo` returns `sub`, `preferred_username`, `email`, `name`,
`first_name`, `last_name`.

## Common errors

| Error | Cause | Fix |
|---|---|---|
| Invalid `redirect_uri` | Doesn't match the registration | Match exactly — no trailing slash, right protocol |
| Invalid client credentials | Wrong `client_id`/`client_secret` | Re-check in the portal; regenerate if the secret is lost |
| User mapping failed | Expected claims missing | Include `profile` and `email` in the scopes |
| ID token validation failed | Signature or expiry | Sync server time; confirm the JWKS endpoint is reachable |
| Discovery failed | Can't reach `.well-known` | Check connectivity / firewall to `idp.valyd.work` |

## Security practice

Keep `client_secret` server-side only. HTTPS everywhere. Register exact redirect URIs, never
wildcards, in production. Rotate secrets on a schedule (90 days recommended) — and remember rotation
invalidates the old secret immediately, so deploy first.
