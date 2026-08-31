# Statuses & decisioning

Two independent dimensions: the **session status** (lifecycle of a whole verification session) and
the **check status** (result of one check inside it).

## Session status

```text
NOT_STARTED --> IN_PROGRESS --> (IN_REVIEW) --> APPROVED | DECLINED
                                            \-> ABANDONED | EXPIRED
```

`IN_REVIEW` is optional — a session may go straight from `IN_PROGRESS` to a terminal state, or pass
through review first.

| Status | Meaning |
|---|---|
| `NOT_STARTED` | Session created, user not yet on the hosted page |
| `IN_PROGRESS` | User is interacting with the flow |
| `IN_REVIEW` | Awaiting human / async review |
| `APPROVED` | All checks passed (or manually approved) |
| `DECLINED` | Checks failed (or manually declined) |
| `ABANDONED` | User left before completing |
| `EXPIRED` | TTL elapsed before completion |

**Terminal** (a webhook is sent, the lifecycle ends): `APPROVED`, `DECLINED`, `ABANDONED`, `EXPIRED`.
**Non-terminal** (still in flight): `NOT_STARTED`, `IN_PROGRESS`, `IN_REVIEW`.

### How to act

```text
NOT_STARTED  -> do nothing yet; wait for the user to open the hosted page. Keep the session pending.
IN_PROGRESS  -> do nothing yet; the user is mid-flow.
IN_REVIEW    -> do nothing yet; await the outcome. It will move to APPROVED or DECLINED.
                DO NOT grant access.
APPROVED     -> fetch GET /api/v2/session/{id}/decision for the full data, then grant access.
DECLINED     -> fetch the decision to see which checks failed; deny, and offer a retry path
                if your policy allows.
ABANDONED    -> treat as not verified; prompt the user to restart (create a NEW session —
                sessions cannot be resumed).
EXPIRED      -> treat as not verified; the TTL elapsed. Create a new session if still needed.
Unsure       -> curl https://idp.valyd.work/api/v2/session/{id} -H "X-API-Key: $VALYD_API_KEY"
```

## Check status

| Check status | Meaning |
|---|---|
| `pending` | not started |
| `running` | in progress |
| `passed` | check succeeded |
| `failed` | check failed |
| `review` | inconclusive; needs human or async review |

```text
passed            -> satisfied. If all checks pass, the session moves toward APPROVED.
failed            -> drives the session toward DECLINED; inspect the decision for the reason.
review            -> inconclusive; the session typically sits in IN_REVIEW until resolved.
                     Do NOT grant access on this check yet.
pending / running -> not finished.
Unsure            -> curl .../session/{id}/decision -H "X-API-Key: $VALYD_API_KEY"
```

Relationship:

- All checks `passed` → session typically `APPROVED`.
- Any check `failed` → session typically `DECLINED`.
- Any check `review` and none failed → session typically `IN_REVIEW` until resolved.

## Mapping to product behaviour

| `d.status` | What to do |
|---|---|
| `APPROVED` | Grant access. Store the decision against the user. |
| `DECLINED` | Show a clear message. Inspect `d.checks` for which check failed and why. **Do not surface raw error messages or codes to the end user.** |
| `IN_REVIEW` | Show "we'll be in touch". A terminal webhook arrives when review completes. |
| `ABANDONED` / `EXPIRED` | Offer to restart — create a new session. |

## Manual override

`PATCH /api/v2/session/{id}/status` with `{ "status": "APPROVED" }` or `"DECLINED"` forces a terminal
decision — for example when a human reviewer on your side resolves an `IN_REVIEW` session.

- Valid only while the session is **non-terminal** (`NOT_STARTED`, `IN_PROGRESS`, `IN_REVIEW`).
- Once terminal it cannot be flipped: `409 already_decided`.
- Authenticated by the project API key alone — **any holder of that key can override**; there is no
  separate reviewer role.
- The decision is recorded on the session and delivered to your webhook like any other result.

```json
{
  "success": false,
  "error": {
    "code": "already_decided",
    "message": "Session is already in a terminal state.",
    "status": "APPROVED"
  }
}
```

## Build defensively

The versioning policy allows **new enum values to be added at any time** — a new `check.status`, a
new failure `signal`, a new event `type`. Never hard-fail on an unrecognized value: treat it as the
closest known category (an unknown terminal status as "not approved"). Likewise, ignore unknown
response fields rather than rejecting the payload.

Breaking changes never land on `/api/v2` in place; they ship under `/api/v3`. Pin `/api/v2` in your
base URL and you are stable until a deprecation is announced, with a **minimum 6-month** migration
window. Deprecated responses may carry a `Deprecation` header pointing at the replacement.
