# Hosted verification

Create a session, redirect the user to a Valyd-hosted page that captures everything, receive the
result via a signed webhook plus the decision API. No UI to build, no PII handled by you.

> **Scope note.** Plain Hosted is **Non-account (Fresh)**: no Valyd login, nothing retained, and the
> decision returns the **raw** captured data. For the **Account** variant — result stored on the
> user's account, reuse (returning users do a selfie only), **proofs-only** results — it's the same
> hosted page; you additionally pass the user's `valyd_access_token` when creating the session. See
> `verify-modes-and-account.md`.

## What you need

| Variable | Where from |
|---|---|
| `VALYD_API_KEY` | Developer Portal → your Verify project → API key (shown once) |
| `VALYD_WEBHOOK_SECRET` | Developer Portal → Verify project → Webhooks |
| `VALYD_WORKFLOW_ID` | Developer Portal → Workflows → copy `workflow_id` |
| `APP_URL` | your public server URL, e.g. `https://api.example.com` |

Every server-to-server call uses `X-API-Key: <App API key>` — **server-side only**. Every response
uses the envelope `{ success, data, error: { code, message } }`.

```bash
npm i @valyd/sdk
```

```js
import { VerifyClient } from "@valyd/sdk";

const verify = new VerifyClient({
  apiKey:        process.env.VALYD_API_KEY,
  webhookSecret: process.env.VALYD_WEBHOOK_SECRET,   // used by verify.webhooks.constructEvent
});
```

## The flow

1. Create a session on your server with a `workflow_id`.
2. Redirect the user's browser to the returned `url`.
3. Valyd captures everything and redirects back to your `redirect_url`.
4. Receive a signed webhook, then fetch the authoritative result from the decision API.

## The two hosted presets

Both use **identical integration code** — only the `workflow_id` differs. Create them in the portal
under Workflows.

```text
Only verify a professional license (no ID scan)  -> "License Verification" workflow_id
Identity + credential (ID scan + selfie + license) -> "KYC + License" workflow_id
Unsure -> open the portal -> Workflows and copy the workflow_id of the preset you created
```

**License Verification** — checks `[credential]`. Flow: state → license type → name + number →
verify. Fastest path; no ID scan.

**KYC + License** — checks `[id_verification, liveness, face_match, credential]`. Flow: scan ID +
selfie (OCR + liveness + 1:1 face match), then state + license type + number. **The name is taken
from the verified ID automatically** — the user doesn't type it — so a license belonging to a
different person is rejected.

---

## Step 1 — create a session (server-side)

```bash
curl -X POST https://idp.valyd.work/api/v2/session \
  -H "X-API-Key: $VALYD_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "workflow_id":  "wf_...",
    "redirect_url": "https://app.example.com/verify/callback",
    "callback":     "https://api.example.com/webhooks/valyd",
    "vendor_data":  "user_123",
    "ttl_seconds":  900,
    "metadata":     { "plan": "pro" }
  }'
```

```js
const session = await verify.sessions.create({
  workflowId:  process.env.VALYD_WORKFLOW_ID,
  redirectUrl: "https://app.example.com/verify/callback",
  callback:    "https://api.example.com/webhooks/valyd",
  vendorData:  "user_123",          // your internal ref — echoed back on the webhook
  ttlSeconds:  900,
  metadata:    { plan: "pro" },
});
// session.url, session.sessionId, session.sessionToken, session.expiresAt
```

**200 OK**, `data`:

```json
{
  "session_id":    "ses_...",
  "status":        "NOT_STARTED",
  "url":           "https://idp.valyd.work/s/...",
  "session_token": "stk_...",
  "features":      ["id_verification","liveness","face_match","credential"],
  "redirect_url":  "https://app.example.com/verify/callback",
  "expires_at":    "2026-06-11T12:00:00Z"
}
```

## Step 2 — redirect the user

Send the browser to `data.url`. Valyd handles the entire capture and verification UI; the steps
auto-adapt to the workflow's checks.

```js
app.post("/start-verification", async (req, res) => {
  const session = await verify.sessions.create({
    workflowId:  process.env.VALYD_WORKFLOW_ID,
    redirectUrl: `${process.env.APP_URL}/verify/callback`,
    callback:    `${process.env.APP_URL}/webhooks/valyd`,
    vendorData:  req.user.id,
  });
  res.redirect(session.url);
});
```

## Step 3 — handle the redirect back

Valyd redirects to `redirect_url?session_id=…&status=…`. **Treat `status` as a hint only** — a user
can edit query params. Always fetch the authoritative decision.

```js
app.get("/verify/callback", (req, res) => {
  const { session_id } = req.query;
  // Don't trust ?status= — confirm via the decision API or webhook.
  res.redirect(`/verify/pending?s=${session_id}`);
});
```

## Step 4 — get the authoritative result

Use the signed webhook (push) and/or the decision endpoint (pull). Signature verification, retries
and deduplication are covered in `webhooks.md`.

```js
app.post("/webhooks/valyd", express.raw({ type: "application/json" }), async (req, res) => {
  try {
    const event = verify.webhooks.constructEvent(req.body, req.headers);
    const decision = await verify.sessions.decision(event.sessionId);   // the full result
    await persist(event.vendorData, decision);
    res.json({ ok: true });
  } catch (err) {
    if (err instanceof ValydVerifyError && err.code === "invalid_signature") {
      return res.status(400).send("bad signature");
    }
    throw err;
  }
});
```

---

## Reading the decision

```text
You received a webhook
  -> it carries event.status and event.decision; still call the decision API for the full breakdown.
You want to pull it yourself
  -> GET /api/v2/session/<session_id>/decision with X-API-Key.

Then read d.status:
  APPROVED  -> verification succeeded (see the per-check rule below for KYC + License)
  DECLINED  -> failed; inspect d.checks for the failing check's error
  IN_REVIEW -> manual review pending; wait for a terminal webhook or poll again

Per check in d.checks ({ type, status, score, data, error }):
  passed             -> this check succeeded
  failed             -> read check.error?.message, e.g. "License belongs to a different name"
  review             -> awaiting manual review
  pending / running  -> not finished yet
```

```bash
curl https://idp.valyd.work/api/v2/session/ses_.../decision \
  -H "X-API-Key: $VALYD_API_KEY"
```

```js
const d = await verify.sessions.decision(sessionId);
// d.status     -> session progress: "APPROVED" | "DECLINED" | "IN_REVIEW"
// d.decision   -> final business outcome, resolved only: "APPROVED" | "DECLINED" (never IN_REVIEW)
// d.checks     -> [{ type, status, score, data, error }]
// d.decided_at -> ISO timestamp

const credential = d.checks.find(c => c.type === "credential");
if (credential.status === "failed") console.log(credential.error?.message);
```

**200 OK**, `data`:

```json
{
  "status":   "APPROVED",
  "decision": "APPROVED",
  "decided_at": "2026-06-11T12:05:00Z",
  "checks": [
    { "type": "id_verification", "status": "passed", "score": 0.97, "data": {} },
    { "type": "liveness",        "status": "passed", "score": 1.00, "data": { "live_score": 1 } },
    { "type": "face_match",      "status": "passed", "score": 0.97, "data": { "similarity": 0.97, "threshold": 0.95 } },
    { "type": "credential",      "status": "passed", "score": 1.00,
      "data": { "match": true, "license": { "status": "active", "expires_at": "2027-01-01" } } }
  ]
}
```

**APPROVED for KYC + License means all four:** the ID was verified (OCR + authenticity), the selfie
was live, the selfie matched the ID portrait, and the license exists **and belongs to the person on
the ID**.

---

## Session & workflow helpers

```js
// Sessions
const session = await verify.sessions.retrieve(sessionId);
const page    = await verify.sessions.list({ status: "APPROVED", vendorData: "user_123", limit: 50 });
await verify.sessions.updateStatus(sessionId, "APPROVED");   // or "DECLINED" — manual override

// Workflows  (<-> /api/v2/workflows)
const wf = await verify.workflows.create({
  name: "KYC + License",
  features: ["id_verification", "liveness", "face_match", "credential"],
});
await verify.workflows.list();
await verify.workflows.retrieve(wf.id);
await verify.workflows.update(wf.id, { name: "KYC + License (v2)" });
await verify.workflows.remove(wf.id);
```

Manual override works only **before** the session is terminal (`NOT_STARTED`, `IN_PROGRESS`,
`IN_REVIEW`). Once `APPROVED`/`DECLINED` it returns `409 already_decided`. It is authenticated by the
project API key alone — there is no separate reviewer role — and the override is delivered to your
webhook like any other result.

## Endpoints used in this flow

| Method | Path | Purpose |
|---|---|---|
| POST | `/api/v2/session` | Create a hosted session |
| GET | `/api/v2/session` | List sessions (`status`, `vendorData`, `limit`) |
| GET | `/api/v2/session/{id}` | Retrieve a session |
| GET | `/api/v2/session/{id}/decision` | Authoritative decision + per-check breakdown |
| PATCH | `/api/v2/session/{id}/status` | Manual override to `APPROVED` / `DECLINED` |
| POST | `/api/v2/workflows` | Create a workflow |
| GET | `/api/v2/workflows` | List |
| GET | `/api/v2/workflows/{id}` | Retrieve |
| PATCH | `/api/v2/workflows/{id}` | Update |
| DELETE | `/api/v2/workflows/{id}` | Delete |
| POST | *your callback URL* | Webhook, signed with `X-Valyd-Signature` |

All on `https://idp.valyd.work`, all authenticated with `X-API-Key`.

> The docs also describe workflows as portal-only in one place and as a REST resource in another.
> The SDK exposes `verify.workflows.*`; if a create call 404s, fall back to creating the workflow in
> the portal. See `gotchas-and-doc-conflicts.md`.

---

## End-to-end Express example

```js
import express from "express";
import { VerifyClient, ValydVerifyError } from "@valyd/sdk";

const app = express();
const verify = new VerifyClient({
  apiKey:        process.env.VALYD_API_KEY,
  webhookSecret: process.env.VALYD_WEBHOOK_SECRET,
});

// 1) Start hosted verification
app.post("/start-verification", express.json(), async (req, res) => {
  const session = await verify.sessions.create({
    workflowId:  process.env.VALYD_WORKFLOW_ID,
    redirectUrl: `${process.env.APP_URL}/verify/callback`,
    callback:    `${process.env.APP_URL}/webhooks/valyd`,
    vendorData:  req.body.userId,
  });
  res.json({ url: session.url, sessionId: session.sessionId });
});

// 2) Redirect-back (status is a hint only)
app.get("/verify/callback", (req, res) => {
  res.redirect(`/verify/pending?s=${req.query.session_id}`);
});

// 3) Signed webhook — MUST use raw body
app.post("/webhooks/valyd", express.raw({ type: "application/json" }), async (req, res) => {
  try {
    const event = verify.webhooks.constructEvent(req.body, req.headers);
    const decision = await verify.sessions.decision(event.sessionId);
    await persist(event.vendorData, decision);
    res.json({ ok: true });
  } catch (err) {
    if (err instanceof ValydVerifyError && err.code === "invalid_signature") {
      return res.status(400).send("bad signature");
    }
    throw err;
  }
});

app.listen(3000);
```

## Common errors

**Invalid webhook signature** — verifying against a re-serialized JSON body, or the wrong secret.
Use `express.raw({ type: "application/json" })` and the correct `VALYD_WEBHOOK_SECRET`. The SDK
throws `ValydVerifyError` with `code === "invalid_signature"`; return 400.

**Trusting the redirect `?status=` as final** — it's a hint. Confirm via webhook or decision API.

**Exposing the API key to the browser** — calling `/api/v2/session` from frontend code. Keep
`X-API-Key` server-side. Only the hosted `url` and `session_token` go to the browser.

**Webhook not received** — the callback URL isn't publicly reachable or returns non-2xx. In
development use a tunnel (ngrok, Cloudflare Tunnel). Return 2xx within ~30 s; do heavy work async.

**401 on session create** — missing/wrong `X-API-Key`, or it's a different credential type (not the
App API key).
