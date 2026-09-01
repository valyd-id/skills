# The two products

Valyd has exactly two products. Pick one; there is no further decision tree.

```text
Do you need to know only whether someone is a live, unique human,
with no user account and nothing saved?
  YES -> Unique Human API
  NO  -> Reusable Verification
```

> The old "Choosing an integration" decision tree, and the Hosted × Core / Account × Non-account
> 2×2, are **retired**. Ignore them wherever they still appear.

---

## Reusable Verification

**Auth:** App API key (SDK client) + the connected user's `valyd_access_token`
**Result:** proofs save to the user's Valyd ID — reusable next time

The developer story is one journey:

1. **[Connect the user](connect-oidc.md)** — standard OpenID Connect; your backend receives their
   `valyd_access_token`. Connect with Valyd can double as your app's sign-in.
2. **[Read what they already have](account-api.md)** — profile, `id_verified`, verified licenses,
   badges, age bands. If the proof you need is there and fresh, **you're done — no check, no cost.**
3. **Something missing? Run your [workflow](workflows.md)** — the saved bundle of checks you
   configured. [Create a session](verification-sessions.md) with the user's token; Valyd guides the
   user through the capture. A returning user re-verifies with a **selfie only**, matched against
   their stored face vector; already-verified KYC and licenses are skipped.
4. **[Read the result](statuses.md)** — the decision arrives on a signed
   [webhook](webhooks.md) or via `verify.sessions.decision()`. The passed proof lands on the user's
   Valyd ID, so next time step 2 answers instead of step 3.

### The mental model

| Term | Meaning |
| --- | --- |
| **Workflow** | A reusable configuration describing which checks your app requires. Built in the Developer Portal. |
| **Session** | One user's run through a workflow on Valyd's verification page. |
| **Checks** | The individual verifications: ID/KYC, liveness, face match, license, age, location. |
| **Decision** | The authoritative combined outcome: `APPROVED` / `DECLINED` / `IN_REVIEW`. |
| **Proof** | The durable outcome saved to the user's Valyd ID when a check ran with their token. |

Judging freshness: the account's `identity` object carries a `verified_at` timestamp, and a license
badge carries the registry's own `status` and `expires_at`. Re-run a check when your policy needs a
fresher answer.

### The session in code

```typescript
const session = await verify.sessions.create({
  workflowId,                      // the checks you picked in the portal
  valydAccessToken: accessToken,   // <- ties the run to the connected user
  redirectUrl: "https://yourapp.com/verified",
});
// -> send them to session.url — proofs come back, PII doesn't
```

---

## Unique Human API

**Auth:** App API key + a `workflowId`. No user account, no OIDC.
**Result:** returned to your system; nothing saved to any account.

Answers one question: *is this a live, unique human?* You create a session for a workflow
containing the **Liveness** and/or **Uniqueness** checks, redirect the person to Valyd's
verification page, and read the verdict. Because the session carries **no user token**, nothing is
saved to a Valyd account and there is no proof to reuse later.

```javascript
import { VerifyClient } from "@valyd/sdk";
const verify = new VerifyClient({ apiKey: process.env.VALYD_API_KEY });

const session = await verify.sessions.create({
  workflowId:  process.env.VALYD_WORKFLOW_ID,   // liveness and/or uniqueness
  redirectUrl: "https://yourapp.com/checked",
  vendorData:  "user-123",
});
// -> redirect the person to session.url

const decision = await verify.sessions.decision(session.sessionId);
```

Full detail: [`unique-human-api.md`](unique-human-api.md).

---

## Data: who holds what

This is the sharpest difference between the two products.

| | Reusable Verification | Unique Human API |
| --- | --- | --- |
| **What you receive** | The decision plus reusable **proofs** — pseudonym, `id_verified`, license badges, age bands | The check result — status, scores, per-check `data` |
| **Raw identity data** | Stays **encrypted with Valyd** under the user's per-user key. Released only via the [consent flow](consent-attributes.md) | You supply the inputs; results are returned to you |
| **Saved to an account** | Yes — the proof is reusable | No — nothing is written, nothing to reuse |
| **Storage duty of care** | Valyd's | **Yours** — encrypt at rest, restrict access, delete per your policy and local law |

The Unique Human API is the one place where check data flows **to you**. Your responsibilities
there: store securely and keep it out of logs, limit access as you would any biometric record,
delete on your own retention schedule, and keep the App API key server-side.

### Biometrics

**Irreversible vectors, never images.** Valyd does not store or return face images. Enrollment
converts a selfie into a one-way biometric vector (template); every later face match compares
vectors. Photos submitted to a check are processed **transiently** for that check and are not
retrievable afterwards. The template is never exposed through any API.

The ID `portrait` in a KYC result is extracted from the document you submitted in that request —
it is not a stored account photo.

---

## Which checks are available where

| Check | Reusable Verification | Unique Human API |
| --- | --- | --- |
| `id_verification` (KYC) | yes | — |
| `liveness` | yes | — |
| `antispoof` | yes | yes |
| `antispoof/identity` | yes | yes |
| `face-uniqueness` | yes | yes |
| `face_match` | yes | — |
| `age` | yes | — |
| `credential` (license) | yes | — |
| `location` | yes | — |

ID/KYC, face match, age, professional license and location run **only** as workflow checks in
Reusable Verification — never as their own public APIs. Details in
[`checks-reference.md`](checks-reference.md).
