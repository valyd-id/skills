# Connect with Valyd — the OIDC sign-in

**Connect with Valyd** establishes the user's reusable Valyd identity in your app — the first step
of [Reusable Verification](products.md). It is built on **standard OpenID Connect**, so it can also
serve as your app's sign-in, like "Sign in with Google".

The user connects, and you get their profile, licenses, and verification proofs. You never see
their documents — you read the **answers**: `id_verified: true`, verified licenses, age bands.

> **Your OIDC provider is `https://idp.valyd.id`**, discovery at
> `/api/.well-known/openid-configuration`. The browser only ever carries a one-time `code`; your
> backend exchanges it with your `client_secret`, so no token touches the front end.

## What you need

| Variable | Where from |
| --- | --- |
| `VALYD_CLIENT_ID` | Developer Portal → your app → Credentials |
| `VALYD_CLIENT_SECRET` | same; **server-side only**, shown once |
| `VALYD_REDIRECT_URI` | the exact callback URL registered in the portal, **no trailing slash** |

Scopes must also be enabled for the app in the portal before you may request them.

---

## The fastest path: the drop-in button

```html
<script src="https://idp.valyd.id/signin/client.js" async></script>
<div class="valyd-signin"
     data-client-id="YOUR_CLIENT_ID"
     data-redirect-uri="https://yourapp.com/auth/valyd/callback"
     data-scope="profile verifications"></div>
```

That is the whole front end. The button sets `valyd_oidc_state` and `valyd_oidc_nonce` cookies,
which your callback reads:

```typescript
import express from "express";
import cookieParser from "cookie-parser";
import { ValydClient } from "@valyd/sdk";   // npm i @valyd/sdk cookie-parser

const app = express();
app.use(cookieParser());
const valyd = new ValydClient({ clientId, clientSecret, redirectUri });

app.get("/auth/valyd/callback", async (req, res) => {
  const { user } = await valyd.handleCallback(req.url, {
    expectedState: req.cookies.valyd_oidc_state,   // the button set this cookie
    nonce: req.cookies.valyd_oidc_nonce,
  });
  // user.valyd_id — stable ID;  user.id_verified — identity proof
  res.redirect("/dashboard");
});
```

## The full server-side flow

When you control both ends, hold the whole transaction server-side instead of in cookies — see
[`oidc-session-security.md`](oidc-session-security.md).

```typescript
app.get("/login", (req, res) => {
  const transaction = valyd.auth.createAuthorizationRequest({
    scope: ["profile", "verifications"],
  });
  req.session.valydOidc = transaction;   // state + nonce + S256 PKCE verifier
  res.redirect(transaction.url);
});

app.get("/auth/valyd/callback", async (req, res) => {
  const transaction = req.session.valydOidc;
  delete req.session.valydOidc;          // consume once
  if (!transaction) return res.status(400).send("Login expired");

  const result = await valyd.auth.handleCallback(req.originalUrl, { transaction });
  req.session.user = result.user;
  res.redirect("/account");
});
```

## The flow, step by step

1. **Start.** Your login route generates a random `state` + `nonce` and an S256 PKCE pair, stores
   them server-side, and redirects to `https://idp.valyd.id/api/auth/oidc/authorize` with
   `client_id`, `redirect_uri`, `response_type=code`, `scope` (**must include `openid`**), `state`
   and `nonce`. The SDK does all of this in `createAuthorizationRequest()`.
2. **User authenticates and consents.** Valyd shows the consent screen with the requested scopes
   and, on approval, issues a one-time `code`.
3. **Callback.** Valyd redirects to your registered `redirect_uri` with `?code=…&state=…`. **The
   `state` is echoed back unchanged.**
4. **CSRF check.** Compare the callback `state` strictly against the stored value. Reject with 400
   on any mismatch, **before touching the code**.
5. **Exchange the code (server-side).** `POST https://idp.valyd.id/api/auth/oidc/token` with
   `grant_type: "authorization_code"`, your client credentials, the `code`, the **same**
   `redirect_uri`, and `code_verifier` when PKCE was used.
6. **Validate the ID token.** Verify the RS256 signature against the JWKS, and check `iss`, `aud`
   (= your `client_id`), `exp`, and that `nonce` equals what you sent.
7. **Fetch the user.** `GET /api/auth/oidc/userinfo` with the Bearer access token.

`handleCallback()` does steps 4–7 in one call.

## The token response

Standard, top-level, **no `data` wrapper**:

```json
{
  "access_token": "eyJhbGciOi...",
  "refresh_token": "rfrsh_abc123...",
  "id_token": "eyJhbGciOiJSUzI1NiIs...",
  "token_type": "Bearer",
  "expires_in": 900,
  "scope": "openid profile verifications"
}
```

What each token is for, and how to validate the ID token: [`tokens.md`](tokens.md).

## Raw HTTP, without the SDK

```http
POST /api/auth/oidc/token HTTP/1.1
Host: idp.valyd.id
Content-Type: application/json

{
  "grant_type": "authorization_code",
  "client_id": "YOUR_CLIENT_ID",
  "client_secret": "YOUR_CLIENT_SECRET",
  "code": "AUTH_CODE_HERE",
  "redirect_uri": "https://yourapp.com/callback",
  "code_verifier": "THE_PKCE_VERIFIER"
}
```

Client authentication is `client_secret_post` (body) or `client_secret_basic` (HTTP Basic).

If you would rather use a standard OIDC library than the SDK, point it at discovery — see
[`oidc.md`](oidc.md).

## Refreshing

```json
{
  "grant_type": "refresh_token",
  "client_id": "YOUR_CLIENT_ID",
  "client_secret": "YOUR_CLIENT_SECRET",
  "refresh_token": "rfrsh_abc123..."
}
```

SDK: `const next = await valyd.auth.refreshToken(stored)` — then persist **both**
`next.accessToken` and `next.refreshToken`. Rotation is on for every refresh; see
[`tokens.md`](tokens.md).

## Security rules

- Keep `client_secret`, tokens, and the OIDC transaction on your backend.
- Register an exact HTTPS redirect URI — scheme, host and path must match, no trailing slash.
- **Never build your own state, PKCE, nonce, or JWT validation when the SDK can do it.**
- Codes are single-use, short-lived and client-bound — exchange immediately; a replay returns
  `invalid_grant`.
- A verification API key is not an OIDC token and must never be placed in a browser.

## Common errors

**`invalid_grant` on the token exchange.** The code was already used, expired, or the
`redirect_uri` doesn't match the one used at authorize. Exchange immediately; send the same
`redirect_uri` in both requests.

**State mismatch on every login.** You are comparing against a value that isn't the one you sent —
or you are using the deprecated `verifyLoginSession()` no-op. See
[`oidc-session-security.md`](oidc-session-security.md).

**Authorization fails before the consent screen renders.** A requested scope is not enabled for
your app in the portal.

**`410 Gone`.** You are calling a legacy `/api/auth/tpsso/*` endpoint. Move to
`/api/auth/oidc/*`.
