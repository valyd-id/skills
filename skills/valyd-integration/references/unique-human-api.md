# Unique Human API

> **Auth:** App API key (server-side) + a `workflowId` · **User account:** none · **Result:**
> returned to your system, nothing saved to an account

Determine whether you're interacting with a **live, unique human** using your Valyd API credentials
alone. No user account, no OIDC, no sign-in.

You create a session for a workflow containing the **Liveness** and/or **Uniqueness** checks,
redirect the person to Valyd's verification page (Valyd handles the camera, capture, retries and
security), and read the verdict when they're done.

## The flow

**1. Portal setup.** In the [Developer Portal](https://dev.valyd.work) create a **project**, open
its **Verification** tab, build a workflow with the checks you need, then copy the project's API
key (shown once) and the `workflowId`.

> For a quick, no-account anti-spoof check, every organization also has a built-in **Unique Human
> API key** on the dashboard.

Portal sign-in works with an **email magic link** as well as a Valyd ID.

**2. Create a session — no user token.**

```javascript
import { VerifyClient } from "@valyd/sdk";
const verify = new VerifyClient({ apiKey: process.env.VALYD_API_KEY });

const session = await verify.sessions.create({
  workflowId:  process.env.VALYD_WORKFLOW_ID,  // liveness and/or uniqueness
  redirectUrl: "https://yourapp.com/checked",
  vendorData:  "user-123",                     // your internal ref
});
// -> redirect the person's browser to session.url
```

**3. Valyd captures.** The page records a live camera burst with a random on-screen action for the
strongest assurance, then sends the person back to your `redirectUrl`.

**4. Read the verdict** — signed [webhook](webhooks.md) or the decision call:

```javascript
const decision = await verify.sessions.decision(session.sessionId);
// decision.status: "APPROVED" | "DECLINED" | "IN_REVIEW"
// decision.checks[] — per-check data:
//   antispoof       -> { human_score: 100, assurance: "captured", ... }
//   face_uniqueness -> { valyd_uuid: "valyd_8f2…", registered: "new" | "existing" }
```

Because the session carries no user token, **nothing is saved to a Valyd account** — the verdict is
yours to act on, and there is no proof to reuse later.

---

## The two checks

### Liveness (anti-spoof)

Detects presentation and spoof attacks — printouts, screens, replays, masks. Returns a
vendor-neutral `human_score` (0–100) with a pass/fail verdict, plus per-frame voting, motion
analysis and same-person consistency.

```javascript
// {
//   assurance: "captured",       // live camera burst with a random on-screen action
//   frames_analyzed: 5,
//   frames_genuine: 5,
//   frames_spoof: 0,
//   motion: "natural",
//   face_consistency: "consistent",
//   human_score: 100
// }
```

Failure `signal` values: `no_face`, `face_unreadable`, `spoof_detected`, `low_confidence`,
`duplicate_frames`, `static_capture`, `discontinuous_motion`, `different_faces`.

### Uniqueness (face uniqueness)

One face = one identity. Enrols or matches the captured face against the Valyd face registry and
returns a stable `valyd_uuid` plus whether it was newly registered.

```javascript
// { valyd_uuid: "valyd_8f2…", registered: "existing" }
```

The same face resolves to the same uuid on every run, so a person opening a second account is
caught. Store the `valyd_uuid` against your own user record.

**Pair the two in one workflow** so only a proven-live face is matched against the registry.

---

## Data: this is the one place check data flows to you

| | You | Valyd |
| --- | --- | --- |
| **Inputs** (the face images) | You capture and submit them | Processed **transiently** for that check, then discarded — not retrievable |
| **Results** (status, scores, per-check `data`) | Returned in the response — **yours to store** | A session record of the outcome, not the raw images |
| **Biometrics** | Never receive a face template | Irreversible vectors, never images; never returned |
| **Storage, protection, deletion** | **Your responsibility** | Limited to the above |

### Your responsibilities

- **Store securely** — the images you submit and the check `data` can be sensitive; encrypt at rest
  and keep them out of logs.
- **Limit access** — treat results with the same controls as any biometric record.
- **Delete on schedule** — you own the retention clock.
- **Keep the key server-side** — the App API key never belongs in a browser.

## Deleting a face (GDPR)

To honour a right-to-erasure request, unlink the face by its `valyd_` uuid from the project's
**Verification** settings in the portal, or with `verify.faceUniquenessUnlink(valydUuid)`. The face
is unlinked from your project and fully deleted once no remaining project — and no Valyd account —
still knows it.

## What this product does *not* do

ID/KYC, face match, age, professional license and location are **not** part of the Unique Human
API. They run only as workflow checks in [Reusable Verification](products.md), where the user
connects with Valyd, the passed proof saves to *their* Valyd ID, and the raw identity data stays
encrypted with Valyd.

## Related

- [`products.md`](products.md) — choosing between the two products
- [`verification-sessions.md`](verification-sessions.md) — the shared session mechanics
- [`checks-reference.md`](checks-reference.md) — full check detail
