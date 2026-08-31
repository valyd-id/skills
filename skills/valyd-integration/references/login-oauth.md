# Login with Valyd — the OAuth2 / TPSSO flow

Valyd's login product is the OAuth2 **Authorization Code** flow, called **TPSSO** internally. It is a
**server-side** integration: redirect the user, they consent, Valyd redirects back with a one-time
`code`, and your server exchanges it for tokens.

> **The one behavioural difference from standard OAuth2:** Valyd does **not** echo your `state`. The
> `state` on the callback is Valyd's own opaque session id. Never compare it. Use login sessions —
> see `login-sessions-csrf.md`.

## What you need first

| Variable | Where from |
|---|---|
| `VALYD_CLIENT_ID` | Developer Portal → your project → Credentials |
| `VALYD_CLIENT_SECRET` | same; **server-side only**, shown once |
| `VALYD_REDIRECT_URI` | the exact callback URL registered in the portal, **no trailing slash** |

Scopes must also be enabled for the project in the portal before you may request them.

## The six steps

### 1. Build the authorization URL

```text
https://idp.valyd.work/auth?client_id={client_id}&redirect_url={redirect_url}&scope={scopes}
```

| Parameter | Required | Notes |
|---|---|---|
| `client_id` | yes | from the portal |
| `redirect_url` | yes | **spelled `redirect_url` here** — must match the registered value exactly |
| `scope` | yes | space-separated, URL-encoded: `profile%20verifications` |
| `state` | optional | pass `session.authorizeState`; **not echoed back** — never a CSRF check on its own |

Encoded example:

```text
https://idp.valyd.work/auth?client_id=YOUR_CLIENT_ID&redirect_url=https://yourapp.com/callback&scope=profile%20verifications
```

### 2. Issue a login session and redirect

```js
import { ValydClient } from "@valyd/sdk";

const valyd = new ValydClient({
  clientId: process.env.VALYD_CLIENT_ID,
  clientSecret: process.env.VALYD_CLIENT_SECRET,
  redirectUri: process.env.VALYD_REDIRECT_URI,
  // baseUrl defaults to https://idp.valyd.work
});

const session = await valyd.createLoginSession();     // { authorizeState, marker }
res.cookie("valyd_login", session.marker, {
  httpOnly: true,
  sameSite: "lax",
  secure: process.env.NODE_ENV === "production",
  maxAge: 10 * 60 * 1000,                             // matches the 10-min marker TTL
});

res.redirect(valyd.getAuthorizationUrl({
  state: session.authorizeState,
  scope: ["profile", "verifications", "zkp"],
  productName: "My App",                              // shown on the consent screen
}));
```

Expected: HTTP 302 to `https://idp.valyd.work/auth?...`, marker cookie set.

### 3. User consents

Valyd renders the consent screen with exactly the scopes you requested and issues a one-time `code`.

### 4. Receive the callback

```text
https://yourapp.com/callback?code=AUTH_CODE_HERE&state=<Valyd's own session id>
```

| Parameter | Meaning |
|---|---|
| `code` | one-time authorization code, single-use, bound to your client |
| `state` | **Valyd's opaque session id — not your value.** Ignore it. |

The code's advertised lifetime differs between docs pages (2 min in the endpoint reference, 5 min in
the flow guide). **Exchange it the instant your callback fires** and the difference never matters.

### 5. CSRF check — verify the marker, not the state

```js
const { valid } = await valyd.verifyLoginSession(req.cookies.valyd_login);
if (!valid) return res.status(400).send("Invalid login session");
```

### 6. Exchange the code, then fetch the user

```js
app.get("/callback", async (req, res) => {
  const { code, error } = valyd.parseCallback(req.url);
  if (error || !code) return res.status(400).send(error ?? "missing code");

  const { valid } = await valyd.verifyLoginSession(req.cookies.valyd_login);
  if (!valid) return res.status(400).send("Invalid login session");

  const tokens = await valyd.exchangeCode(code);          // server-side only
  const user = await valyd.getUserInfo(tokens.accessToken);

  res.clearCookie("valyd_login");
  // ...set your own app session, then redirect to /dashboard
});
```

## The flow at a glance

| Step | What | SDK method |
|---|---|---|
| 1 | Start login, store marker | `createLoginSession()` |
| 2 | Redirect to Valyd | `getAuthorizationUrl()` |
| 3 | Read callback query | `parseCallback()` |
| 4 | CSRF check | `verifyLoginSession(marker)` |
| 5 | Get tokens | `exchangeCode(code)` |
| 6 | User data | `getUserInfo()`, `getVerifications()`, `getLicenses()` |

## Token exchange without the SDK

`POST https://idp.valyd.work/api/auth/tpsso/token`, JSON body. Note the parameter is spelled
**`redirect_uri`** here (not `redirect_url` as on authorize). Send the same value you used at
authorize — Valyd validates it against the code, and it will become mandatory in a future release.

```http
POST /api/auth/tpsso/token HTTP/1.1
Host: idp.valyd.work
Content-Type: application/json

{
  "grant_type": "authorization_code",
  "client_id": "YOUR_CLIENT_ID",
  "client_secret": "YOUR_CLIENT_SECRET",
  "code": "AUTH_CODE_HERE",
  "redirect_uri": "https://yourapp.com/callback"
}
```

Read the token at **`response.data.access_token`**.

Python (Flask):

```python
response = requests.post(
    "https://idp.valyd.work/api/auth/tpsso/token",
    json={
        "grant_type": "authorization_code",
        "client_id": "YOUR_CLIENT_ID",
        "client_secret": "YOUR_CLIENT_SECRET",
        "code": code,
        "redirect_uri": "https://yourapp.com/callback",
    },
)
tokens = response.json()["data"]
session["access_token"] = tokens["access_token"]
```

PHP:

```php
curl_setopt_array($ch, [
    CURLOPT_URL => 'https://idp.valyd.work/api/auth/tpsso/token',
    CURLOPT_POST => true,
    CURLOPT_POSTFIELDS => json_encode([
        'grant_type' => 'authorization_code',
        'client_id' => 'YOUR_CLIENT_ID',
        'client_secret' => 'YOUR_CLIENT_SECRET',
        'code' => $code
    ]),
    CURLOPT_HTTPHEADER => ['Content-Type: application/json'],
    CURLOPT_RETURNTRANSFER => true,
]);
$data = json_decode(curl_exec($ch), true);
$_SESSION['access_token'] = $data['data']['access_token'];
```

Java (Spring) — same JSON body via `RestTemplate.postForEntity` to the same URL.

## Renewing an access token

Access tokens last **15 minutes** (`expires_in: 900`). Refresh from your **backend**, with client
credentials — a refresh token is a server-side credential.

```json
{
  "refresh_token": "rfrsh_abc123...",
  "client_id": "YOUR_CLIENT_ID",
  "client_secret": "YOUR_CLIENT_SECRET"
}
```

**Rotation is on by default.** Each successful refresh returns a **new** `refresh_token` and revokes
the one you sent — persist the replacement atomically. Replaying a rotated-away token is treated as
theft and **revokes every refresh token for that user and client**; the user simply signs in again.

SDK: `const next = await valyd.auth.refreshToken(stored)` — then store both
`next.accessToken` and `next.refreshToken`. Requires `@valyd/sdk` **1.6.0+**.

The refresh response nests the tokens one level deeper than `/token` does:
`data.tokens.access_token`.

## Decision tree

```text
Request hit your login/start route
  -> createLoginSession(); store session.marker server-side;
     getAuthorizationUrl({ state: session.authorizeState, scope, productName }); redirect.

Request hit your /callback route
  -> const { code, error } = valyd.parseCallback(req.url)
     error set OR no code  -> HTTP 400 (error ?? "missing code"). STOP.
     code present          -> CSRF check.

CSRF check (every callback)
  -> const { valid } = await valyd.verifyLoginSession(req.cookies.valyd_login)
     valid === false -> HTTP 400 "Invalid login session". STOP.
     valid === true  -> token exchange.
  NEVER compare the callback `state` to anything you sent.

Token exchange
  -> const tokens = await valyd.exchangeCode(code)   // server-side, immediately
     code expired -> restart from the login route.
  -> const user = await valyd.getUserInfo(tokens.accessToken)
  -> clear the marker cookie, set your own session.

Not on Node / not using the SDK
  -> POST https://idp.valyd.work/api/auth/tpsso/token with JSON
     { grant_type, client_id, client_secret, code, redirect_uri }
     and read response.data.access_token.
```

## Verification

- The login route returns 302 with `Location: https://idp.valyd.work/auth?client_id=...`.
- The callback receives a `code` query param.
- Token exchange returns 200 with `data.access_token`:

```bash
curl -i -X POST https://idp.valyd.work/api/auth/tpsso/token \
  -H "Content-Type: application/json" \
  -d '{"grant_type":"authorization_code","client_id":"YOUR_CLIENT_ID","client_secret":"YOUR_CLIENT_SECRET","code":"AUTH_CODE_HERE"}'
```

## Common errors

**Comparing the callback `state` — the flow is always rejected.**
Valyd does not echo your `state`. Remove the comparison; use `createLoginSession()` +
`verifyLoginSession(marker)`.

**"missing code" / expired code.**
No `code` on the callback (user denied, or an `error` came back), or the code was exchanged too
late. Return 400 and restart; exchange immediately on the server.

**`redirect_url` mismatch.**
The value sent at authorize does not exactly match the portal registration. Match scheme, host and
path, with no trailing slash.
