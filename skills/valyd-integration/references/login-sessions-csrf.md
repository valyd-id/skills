# Login sessions — the CSRF mechanism

**The single most common Valyd integration bug.** Read this before writing any callback handler.

## The problem

The classic OAuth CSRF check is: generate a random `state`, send it on `/authorize`, compare what
the IdP echoes back. **This does not work with Valyd TPSSO.** Valyd substitutes its own opaque
session id for `state` on the redirect back, so `sentState === callbackState` is always false. An
integration built that way rejects every legitimate login — and offers no protection either way.

## The solution

`@valyd/sdk` ships a purpose-built mechanism: **login sessions**. Create one before the redirect,
store its `marker` server-side, verify the marker on the callback.

### Step 1 — before redirecting

```ts
const session = await valyd.createLoginSession();
// -> { authorizeState: "...", marker: "v1.<sig>.<payload>" }

res.cookie("valyd_login", session.marker, {
  httpOnly: true,
  sameSite: "lax",
  secure: true,
  maxAge: 10 * 60 * 1000,   // 10 minutes — matches the marker TTL
});

res.redirect(valyd.getAuthorizationUrl({
  state: session.authorizeState,
  scope: ["profile", "verifications"],
}));
```

### Step 2 — on the callback, before `exchangeCode`

```ts
const marker = req.cookies.valyd_login;
const { valid } = await valyd.verifyLoginSession(marker);
if (!valid) {
  // Expired login, missing cookie, or tampered marker.
  return res.status(400).send("Invalid login session");
}
```

```text
Marker present, within its 10-minute TTL, untampered
  -> { valid: true }; proceed to exchangeCode().
Marker expired, cookie missing, or marker tampered with
  -> { valid: false }; HTTP 400 "Invalid login session". Do NOT call exchangeCode().
Unsure
  -> call await valyd.verifyLoginSession(marker) and branch on `valid`. It never throws.
```

## Marker properties

| Property | Detail |
|---|---|
| Format | HMAC-signed string, `v1.<sig>.<payload>` — signed with your client secret on Valyd's side |
| TTL | **10 minutes**; after that `verifyLoginSession` returns `{ valid: false }` |
| Storage | Server only — `httpOnly` cookie, encrypted session, or KV. Never expose to JS. |
| Return value | `{ valid: boolean }` — **never throws** on an invalid marker |
| When to verify | On the callback, **before** `exchangeCode()` |

Store the marker verbatim. Do not re-encode, truncate, or round-trip it through anything lossy.

A full Express example lives in the SDK repo:
`https://github.com/valyd/idp-sdk/blob/HEAD/examples/express-login.ts`

## Verifying it works

- **Happy path** — complete a real login; the callback gets `{ valid: true }` and proceeds.
- **Negative path** — clear the cookie (or change one character of the marker), then hit the
  callback. Expect `{ valid: false }` and HTTP 400. If this passes, your check is not running.

## Common errors

**Using `state` comparison for CSRF.**
Always fails. Replace with the login-session mechanism above.

**`{ valid: false }` for a legitimate user.**
The marker expired (>10 minutes to complete login), the cookie was not returned, or it was altered.
Set `maxAge: 10 * 60 * 1000`, keep it `httpOnly`, confirm the browser sends it back on the callback.

**Marker readable by client-side JavaScript.**
A non-`httpOnly` cookie or `localStorage` undermines the whole protection. Server-side storage only.
