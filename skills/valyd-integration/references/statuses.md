# Session lifecycle, statuses & decisions

A verification session is one person's single run through a [workflow](workflows.md)'s checks on
Valyd's verification page. It moves through a small state machine that **always ends in a terminal
decision**.

## The lifecycle

```text
[*] -> NOT_STARTED          create session
NOT_STARTED -> IN_PROGRESS  user opens the verification page
IN_PROGRESS -> IN_REVIEW    needs human / async review
IN_PROGRESS -> APPROVED | DECLINED | ABANDONED | EXPIRED
IN_REVIEW   -> APPROVED | DECLINED
```

1. **Created (`NOT_STARTED`).** Your backend calls `verify.sessions.create` with a `workflowId`.
   The response carries the verification page `url`, a `sessionToken`, and `expiresAt`. Tag it with
   `vendorData` and bound its lifetime with `ttlSeconds`.
2. **In progress (`IN_PROGRESS` → optionally `IN_REVIEW`).** The user completes the workflow's
   checks. A session needing manual or async review passes through `IN_REVIEW` first; otherwise it
   goes straight to a terminal state.
3. **Terminal (`APPROVED` · `DECLINED` · `ABANDONED` · `EXPIRED`).** The lifecycle ends. **Terminal
   is terminal** — an abandoned or expired session is never resumed; create a new one.

## Session status values

| Status | Meaning |
| --- | --- |
| `NOT_STARTED` | Session created, user not yet on the verification page |
| `IN_PROGRESS` | User is interacting with the flow |
| `IN_REVIEW` | Awaiting human / async review |
| `APPROVED` | All checks passed (or manually approved) |
| `DECLINED` | Checks failed (or manually declined) |
| `ABANDONED` | User left before completing |
| `EXPIRED` | TTL elapsed before completion |

**Terminal** (a webhook fires, the lifecycle ends): `APPROVED`, `DECLINED`, `ABANDONED`, `EXPIRED`.
**Non-terminal** (still in flight): `NOT_STARTED`, `IN_PROGRESS`, `IN_REVIEW`.

## Two signals, one authority

- **The redirect `?status=` is a hint.** The browser returns to your `redirectUrl` with
  `?session_id=…&status=…`. **Never grant access on that query param** — a user can edit it.
- **A signed webhook fires on the terminal state** — a notification, not the result.
- **`verify.sessions.decision(id)` is authoritative** — session status plus the per-check breakdown.

## How to act

```text
NOT_STARTED / IN_PROGRESS -> wait; the run is not finished. Keep the session pending.
IN_REVIEW                 -> wait for the outcome; do NOT grant access yet.
APPROVED                  -> fetch the decision, then grant access / complete onboarding.
DECLINED                  -> fetch the decision to see which checks failed; deny, and offer a
                             retry if your policy allows.
ABANDONED / EXPIRED       -> treat as not verified; create a NEW session if they still need to verify.
```

| `d.status` | What to do |
| --- | --- |
| `APPROVED` | Grant access. Store the decision against the user. |
| `DECLINED` | Show a clear message. Inspect `d.checks` for which check failed and why. **Never surface raw error messages or codes to the end user.** |
| `IN_REVIEW` | Show "we'll be in touch". A terminal webhook arrives when review completes. |
| `ABANDONED` / `EXPIRED` | Offer to restart — create a new session. |

## Check status values

| Check status | Meaning |
| --- | --- |
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
```

Relationship:

- All checks `passed` → session typically `APPROVED`.
- Any check `failed` → session typically `DECLINED`.
- Any check `review` and none failed → session typically `IN_REVIEW` until resolved.

## Reading the decision

```js
const d = await verify.sessions.decision(sessionId);
// d.status  -> "APPROVED" | "DECLINED" | "IN_REVIEW"
// d.checks  -> [{ type, status, score, data, error }]

const credential = d.checks.find(c => c.type === "credential");
if (credential?.status === "failed") {
  console.log(credential.error?.message);   // e.g. "License belongs to a different name"
}
```

A decision from a **connected** session also carries `identity` — the reusable proof
(`valyd_id`, pseudonym, `id_verified`, age bands, licenses, `verified_at`) — and marks steps
skipped from the account with `reused: true`.

For a KYC + License workflow, `APPROVED` means **all four**: the ID was authentic, the selfie was
live, the selfie matched the ID portrait, and the license belongs to the person on the ID.

## Manual override

`verify.sessions.updateStatus(id, "APPROVED" | "DECLINED")` forces a terminal decision — for
example when a human reviewer on your side resolves an `IN_REVIEW` session.

- Valid **only while non-terminal**. Once terminal it cannot be flipped.
- Authenticated by the project API key alone — **any holder of that key can override**; there is no
  separate reviewer role.
- The decision is recorded on the session and delivered to your webhook like any other result.

## Build defensively

New enum values can be added at any time — a new check status, a new failure `signal`, a new event
`type`. **Never hard-fail on an unrecognized value**: treat it as the closest known category (an
unknown terminal status as "not approved"). Ignore unknown response fields rather than rejecting
the payload.

## Related

- [`verification-sessions.md`](verification-sessions.md) — creating the session and reading results
- [`webhooks.md`](webhooks.md) — the signed terminal-state notification
- [`checks-reference.md`](checks-reference.md) — what each check returns
