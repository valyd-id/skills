# Error reference

Every error, both products, in the shared envelope:

```json
{
  "success": false,
  "error": { "code": "<error_code>", "message": "<human-readable message>" }
}
```

Match on `error.code`, not on the message text.

---

## Login with Valyd (TPSSO / OIDC)

```text
invalid_client      -> 401, credentials wrong
invalid_token       -> 401, access token bad/expired
insufficient_scope  -> 403, token lacks a scope
invalid_grant       -> 400, code or refresh token bad/expired/reused, or redirect_uri mismatch
invalid_request     -> 400, missing or malformed parameter
access_denied       -> 403, the user declined on the consent screen
verifyLoginSession returns { valid: false }  -> SDK/app-level, no HTTP code
```

### `invalid_client` — 401

The `client_id` or `client_secret` is invalid or doesn't match. Verify in the Developer Portal →
your project → Credentials. Also returned by `POST /refresh` when you send only a `refresh_token`
without client credentials.

```json
{ "success": false, "error": { "code": "invalid_client", "message": "client_id/client_secret invalid" } }
```

### `invalid_token` — 401

The access token is invalid, malformed, or expired (TPSSO access tokens last 15 minutes). Use
`/refresh`, or re-authenticate the user.

### `insufficient_scope` — 403

The token lacks the required scope. Add it to the authorization URL and re-authenticate — **and
make sure the scope is enabled for your project in the portal**, or authorization will fail before
the consent screen renders.

### `invalid_grant` — 400

The authorization code or refresh token is invalid, expired, already used, or the `redirect_uri`
doesn't match the registered value. Authorization codes are single-use and short-lived — exchange
them the instant the callback fires. For a refresh token, note that **rotation revokes the previous
token**, and replaying a rotated-away token revokes every refresh token for that user and client.

### `invalid_request` — 400

Missing required parameters or invalid values. e.g. `"Missing required parameter: code"`.

### `access_denied` — 403

The user declined on the consent screen. Not a bug — prompt them again or explain why the
permissions are needed.

### Invalid login session — SDK / app level

`valyd.verifyLoginSession(marker)` returned `{ valid: false }`. It never throws. Causes:

- the login session expired (10-minute TTL)
- the marker cookie is missing
- the marker was tampered with
- you are verifying with a different `clientSecret` than the one that created the session

Fix: send the user back through `/login` for a fresh session. Store the marker server-side
(httpOnly cookie or session) so it survives the redirect round-trip. **Do not** fall back to
comparing the callback `state` — that is Valyd's session id, not your CSRF token.

### OIDC-specific

| Error | Cause | Fix |
|---|---|---|
| Invalid `redirect_uri` | Doesn't match the registration | Match exactly — no trailing slash, correct protocol |
| Invalid client credentials | Wrong id/secret | Re-check in the portal; regenerate if lost |
| User mapping failed | Expected claims missing | Include `profile` and `email` in the scopes |
| ID token validation failed | Signature or expiry | Sync server time; confirm the JWKS endpoint is reachable |
| Discovery endpoint failed | Can't reach `.well-known` | Check connectivity/firewall to `idp.valyd.work` |

---

## Verification APIs

| HTTP | Code | When |
|---|---|---|
| 400 | `validation_error` | Malformed body or missing required field |
| 401 | `invalid_api_key` | Missing/invalid `X-API-Key` header |
| 404 | `not_found` | Session or resource does not exist |
| 409 | `already_decided` | Manual override on a session that is already terminal |
| 409 | `idempotency_in_progress` | A request with this `Idempotency-Key` is still in flight |
| 422 | `unprocessable` | Could not process the input, e.g. an unreadable image |
| 422 | `idempotency_key_reused` | Same `Idempotency-Key`, different request body |
| 429 | `rate_limited` | Public/demo rate limit exceeded |
| 500 | `internal_error` | Unexpected server error — safe to retry |

```text
400 validation_error  -> fix the request body / add the missing field, then retry.
401 invalid_api_key   -> set X-API-Key to a valid App API key. Do NOT retry until fixed.
404 not_found         -> verify the session/resource id. Do NOT retry the same id.
409 already_decided   -> the session is terminal; overrides only work before that.
409 idempotency_in_progress -> retry shortly.
422 unprocessable     -> input unusable (e.g. blurry image); collect new input and retry.
422 idempotency_key_reused  -> you reused a key with a different body; use a fresh key.
429 rate_limited      -> back off and retry later.
500 internal_error    -> safe to retry with backoff.
```

Authenticated endpoints are **billed per call** against your App. Public/demo endpoints return 429
when limits are exceeded.

### Credential (license) decline codes

These arrive on a **successful** HTTP call, as `check.error.code` with `check.status === "failed"`:

| Code | Meaning |
|---|---|
| `license_not_found` | No license matching that number + state + type |
| `license_expired` | Found, but past its expiry date |
| `license_inactive` | Exists but suspended, revoked, or lapsed |
| `name_mismatch` | Valid license, registered to a different person |
| `board_unavailable` | The licensing board's system was unreachable — retry in a few minutes |

**Never surface these codes or raw check data to the end user.** Map them to your own message.

---

## SDK errors (`ValydVerifyError`)

| Code | Meaning | Fix |
|---|---|---|
| `config_error` | Missing `apiKey` / `webhookSecret` | Pass them to the constructor |
| `network_error` | DNS / socket failure | Retry with backoff |
| `timeout` | Exceeded `timeoutMs` (default 15 s) | `timeoutMs: 90_000` for credential lookups |
| `invalid_signature` | Webhook HMAC mismatch or stale timestamp | Use the raw body; check the secret; return 400 |
| `API_KEY_INVALID` | API-side rejection of the key | Rotate / refetch the key |
| `VALIDATION_ERROR` | API-side request validation | Fix the payload |

---

## The five most common integration failures

1. **Comparing the callback `state` for CSRF.** Always fails — Valyd doesn't echo it. Use
   `verifyLoginSession(marker)`.
2. **Webhook signature never matches.** The body was parsed and re-serialized. HMAC the raw bytes.
3. **Credential lookup times out.** Default client timeout is shorter than the 10–60 s the registry
   takes. Set ≥90 s.
4. **`redirect_url` mismatch.** Usually a trailing slash. Match the portal registration character
   for character.
5. **Gating access on `?status=` from the redirect.** It's a user-editable hint. Read
   `sessions.decision(id)`.
