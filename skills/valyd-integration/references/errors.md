# Error reference

Match on the error `code`, not the message text.

Every response carries an **`X-Request-Id`** header — log it, and quote it when contacting support.
**Never send support your API keys, tokens, or identity data.**

---

## Connect with Valyd (OIDC)

| Code | HTTP | Cause | Fix |
| --- | --- | --- | --- |
| `invalid_client` | 401 | `client_id` / `client_secret` wrong or mismatched | Re-check in the portal. Confirm you're not mixing environments. |
| `invalid_token` | 401 | Access token invalid, malformed, or expired (~15 min) | Refresh, or re-authenticate |
| `insufficient_scope` | 403 | Token lacks the required scope | Add the scope to the authorization request and re-authenticate — **and** enable it for the app in the portal |
| `invalid_grant` | 400 | Code or refresh token invalid, expired, already used; or `redirect_uri` mismatch | Exchange codes immediately; send the same `redirect_uri` at authorize and token |
| `invalid_request` | 400 | Missing or malformed parameter | Check required parameters |
| `access_denied` | 403 | The user declined on the consent screen | Not a bug — prompt again or explain why |
| — | **410** | You called a legacy `/api/auth/tpsso/*` endpoint | Move to `/api/auth/oidc/*` |

### `invalid_grant` in detail

The most common causes, in order:

1. The code was already exchanged — they are **single-use**.
2. More time passed than the code's short lifetime.
3. The `redirect_uri` at the token endpoint differs from the one used at authorize.
4. A **rotated-away refresh token was replayed** — which also revokes the user's entire
   refresh-token family for your client. The user simply signs in again.

### State mismatch on every login

Not an API error — a code bug. Either you are comparing against something other than the value you
stored, or you are using the deprecated `verifyLoginSession()` no-op. See
[`oidc-session-security.md`](oidc-session-security.md).

### ID token validation failures

| Symptom | Cause |
| --- | --- |
| Signature invalid | Wrong JWKS, or an algorithm other than RS256 — **never accept `alg: "none"`** |
| `aud` mismatch | Must equal your `client_id` |
| `nonce` mismatch | Must equal the value you sent at authorize — this is your replay protection |
| Expired | Clock skew; sync server time |

---

## Verification API

| HTTP | Code | When |
| --- | --- | --- |
| 400 | `validation_error` | Malformed body or missing required field |
| 401 | `invalid_api_key` | Missing/invalid `X-API-Key` — or a key from another environment |
| 404 | `not_found` | Session or resource does not exist |
| 409 | `already_decided` | Manual override on an already-terminal session |
| 409 | `idempotency_in_progress` | A request with this `Idempotency-Key` is still in flight |
| 422 | `unprocessable` | Could not process the input |
| 422 | `idempotency_key_reused` | Same `Idempotency-Key`, different request body |
| 429 | `rate_limited` | Rate limit exceeded |
| 500 | `internal_error` | Unexpected server error — safe to retry |

```text
400 validation_error       -> fix the request body, then retry.
401 invalid_api_key        -> set X-API-Key to a valid App API key FOR THIS ENVIRONMENT.
                              Do NOT retry until fixed.
404 not_found              -> verify the session id. Do NOT retry the same id.
409 already_decided        -> the session is terminal; overrides only work before that.
409 idempotency_in_progress-> retry shortly.
422 unprocessable          -> the input was unusable; collect new input and retry.
422 idempotency_key_reused -> you reused a key with a different body; use a fresh key.
429 rate_limited           -> back off and retry later.
500 internal_error         -> safe to retry with backoff.
```

Authenticated endpoints are **billed per call** against your app's wallet — which is exactly why
`idempotencyKey` matters on retries.

### Check-level failures

These arrive on a **successful** HTTP call, as `check.error.code` with `check.status === "failed"`.

Credential (license) declines:

| Code | Meaning |
| --- | --- |
| `license_not_found` | No license matching that number + state + type |
| `license_expired` | Found, past its expiry date |
| `license_inactive` | Exists but suspended, revoked, or lapsed |
| `name_mismatch` | Valid license, registered to a different person |
| `board_unavailable` | The licensing board was unreachable — retry in a few minutes |

Anti-spoof failure `signal` values: `no_face`, `face_unreadable`, `spoof_detected`,
`low_confidence`, `duplicate_frames`, `static_capture`, `discontinuous_motion`, `different_faces`.

**Never surface these codes or raw check data to the end user.** Map them to your own message.

---

## SDK errors (`ValydVerifyError`)

| Code | Meaning | Fix |
| --- | --- | --- |
| `config_error` | Missing `apiKey` / `webhookSecret` | Pass them to the constructor |
| `network_error` | DNS / socket failure | Retry with backoff |
| `timeout` | Exceeded `timeoutMs` (default 15 s) | Raise it; credential registry lookups take 10–60 s |
| `invalid_signature` | Webhook HMAC mismatch or stale timestamp | Use the raw body; check the secret; return 400 |
| `API_KEY_INVALID` | API-side rejection of the key | Rotate / refetch; confirm the environment |
| `VALIDATION_ERROR` | API-side request validation | Fix the payload |

**A method that no longer exists** is not an error code but is the most common SDK surprise — see
the removals table in [`sdk.md`](sdk.md).

---

## The failures that actually happen

1. **A CSRF check that does nothing.** `verifyLoginSession()` is a no-op; it returns and your check
   passes for everyone. Replace with a `state` comparison.
2. **`410 Gone`.** You're on the legacy TPSSO namespace.
3. **Calling a per-check endpoint that isn't public any more.** Run it as a workflow check.
4. **Webhook signature never matches.** The body was parsed and re-serialized. HMAC the raw bytes.
5. **Gating access on `?status=`.** It's a user-editable hint. Read `sessions.decision(id)`.
6. **Mixing environments.** A dev key against a production host, or vice versa. Both 401.
7. **Losing a rotated refresh token.** Persist the new value atomically or the next refresh trips
   theft detection.
