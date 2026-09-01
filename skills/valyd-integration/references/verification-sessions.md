# Running a verification session

One flow covers **both products**. The only difference is whether you attach a user's token.

```text
sessions.create({ workflowId, ... })  ->  redirect to session.url  ->  sessions.decision(id)
```

| | Reusable Verification | Unique Human API |
| --- | --- | --- |
| `valydAccessToken` | **required** — ties the run to the connected user | **omitted** — no account involved |
| Result | proofs save to the user's Valyd ID | returned to you; nothing saved |
| Reuse | already-proven steps are skipped | every check runs every time |

## What you need

| Variable | Where from |
| --- | --- |
| `VALYD_API_KEY` | Developer Portal → your project → API key (`vrf_…`, shown once) |
| `VALYD_WORKFLOW_ID` | Developer Portal → Workflows → copy the `workflowId` |
| `VALYD_WEBHOOK_SECRET` | Developer Portal → Verification → Webhooks (`whsec_…`) |

The App API key is **server-side only**. Only `session.url` and the `sessionToken` reach the
browser.

```js
import { VerifyClient } from "@valyd/sdk";

const verify = new VerifyClient({
  apiKey:        process.env.VALYD_API_KEY,
  webhookSecret: process.env.VALYD_WEBHOOK_SECRET,
});
```

---

## Step 1 — create the session

```js
const session = await verify.sessions.create({
  workflowId:       process.env.VALYD_WORKFLOW_ID,
  valydAccessToken: accessToken,          // omit for the Unique Human API
  redirectUrl:      `${APP_URL}/verify/callback`,
  callback:         `${APP_URL}/webhooks/valyd`,
  vendorData:       user.id,              // your internal ref — echoed back on the webhook
  ttlSeconds:       900,
  metadata:         { plan: "pro" },
  idempotencyKey:   crypto.randomUUID(),  // safe retries — see below
});
```

The response carries the verification page `url`, a `sessionId`, a `sessionToken`, `expiresAt`, and
a `features` array echoing the workflow's checks.

### Idempotency

Session creation is **billable**. Pass an `idempotencyKey` — a UUID per logical operation, reused
across retries of that operation — and Valyd runs the check once, records the result against your
key, and returns that stored result for any later request with the same key. Without it, a timeout
you retry can run and charge the check twice.

## Step 2 — redirect the person

```js
res.redirect(session.url);
```

Valyd's verification page handles the entire capture — camera, retries, security, and the steps
auto-adapt to the workflow's checks. **There is no browser SDK and nothing to embed.**

## Step 3 — handle the redirect back

Valyd returns the user to `redirectUrl?session_id=…&status=…`.

```js
app.get("/verify/callback", (req, res) => {
  // ?status= is a UI hint only — a user can edit it. Never gate access on it.
  res.redirect(`/verify/pending?s=${req.query.session_id}`);
});
```

## Step 4 — read the authoritative decision

Two signals arrive; only one is authority.

- **The redirect `?status=` is a hint.**
- **A signed webhook fires on the terminal state** — a notification, not the result.
- **`verify.sessions.decision(id)` is authoritative** — the status plus the per-check breakdown.

```js
const d = await verify.sessions.decision(sessionId);
// d.status  -> "APPROVED" | "DECLINED" | "IN_REVIEW"
// d.checks  -> [{ type, status, score, data, error }]

const credential = d.checks.find(c => c.type === "credential");
if (credential?.status === "failed") {
  console.log(credential.error?.message);   // e.g. "License belongs to a different name"
}
```

A decision from a connected session also carries `identity` — the reusable proof
(`valyd_id`, pseudonym, `id_verified`, age bands, licenses, `verified_at`) — and marks reused
steps with `reused: true`.

---

## The webhook handler

Full detail in [`webhooks.md`](webhooks.md); the shape you need:

```js
import { ValydVerifyError } from "@valyd/sdk";

app.post("/webhooks/valyd", express.raw({ type: "application/json" }), async (req, res) => {
  let event;
  try {
    event = verify.webhooks.constructEvent(req.body, req.headers);   // raw Buffer, not parsed JSON
  } catch (err) {
    if (err instanceof ValydVerifyError && err.code === "invalid_signature") {
      return res.status(400).send("bad signature");
    }
    throw err;
  }

  if (await alreadyProcessed(event.eventId)) return res.json({ ok: true });   // at-least-once

  const decision = await verify.sessions.decision(event.sessionId);
  await persist(event.vendorData, decision);
  res.json({ ok: true });
});
```

**Webhooks are optional.** Polling `sessions.decision(id)` when the user returns is a legitimate
design. The webhook is more reliable only in that it fires even if the user closes the tab.

---

## Other session methods

```js
await verify.sessions.retrieve(sessionId);
await verify.sessions.list({ status: "APPROVED", vendorData: "user_123", limit: 50 });
await verify.sessions.updateStatus(sessionId, "APPROVED");   // or "DECLINED" — manual override
```

Manual override works only while the session is **non-terminal**. Once `APPROVED`/`DECLINED` it
cannot be flipped. It is authenticated by the project API key alone — there is no separate reviewer
role — and the override is delivered to your webhook like any other result.

There is **no workflow CRUD in the SDK** — workflows are composed in the portal. See
[`workflows.md`](workflows.md).

---

## Sessions are never resumed

A terminal session is terminal. `ABANDONED` and `EXPIRED` cannot be restarted — create a new
session. Bound the window with `ttlSeconds` when you create it.

Full lifecycle and status handling: [`statuses.md`](statuses.md).

## Common errors

**Invalid webhook signature** — the body was parsed and re-serialised before verification, or the
wrong secret was used. Use `express.raw({ type: "application/json" })`.

**Trusting `?status=` as final** — it's a UI hint. Read the decision.

**401 on session create** — the `X-API-Key` is missing, wrong, from another environment, or being
sent from the browser.

**Webhook never arrives** — the callback URL isn't publicly reachable or returns non-2xx. Use a
tunnel (ngrok, Cloudflare Tunnel) in development; return 2xx within ~30 s and do heavy work async.

**Double charges after a retry** — you omitted `idempotencyKey`.
