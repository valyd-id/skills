# Worked recipes

Four end-to-end builds. All four use the **same session flow** — only the workflow and whether you
attach a user token change.

| Recipe | Product | Workflow checks |
| --- | --- | --- |
| Ship KYC for a connected user | Reusable Verification | `[id_verification, liveness, face_match]` |
| Verify a professional license | Reusable Verification | `[credential]` or `[id_verification, liveness, face_match, credential]` |
| Electronic Visit Verification (EVV) | Reusable Verification | `[id_verification, liveness, face_match, credential, location]` |
| Liveness / anti-spoof gate | Unique Human API | `[antispoof]` (+ `face-uniqueness`) |

---

## 1. Ship KYC for a connected user

Connect the user, **read what they already have**, and only run a workflow for what's missing.

```bash
npm i @valyd/sdk
```

```bash
# .env
VALYD_CLIENT_ID=...
VALYD_CLIENT_SECRET=...
VALYD_REDIRECT_URI=https://app.example.com/auth/valyd/callback
VALYD_API_KEY=vrf_...
VALYD_WEBHOOK_SECRET=whsec_...
VALYD_WORKFLOW_ID=wf_...
VALYD_IDP_URL=https://idp.valyd.id
```

**Step 1 — connect the user.**

```js
import { ValydClient, VerifyClient } from "@valyd/sdk";

const valyd = new ValydClient({
  clientId: process.env.VALYD_CLIENT_ID,
  clientSecret: process.env.VALYD_CLIENT_SECRET,
  redirectUri: process.env.VALYD_REDIRECT_URI,
  baseUrl: process.env.VALYD_IDP_URL,
});

app.get("/login", (req, res) => {
  const transaction = valyd.auth.createAuthorizationRequest({
    scope: ["profile", "verifications"],
  });
  req.session.valydOidc = transaction;
  res.redirect(transaction.url);
});

app.get("/auth/valyd/callback", async (req, res) => {
  const transaction = req.session.valydOidc;
  delete req.session.valydOidc;
  if (!transaction) return res.status(400).send("Login expired");

  const { user, accessToken } = await valyd.auth.handleCallback(req.originalUrl, { transaction });
  req.session.user = user;                 // user.valyd_id is your primary key
  req.session.valydAccessToken = accessToken;
  res.redirect("/onboarding");
});
```

**Step 2 — read before you verify.** This is free; the check is not.

```js
app.get("/onboarding", async (req, res) => {
  const proofs = await valyd.auth.getVerifications(req.session.valydAccessToken);
  if (proofs.id_verified) return res.redirect("/dashboard");   // done — no check, no cost
  res.redirect("/verify");
});
```

**Step 3 — run the workflow for what's missing.**

```js
const verify = new VerifyClient({
  apiKey: process.env.VALYD_API_KEY,
  webhookSecret: process.env.VALYD_WEBHOOK_SECRET,
  baseUrl: process.env.VALYD_IDP_URL,
});

app.post("/verify", async (req, res) => {
  const session = await verify.sessions.create({
    workflowId:       process.env.VALYD_WORKFLOW_ID,
    valydAccessToken: req.session.valydAccessToken,   // proofs save to their Valyd ID
    redirectUrl:      `${process.env.APP_URL}/verify/callback`,
    callback:         `${process.env.APP_URL}/webhooks/valyd`,
    vendorData:       req.session.user.valyd_id,
    idempotencyKey:   crypto.randomUUID(),
  });
  res.redirect(session.url);
});
```

**Step 4 — redirect back, then read the real decision.**

```js
app.get("/verify/callback", (req, res) => {
  // ?status= is a hint — never gate on it
  res.redirect(`/verify/pending?s=${req.query.session_id}`);
});

app.post("/webhooks/valyd", express.raw({ type: "application/json" }), async (req, res) => {
  let event;
  try {
    event = verify.webhooks.constructEvent(req.body, req.headers);
  } catch (err) {
    if (err instanceof ValydVerifyError && err.code === "invalid_signature")
      return res.status(400).send("bad signature");
    throw err;
  }
  if (await alreadyProcessed(event.eventId)) return res.json({ ok: true });

  const decision = await verify.sessions.decision(event.sessionId);
  if (decision.status === "APPROVED") await grantAccess(event.vendorData, decision);
  res.json({ ok: true });
});
```

**Next time this user logs in, step 2 answers and step 3 never runs.** That is the whole point.

---

## 2. Verify a professional license

Same code — a different `workflowId`.

- **License only** (`[credential]`) — state → license type → name + number → verify. No ID scan.
- **KYC + License** (`[id_verification, liveness, face_match, credential]`) — the name comes from
  the **verified ID**, so the user cannot present someone else's license.

```js
const session = await verify.sessions.create({
  workflowId:       process.env.VALYD_LICENSE_WORKFLOW_ID,
  valydAccessToken: accessToken,
  redirectUrl:      `${APP_URL}/license/done`,
  callback:         `${APP_URL}/webhooks/valyd`,
  vendorData:       user.valyd_id,
});
```

Optionally build a state/type picker first — read-only and cacheable:

```js
const { states }    = await verify.credentials.states();
const { providers } = await verify.credentials.providers("CA");   // each has required_fields
```

Reading the result:

```js
const d = await verify.sessions.decision(sessionId);
const credential = d.checks.find(c => c.type === "credential");

if (credential?.status === "passed") {
  // credential.data.license -> { status, expires_at, specialty, ... }
} else {
  // credential.error.code -> license_not_found | license_expired | license_inactive
  //                        | name_mismatch | board_unavailable
}
```

**Registry lookups take 10–60 s**, so the session may sit in progress noticeably longer than a
pure-KYC one. Never forward the raw error code to the end user.

---

## 3. Electronic Visit Verification (EVV)

Prove the **right clinician** reached the **right home** — verified identity, a live medical
license, a face match, and geolocation. Built on Reusable Verification, so clinicians **verify once
and reuse everywhere**.

The whole thing is **one workflow**, composed in the portal — you do not wire the checks together
in code.

Workflow: `[id_verification, liveness, face_match, credential, location]`

```js
// One call runs the whole flow, in the order you configured in the portal.
const session = await verify.sessions.create({
  workflowId:       process.env.VALYD_EVV_WORKFLOW_ID,
  valydAccessToken: accessToken,        // the clinician's token — unlocks reuse
  redirectUrl:      `${APP_URL}/visits/verified`,
  callback:         `${APP_URL}/webhooks/valyd`,
  vendorData:       visit.caregiverId,
  metadata:         { expected_lat: home.lat, expected_lng: home.lng },
});
res.redirect(session.url);
```

One signed webhook fires with one decision covering every check:

```json
{
  "status": "APPROVED",
  "checks": [
    { "type": "id_verification", "status": "passed", "data": { "reused": true } },
    { "type": "liveness",        "status": "passed" },
    { "type": "face_match",      "status": "passed", "score": 0.98 },
    { "type": "credential",      "status": "passed", "data": { "license": { "status": "active" } } },
    { "type": "location",        "status": "passed", "data": { "distance_m": 12, "radius_m": 200, "match": true } }
  ],
  "identity": { "valyd_id": "valyd_225c…", "id_verified": true, "verified_at": "…" }
}
```

`reused: true` marks steps skipped from the clinician's account.

**The rules that matter here:**

- **KYC and license are one-time onboarding steps.** Do not put a KYC or license button on every
  visit. The recurring visit action is face match + location — which, for an already-verified
  clinician, is what the workflow reduces to automatically.
- **Location can never be skipped.** A blocked permission or missing coordinates is a hard
  `failed`. Where a radius is configured, the status **is** the geofence verdict.
- **A returning clinician re-verifies with a selfie only**, matched against their stored face
  vector.

> Older EVV documentation shows this built from direct `verify.standalone.*` calls and browser
> capture helpers. Those methods were removed in v1.10.4/v1.10.5 — the current shape is the single
> workflow session above. See [`gotchas-and-doc-conflicts.md`](gotchas-and-doc-conflicts.md).

---

## 4. Liveness / anti-spoof gate — no login

The [Unique Human API](unique-human-api.md): is this a live, unique human? App API key only.

```js
import { VerifyClient } from "@valyd/sdk";
const verify = new VerifyClient({ apiKey: process.env.VALYD_API_KEY });

// No valydAccessToken -> nothing is saved to any account
const session = await verify.sessions.create({
  workflowId:  process.env.VALYD_LIVENESS_WORKFLOW_ID,   // antispoof (+ face-uniqueness)
  redirectUrl: "https://yourapp.com/checked",
  vendorData:  "signup-" + signupId,
});
res.redirect(session.url);
```

Valyd's page records a live camera burst and issues a **random on-screen action** — turn your head,
open your mouth, nod, move closer. An attacker can't know the action in advance, which is what
makes a photo, screen replay, or pre-recorded clip fail. That path yields the strongest assurance,
`assurance: "captured"`.

```js
const d = await verify.sessions.decision(session.sessionId);
const spoof  = d.checks.find(c => c.type === "antispoof");
const unique = d.checks.find(c => c.type === "face_uniqueness");

if (d.status === "APPROVED") {
  // spoof.data.human_score -> 0–100
  // unique.data -> { valyd_uuid, registered: "new" | "existing" }
  if (unique?.data.registered === "existing") {
    // this face already opened an account — your policy decides
  }
}
```

Store the `valyd_uuid` against your own user record. **You hold this result** — encrypt it at rest,
keep it out of logs, and delete it on your own retention schedule. See
[`unique-human-api.md`](unique-human-api.md).

---

## Cross-cutting

Every recipe above shares the same four rules:

1. Read the account before paying for a check.
2. The decision call is the authority — not `?status=`, not the webhook body.
3. The webhook handler uses the raw body, checks the timestamp, and dedupes on the event id.
4. Every billable create carries an `idempotencyKey`.
