# OIDC session security — state, nonce, PKCE

> **This reversed in August 2026.** Valyd's IdP now **echoes your `state` back unchanged**, and the
> standard OAuth `state` comparison is the correct, required CSRF check. If you have read older
> Valyd material telling you the opposite, it is wrong — see "The deprecated marker pattern" below.

Connect with Valyd uses the standard OIDC Authorization Code flow. Keep one complete login
transaction in the user's **server-side** session:

| Value | What it protects against |
| --- | --- |
| `state` | callback CSRF |
| `nonce` | ID token replay — it is bound into the `id_token` |
| S256 PKCE verifier | code interception — binds the code to this transaction |
| `redirect_uri` | must exactly match the registered callback |

The SDK generates and validates these together.

## Start login

```typescript
app.get("/login", (req, res) => {
  const transaction = valyd.createAuthorizationRequest({
    scope: ["profile", "verifications"],
  });

  req.session.valydOidc = transaction;
  res.redirect(transaction.url);
});
```

Store the transaction **on the server**. Do not put it in local storage, and never expose the PKCE
verifier to browser JavaScript.

## Handle the callback

```typescript
app.get("/auth/valyd/callback", async (req, res) => {
  const transaction = req.session.valydOidc;
  delete req.session.valydOidc;   // consume once

  if (!transaction) return res.status(400).send("Login expired");

  const result = await valyd.handleCallback(req.originalUrl, { transaction });
  req.session.user = result.user;
  res.redirect("/account");
});
```

`handleCallback()` compares state, sends the PKCE verifier, exchanges the one-time code, verifies
the RS256 signature through discovery/JWKS, and validates issuer, audience, time claims, and nonce.

## Doing it by hand

If you are not using the SDK, the CSRF check is a strict equality against what you stored:

```js
if (!transaction || req.query.state !== transaction.state) {
  return res.status(400).send("Invalid state");   // reject BEFORE touching the code
}
```

Then exchange the code, and validate the ID token's `nonce` against `transaction.nonce`. Details in
[`tokens.md`](tokens.md).

## Production checklist

- Use an encrypted, server-side session store such as Redis or your database.
- Make the session cookie `HttpOnly`, `Secure`, and `SameSite=Lax`.
- Expire unused OIDC transactions after a few minutes.
- **Consume each transaction once** and reject callbacks without one.
- Never log authorization codes, tokens, client secrets, or PKCE verifiers.
- Persist each newly rotated refresh token atomically.

---

## The deprecated marker pattern — delete it

Before the OIDC migration, Valyd's TPSSO flow did **not** echo `state`; it substituted its own
opaque session id. The workaround was a "login session" with an HMAC-signed `marker`:

```js
// OBSOLETE — do not write this
const session = await valyd.createLoginSession();
res.cookie("valyd_login", session.marker, { httpOnly: true });
// ...
const { valid } = await valyd.verifyLoginSession(marker);
if (!valid) return res.status(400).send("Invalid login session");
```

**`createLoginSession()` and `verifyLoginSession()` are now deprecated no-ops**, kept only for
backward compatibility. They still compile and still return, so a CSRF check built on them
**silently passes for everyone**. That is a security bug, not a stale API.

If you find this pattern in a codebase:

1. Delete the `createLoginSession()` call and the marker cookie.
2. Replace it with `createAuthorizationRequest()` stored in the server-side session.
3. Delete the `verifyLoginSession()` check and replace it with `handleCallback({ transaction })`,
   or an explicit `state` comparison.
4. Check nothing else reads the `valyd_login` cookie.

## Related

- [`connect-oidc.md`](connect-oidc.md) — the whole sign-in flow
- [`tokens.md`](tokens.md) — validating the ID token, and what `nonce` is checked against
- [`gotchas-and-doc-conflicts.md`](gotchas-and-doc-conflicts.md) — other stale patterns
