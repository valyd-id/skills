# Workflows

A **workflow** defines what Valyd needs to verify for your application — a reusable bundle of
checks. You define it once in the Developer Portal and reference its `workflowId` every time you
create a session. Valyd's verification page auto-adapts its steps to the workflow's checks, and all
results come back together in **one decision**.

> **Workflows are created and edited in the Developer Portal only.** `verify.workflows.*` CRUD was
> **removed from the SDK** — there is no create/list/update/remove. Compose the workflow in the
> portal, copy the `workflowId`, and pass it to `sessions.create()`.

## Creating one

Developer Portal → **Workflows** → create from a preset → copy the `workflowId`.

Two presets ship, and both use the **same integration code** — only the `workflowId` differs:

### License Verification — *credential only*

- Checks: `[credential]`
- Flow: state → license type → name + license number → verify
- Fastest path to verify a professional license. No ID scan required.

### KYC + License — *identity + credential*

- Checks: `[id_verification, liveness, face_match, credential]`
- Flow: scan ID + selfie (OCR + liveness + 1:1 face match), then state + license type + number
- The name is taken from the verified ID automatically — **the user never types a name**, so a
  license belonging to a different person is rejected.

```text
Only verify a professional license (no ID scan)   -> the "License Verification" workflowId
Identity + credential (ID scan + selfie + license) -> the "KYC + License" workflowId
Unsure -> open https://dev.valyd.id -> Workflows and copy the workflowId of the preset you created
```

## Using it

```javascript
const session = await verify.sessions.create({
  workflowId:  process.env.VALYD_WORKFLOW_ID,
  redirectUrl: "https://app.example.com/verify/callback",
});
```

The session response's `features` array echoes the workflow's checks. Switching from license-only
to full KYC + license is a **one-variable change**.

## Bundling several checks

One session can run several checks back to back — the person completes them all on one page and you
get **one combined decision**. A common shape is EVV / home-health onboarding, where a caregiver
must prove **who they are, that they're licensed, and where they are** in a single sitting:

- **ID / KYC** — scan a government ID (OCR + authenticity) with a live selfie and 1:1 face match
- **Professional license** — verified against the name on the ID
- **Location** — confirm they're at the visit location

You don't wire these together in code. In the portal's workflow builder you pick the checks and
their order once, and Valyd hands you a single `workflowId`. Then the same one call runs the whole
flow:

```javascript
// The workflow already bundles [id_verification, liveness, face_match, credential, location].
const session = await verify.sessions.create({
  workflowId:  process.env.VALYD_EVV_WORKFLOW_ID,
  redirectUrl: `${process.env.APP_URL}/visits/verified`,
  callback:    `${process.env.APP_URL}/webhooks/valyd`,
  vendorData:  visit.caregiverId,
});
// -> res.redirect(session.url)  — one page, ID -> license -> location, in order
```

One signed webhook fires with one decision covering every check; the decision's `checks` array
carries the per-check breakdown.

## Reuse on connected sessions

When a session is created with the connected user's `valydAccessToken`, the flow **skips steps the
account has already completed** — an already-KYC'd user isn't asked to rescan their ID, and a
returning user re-verifies with a **selfie only**, matched against their stored face vector.

A session created **without** a user's token always runs every check in the workflow.

This is why [reading the account first](account-api.md) matters: the cheapest check is the one you
don't run.

## Changing a workflow

Update a workflow's name or features in the portal. **A session runs the checks of the workflow it
was created with** — create a new session to pick up changes.

Keep separate workflows (and separate apps) for test and production rather than mutating one in
place.

## Related

- [`checks-reference.md`](checks-reference.md) — what each check does and returns
- [`verification-sessions.md`](verification-sessions.md) — the full session flow
- [`unique-human-api.md`](unique-human-api.md) — workflows with no user token
