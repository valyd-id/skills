# Webhooks

Valyd POSTs to your app-level or session-level callback URL when a verification session reaches a
terminal state. **The webhook is a notification only** — always call
`GET /api/v2/session/{id}/decision` for the full extracted data.

> **Read the RAW request body before parsing it as JSON.** You need the exact bytes that were sent
> to verify the HMAC. If your framework auto-parses and re-serializes, the bytes differ and
> verification fails 100% of the time.

## What you need

| Variable | Where from |
| --- | --- |
| `VALYD_WEBHOOK_SECRET` | Developer Portal → Verify project → Webhooks (`whsec_…`) |
| `VALYD_API_KEY` | App API key, to fetch the full decision |

Plus a publicly reachable HTTPS endpoint, and the ability to read the raw body in your framework.

Register the callback URL app-wide in the portal, or pass a per-session `callback` when creating the
session.

## Headers

| Header | Meaning |
| --- | --- |
| `X-Valyd-Timestamp` | unix seconds when the request was signed |
| `X-Valyd-Event-Id` | unique event id — **use it to deduplicate** |
| `X-Valyd-Signature` | `HMAC_SHA256("{timestamp}.{rawBody}", webhook_signing_secret)`, lowercase hex |

Verify against the raw body, compare in constant time, and reject stale timestamps (> 5 minutes).

## Verifying — with the SDK

```js
import express from "express";
import { VerifyClient, ValydVerifyError } from "@valyd/sdk";

const verify = new VerifyClient({
  apiKey:        process.env.VALYD_API_KEY,
  webhookSecret: process.env.VALYD_WEBHOOK_SECRET,
});

// IMPORTANT: raw body for signature verification
app.post("/webhooks/valyd", express.raw({ type: "application/json" }), async (req, res) => {
  try {
    const event = verify.webhooks.constructEvent(req.body, req.headers);
    // event.sessionId, event.type, event.status, event.decision, event.vendorData
    await persistDecision(event);
    res.json({ ok: true });
  } catch (err) {
    if (err instanceof ValydVerifyError && err.code === "invalid_signature") {
      return res.status(400).send("bad signature");
    }
    throw err;
  }
});
```

`verify.webhooks.verify(rawBody, headers, secret?, { toleranceSeconds? })` is the boolean variant —
no parse, no throw.

## Verifying — without the SDK

```js
import crypto from "node:crypto";

function verifyValydSignature(rawBody, headers, secret) {
  const ts  = headers["x-valyd-timestamp"];
  const sig = headers["x-valyd-signature"];
  if (!ts || !sig) throw new Error("missing signature headers");

  // Reject stale (> 5 min). The timestamp is signed into the HMAC, so it can't be tampered with.
  if (Math.abs(Date.now() / 1000 - Number(ts)) > 300) throw new Error("stale timestamp");

  const expected = crypto
    .createHmac("sha256", secret)
    .update(`${ts}.${rawBody}`)
    .digest("hex");

  const a = Buffer.from(expected, "hex");
  const b = Buffer.from(sig, "hex");
  if (a.length !== b.length || !crypto.timingSafeEqual(a, b)) throw new Error("invalid signature");
}
```

Full Express handler:

```js
app.post("/api/valyd-webhook", express.raw({ type: "application/json" }), (req, res) => {
  const ts  = req.header("X-Valyd-Timestamp");
  const sig = req.header("X-Valyd-Signature") || "";
  const raw = req.body;   // Buffer

  const expected = crypto
    .createHmac("sha256", process.env.VALYD_WEBHOOK_SECRET)
    .update(`${ts}.${raw.toString("utf8")}`)
    .digest("hex");

  const ok = sig.length === expected.length &&
    crypto.timingSafeEqual(Buffer.from(expected), Buffer.from(sig));
  if (!ok) return res.status(400).send("bad signature");

  if (Math.abs(Date.now() / 1000 - Number(ts)) > 300) return res.status(400).send("stale timestamp");

  const event = JSON.parse(raw.toString("utf8"));
  // Respond fast; fetch the full decision asynchronously.
  res.status(200).json({ ok: true });
});
```

## Event body

```json
{
  "event_id":    "evt_...",
  "type":        "verification.approved",
  "session_id":  "ses_...",
  "status":      "APPROVED",
  "vendor_data": "user_123",
  "decision":    "approved",
  "occurred_at": "2026-06-11T12:05:00Z"
}
```

- `event_id` — matches `X-Valyd-Event-Id`; deduplicate on it, for retries **and manual resends**
  (both carry the same id).
- `session_id` — pass to `GET /api/v2/session/{id}/decision`.
- `status` — the session status; see `statuses.md`.
- `decision` — the final business outcome (`approved` / `declined`).

**Event types:**

| Type | Meaning |
| --- | --- |
| `verification.approved` | terminal, passed |
| `verification.declined` | terminal, failed |
| `verification.in_review` | manual/agent review pending — **not terminal** |
| `verification.abandoned` | user left the hosted flow without finishing |
| `verification.expired` | the session TTL elapsed |

Per the versioning policy, **a new event type can be added at any time** — handle unknown values
gracefully rather than hard-failing. (One docs page shows a `verification.completed` type not in this
list; see `gotchas-and-doc-conflicts.md`.)

The `decision` field on the webhook and the decision API's response are documented with slightly
different shapes across pages — treat the decision API as the source of truth for structure.

## Delivery and retries

- Return **2xx fast** — defer heavy work to a background queue. Your handler should respond within
  ~30 s.
- Non-2xx or timed-out deliveries are **retried automatically**. Delivery is **at-least-once**;
  deduplicate on `X-Valyd-Event-Id`.
- Verify the signature and reject stale timestamps on **every** delivery.

### Retry schedule

Up to **10 attempts**:

| After attempt | Wait before next try |
| --- | --- |
| 1 | 5 seconds |
| 2 | 30 seconds |
| 3 | 2 minutes |
| 4 | 10 minutes |
| 5 and later | 30 minutes |

A persistently failing endpoint is retried across roughly **2.5 hours** before Valyd stops. Every
retry carries the **same** `X-Valyd-Event-Id`, so deduplicating collapses all retries of one event
into a single unit of work.

### Delivery log and manual resend

Every attempt — successful or failed — is recorded in the Developer Portal, on your application's
**Verification** page under **Recent webhook deliveries**: destination URL, exact payload and
headers sent, the receiver's HTTP status and response body, and any transport error.

**Resend** re-queues a webhook — after you've fixed a handler bug, or if the endpoint was down during
the automatic retry window. A manual resend carries the **same** `X-Valyd-Event-Id`, so an idempotent
handler treats it as the same event.

> If a webhook is missed entirely, `GET /api/v2/session/{id}/decision` is the authoritative source —
> fetch or poll it any time. **Webhooks are optional**; polling the decision when the user returns
> is a legitimate design. The webhook is more reliable only in that it fires even if the user closes
> the tab.

## Fetch the full decision

```bash
curl https://idp.valyd.work/api/v2/session/SES_ID/decision -H "X-API-Key: $VALYD_API_KEY"
```

## Verifying your handler works

- Trigger a real terminal session (or use Resend) and confirm a valid signature logs and returns 200.
- **Deliberately corrupt the secret and confirm you return 400 `bad signature`** — this proves
  verification is actually running rather than silently passing.

## Common errors

**Signature always mismatches.** The framework parsed and re-serialized the JSON body. Capture the
raw body (`express.raw({ type: "application/json" })`) and HMAC the exact bytes before any parsing.

**Signature mismatches despite the raw body.** Wrong signing input or secret. The signed string is
`"{timestamp}.{rawBody}"` — timestamp, a literal dot, then the raw body — and the secret must be the
**webhook signing secret**, not the App API key.

**Duplicate processing.** Your endpoint returned non-2xx or timed out, so Valyd retried. Return 2xx
immediately, process asynchronously, deduplicate on `X-Valyd-Event-Id`.

**No webhook at all.** URL/secret not configured, or the endpoint isn't publicly reachable. Use a
tunnel in development.
