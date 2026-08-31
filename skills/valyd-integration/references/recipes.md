# Worked recipes

Four end-to-end builds. Each names the mode it uses so you can map it back to the 2×2.

| Recipe | Mode | Time |
|---|---|---|
| Ship hosted KYC | Non-account × Hosted | ~30 min |
| Verify a professional license | Non-account × Core | ~15 min |
| Electronic Visit Verification (EVV) | **Account** × either | — |
| Anti-spoof / liveness demo | Non-account × Hosted | — |

---

## 1. Ship hosted KYC

Session → redirect → signed webhook → authoritative decision. Covers both "License Verification"
and "KYC + License"; **only the `workflow_id` changes**.

`Express / Node.js` · `@valyd/sdk`

| Variable | Where from |
|---|---|
| `VALYD_API_KEY` | portal → Verify project → API key (shown once) |
| `VALYD_WEBHOOK_SECRET` | portal → Verify project → Webhooks |
| `VALYD_WORKFLOW_ID` | portal → Workflows → copy `workflow_id` |
| `APP_URL` | your public server URL |

**Step 1 — create a session.** Pass `vendor_data` to correlate the result back to your user.

```js
import { VerifyClient } from "@valyd/sdk";
const verify = new VerifyClient({
  apiKey:        process.env.VALYD_API_KEY,
  webhookSecret: process.env.VALYD_WEBHOOK_SECRET,
});

const session = await verify.sessions.create({
  workflowId:  process.env.VALYD_WORKFLOW_ID,
  redirectUrl: `${process.env.APP_URL}/verify/callback`,
  callback:    `${process.env.APP_URL}/webhooks/valyd`,
  vendorData:  req.user.id,     // echoed back on the webhook
  ttlSeconds:  900,
});
```

**Step 2 — redirect.** `res.redirect(session.url)`. Valyd's hosted page handles the whole capture
UI; no camera or document handling on your side.

**Step 3 — handle the redirect back.** `?status=` is a **hint**; a user can manipulate query params.

```js
app.get("/verify/callback", (req, res) => {
  res.redirect(`/verify/pending?s=${req.query.session_id}`);   // show a processing page
});
```

**Step 4 — receive and verify the webhook.** Raw body, dedupe on the event id.

```js
import { ValydVerifyError } from "@valyd/sdk";

app.post("/webhooks/valyd", express.raw({ type: "application/json" }), async (req, res) => {
  try {
    const event = verify.webhooks.constructEvent(req.body, req.headers);
    if (await alreadyProcessed(event.eventId)) return res.json({ ok: true });
    await handleEvent(event);
    res.json({ ok: true });
  } catch (err) {
    if (err instanceof ValydVerifyError && err.code === "invalid_signature") {
      return res.status(400).send("bad signature");
    }
    throw err;
  }
});
```

**Step 5 — read the authoritative decision.**

```js
const d = await verify.sessions.decision(event.sessionId);
const credential = d.checks.find(c => c.type === "credential");
if (credential?.status === "failed") {
  console.error("License check failed:", credential.error?.message);
}
```

**Acting on the result**

| `d.status` | Meaning | Do |
|---|---|---|
| `APPROVED` | All checks passed | Grant access; store the decision |
| `DECLINED` | One or more failed | Clear message; inspect `d.checks`. **Don't show raw errors to the user.** |
| `IN_REVIEW` | Awaiting manual review | "We'll be in touch"; a terminal webhook follows |
| `ABANDONED` / `EXPIRED` | Left or timed out | Offer restart — **sessions cannot be resumed**, create a new one |

For KYC + License, `APPROVED` means all four: the ID was authentic, the selfie was live, the selfie
matched the ID portrait, and the license belongs to the person on the ID.

**Errors:** invalid signature (re-serialized body / wrong secret) · trusting `?status=` · 401 on
session create (key missing, wrong, or sent from the browser) · webhook not received (callback not
publicly reachable — use ngrok / Cloudflare Tunnel in dev; return 2xx within ~30 s).

---

## 2. Verify a professional license

No hosted flow — your backend calls the API directly. Needs only `VALYD_API_KEY`.

**Step 1 — discover providers (optional).** Read-only, cacheable; boards don't change often.

```bash
curl https://idp.valyd.work/api/v2/credential/states -H "X-API-Key: $VALYD_API_KEY"
curl https://idp.valyd.work/api/v2/credential/states/CA/providers -H "X-API-Key: $VALYD_API_KEY"
```

You do **not** need to pass a `provider_id` / `provider_code` to the verification endpoint — Valyd
resolves the board from `license_type` + `license_state`. Discovery is for building a
provider-aware UI.

**Step 2 — submit.** Synchronous, **10–60 s**; timeout ≥90 s.

```js
import { VerifyClient } from "@valyd/sdk";
const verify = new VerifyClient({ apiKey: process.env.VALYD_API_KEY, timeoutMs: 90_000 });

const { check } = await verify.standalone.credentialVerification({
  licenseState:  "CA",
  licenseType:   "MD",          // provider auto-resolved
  licenseNumber: "G12345",
  fullName:      "Jane Smith",  // the board matches on name + number
});
```

```bash
curl -X POST https://idp.valyd.work/api/v2/credential-verification \
  -H "X-API-Key: $VALYD_API_KEY" -H "Content-Type: application/json" \
  --max-time 90 \
  -d '{"first_name":"Jane","last_name":"Smith","license_type":"MD","license_state":"CA","license_number":"G12345"}'
```

**Server-side only.** Never call this from the browser — your API key would be exposed. Have the
user submit their license details to your backend.

**Step 3 — read the result.** Act on the status; inspect the check for detail.

```js
if (result.status === "APPROVED") {
  await db.users.update(userId, { verificationId: result.verification_id, licenseVerified: true });
} else {
  const check = result.checks.find(c => c.type === "credential");
  throw new VerificationError(check?.error?.code ?? "unknown");
}
```

| `error.code` | Meaning |
|---|---|
| `license_not_found` | No license matching that number + state + type |
| `license_expired` | Found, past expiry |
| `license_inactive` | Suspended, revoked, or lapsed |
| `name_mismatch` | Valid, but registered to a different person |
| `board_unavailable` | Board unreachable — retry in a few minutes |

Surface a friendly message; **never forward raw check data or error codes to the frontend.**

> The response shape shown on this recipe page (`verification_id`, top-level `status: "APPROVED"`,
> `checks[]`) differs from the Core APIs envelope (`{ success, data: { status: "passed", check } }`).
> See `gotchas-and-doc-conflicts.md` — read the actual response and code to what you get.

**Errors:** timeout (client timeout < board response time — set ≥90 s) · `400 invalid_license_type`
(unsupported for that state — check the providers endpoint) · `401` (key missing/wrong, or wrong
environment).

---

## 3. Electronic Visit Verification (EVV) — Account mode

Prove the **right clinician** reached the **right home**: verified identity + live medical license +
face match + geolocation. Built on Account (Managed) identity, so clinicians **verify once and reuse
everywhere**. Live demo: `https://homehealth.valyd.work`.

**What a visit proves**

| Check | Source |
|---|---|
| Identity (KYC) | Verified once on the clinician's Valyd account; reused after, never re-done |
| Medical license | Checked live against the state board, stored on the account, re-checked over time |
| Face at the door | A live selfie matched against the stored Valyd face vector |
| Location | A real GPS fix is **mandatory**. Inside the geofence → passed; outside → failed. |

**The flow:** Connect Valyd (OAuth) → create session with the token → capture → read decision.

**Credentials** — one console. OAuth app (`client_id`, `client_secret`, redirect URI, scopes
`profile`, `verifications`, `doctor_license`), plus a Verify project (API key `vrf_…`, webhook
secret) and a **"Home Health · EVV"** workflow (ID + liveness + face_match + credential + location)
→ `workflow_id`.

### Flow A — Hosted

Valyd renders the UI. Already-verified clinicians skip KYC + license and do **only the face scan**.

```js
import { Valyd } from "@valyd/sdk";
const valyd = new Valyd({
  clientId, clientSecret, apiKey, webhookSecret,
  env: "development",              // -> idp.valyd.work. WITHOUT this the SDK targets production.
});

// 1) "Connect Valyd" — log the clinician in
app.get("/evv/login", (req, res) =>
  res.redirect(valyd.auth.getAuthorizationUrl({
    scope: ["profile", "verifications", "doctor_license"],
  })));

// 2) On callback, create a hosted EVV session bound to their Valyd identity
app.get("/evv/callback", async (req, res) => {
  const { accessToken, user } = await valyd.auth.exchangeCode(req.query.code);
  const session = await valyd.verify.sessions.create({
    workflowId: EVV_WORKFLOW_ID,
    valydAccessToken: accessToken,                                  // <- identifies the person
    vendorData: user.valyd_id,
    metadata: { expected_lat: home.lat, expected_lng: home.lng },   // the assigned home
  });
  res.json({ url: session.url });
});

// 3) Read the result server-side (source of truth)
app.post("/webhooks/valyd", express.raw({ type: "*/*" }), async (req, res) => {
  const event = valyd.verify.webhooks.constructEvent(req.body, req.headers);
  const decision = await valyd.verify.sessions.decision(event.sessionId);
  if (decision.status === "APPROVED") markVisitVerified(event.vendorData, decision);
  res.json({ ok: true });
});
```

Browser side is just links and a redirect — there is no browser SDK.

### Flow B — Core APIs

Your own UI. Still starts with Connect Valyd (to get the token). KYC is a redirect to Valyd; license
and face+location are Core API calls.

```js
app.get("/evv/callback", async (req, res) => {
  const { accessToken, user } = await valyd.auth.exchangeCode(req.query.code);

  // KYC gate — there is NO KYC API; redirect if the account isn't verified
  const v = await valyd.auth.getVerifications(accessToken);
  if (valyd.verify.kyc.isRequired(v))
    return res.redirect(valyd.verify.kyc.redirectUrl({ returnTo: "https://app/evv" }));

  // License — state + type only; provider resolved, name comes from the account
  await valyd.verify.standalone.credentialVerification({
    licenseState: "CO", licenseType: "MD", licenseNumber: "TL.0011377",
  });
  res.json({ ok: true });
});

// The visit itself — ACCOUNT mode: selfie only + location. No ID image.
const face = await valyd.verify.standalone.faceMatch({
  valydAccessToken: accessToken,
  selfie: v.selfie,
});

const loc = await valyd.verify.standalone.locationMatch({
  latitude: v.latitude, longitude: v.longitude, accuracy: v.accuracy,
  expectedLatitude: home.lat, expectedLongitude: home.lng, radiusM: 200,
});
// A radius was given, so the STATUS is the verdict:
//   "passed" -> inside the geofence (loc.check.data.match === true)
//   "failed" -> too far, loc.check.error explains by how much
const atTheHome = face.check.status === "passed" && loc.check.status === "passed";
```

Browser capture without an SDK:

```js
const stream = await navigator.mediaDevices.getUserMedia({ video: { facingMode: "user" } });
// draw a video frame to a <canvas>, then canvas.toDataURL("image/jpeg") -> selfie
const pos = await new Promise((res, rej) =>
  navigator.geolocation.getCurrentPosition(res, rej, { enableHighAccuracy: true }));
// POST { selfie, latitude, longitude, accuracy } to YOUR server, which calls
// valyd.verify.standalone.* with your API key.
```

### Reading the result

**Webhooks are optional.** Both paths end at `sessions.decision(id)`. Option A: poll the decision
when the user returns. Option B: webhook (fires even if the user closes the tab).

```json
{
  "session_id": "ses_8f...",
  "status":     "APPROVED",
  "vendor_data":"valyd_225c7f2ac450496f97bbbc57354a5898",
  "valyd_id":   "valyd_225c7f2ac450496f97bbbc57354a5898",
  "checks": [
    { "type": "id_verification", "status": "passed", "data": { "reused": true } },
    { "type": "liveness",        "status": "passed" },
    { "type": "face_match",      "status": "passed", "score": 0.98 },
    { "type": "credential",      "status": "passed", "data": { "license": { "status": "active" } } },
    { "type": "location",        "status": "passed", "data": { "distance_m": 12, "radius_m": 200, "match": true } }
  ],
  "identity": {
    "full_name": "Grace Lee Casado",
    "licenses": [ { "license_state": "CO", "status": "active", "expire_date": "2027-01-01" } ]
  },
  "decided_at": "2026-07-01T18:04:10Z"
}
```

`reused: true` marks steps skipped from the Valyd account.

### EVV rules worth repeating

- `new Valyd({...})` generates nothing — it only holds config. **`env` picks the environment.**
- The Valyd token goes in the **session**, not the workflow. It identifies the person (`valyd_id`)
  and unlocks KYC/license reuse.
- **KYC is not an API.** `kyc.isRequired(v)` → `kyc.redirectUrl({ returnTo })`.
- Expected home location is passed **per session** via `metadata.expected_lat` / `expected_lng`
  (+ `radius_m`).
- **Location can never be skipped.** A blocked permission or missing coordinates is a hard `failed`.
  With an expected point **and** `radius_m`, the status **is** the verdict. Do not treat location as
  report-only or always-passing.
- **KYC and license are one-time onboarding steps.** Do not put a KYC/license button on every visit.
  The recurring action is face match + location.
- **Account face = selfie only**, matched to the stored vector. Never ask for an ID/reference image.

---

## 4. Anti-spoof / liveness demo

Log a user in with Valyd, then run a liveness check that flags photos, masks, replays and deepfakes.

**How the hosted check is graded.** Valyd tracks the face, records a short live burst, and asks for
one **random** action (turn head, open mouth, nod, move closer). Every frame is scored for passive
liveness and combined into a single `human_score` (0–100), together with motion analysis, same-face
consistency across frames, and verification that the requested action actually happened. This
active, capture-controlled path returns the strongest assurance level, `captured` — an attacker
can't know the action in advance, which is what makes a photo, screen replay, or pre-recorded clip
fail.

**The flow**

1. `POST /api/v2/session` with `{ workflow_id, vendor_data }` → returns a hosted `url`
2. Open the hosted url — the user does the live face capture there
3. Poll `GET /api/v2/session/{session_id}/decision` → status + decision

`APPROVED` → a real, live person. Any other terminal status → not verified (spoof suspected,
abandoned, or expired).

```bash
curl -X POST "https://idp.valyd.work/api/v2/session" \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $VALYD_API_KEY" \
  -d '{ "workflow_id": "<liveness workflow>", "vendor_data": "antispoof-demo-123" }'

curl "https://idp.valyd.work/api/v2/session/SESSION_ID/decision" -H "X-API-Key: $VALYD_API_KEY"
```

Node server sketch:

```js
import { ValydClient } from "@valyd/sdk";
const idp = new ValydClient({ clientId, clientSecret, redirectUri, baseUrl: "https://idp.valyd.work" });

app.get("/login", (_req, res) =>
  res.redirect(idp.getAuthorizationUrl({ scope: ["profile", "verifications"] })));

app.get("/callback", async (req, res) => {
  const { accessToken } = await idp.exchangeCode(String(req.query.code));
  const me = await idp.getUserInfo(accessToken);
  const u = me.data;
  res.json({ valyd_id: u.valyd_id, name: u.full_name, id_verified: u.id_verified });
});

app.get("/verify-face", async (_req, res) => {
  const r = await fetch(IDP + "/api/v2/session", {
    method: "POST",
    headers: { "Content-Type": "application/json", "X-API-Key": APP_KEY },
    body: JSON.stringify({ workflow_id: WORKFLOW_ID, vendor_data: "demo-" + Date.now() }),
  });
  const { data } = await r.json();
  res.json({ sessionId: data.session_id, open: data.url });
});

app.get("/result/:sessionId", async (req, res) => {
  const r = await fetch(`${IDP}/api/v2/session/${req.params.sessionId}/decision`,
    { headers: { "X-API-Key": APP_KEY } });
  const { data } = await r.json();
  res.json({ status: data.status, human: data.status === "APPROVED" });
});
```

> The published demo page carries **public demo credentials** on a balance-capped account so anyone
> can run it. Never copy those into a real integration — create your own at
> `https://dev.valyd.work`. In your own app the `client_secret` and app key stay on your server.

**Want to know *who* the live person is?** `POST /api/v2/antispoof/identity` runs the same pipeline
and, only when it passes, resolves the face against the global Valyd gallery, returning a stable
`valyd_` uuid. The same face always maps to the same uuid — sybil / duplicate-account detection in
one request. See `verify-core-apis.md`.
