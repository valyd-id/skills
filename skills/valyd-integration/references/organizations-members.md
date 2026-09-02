# Organizations, teams & the Members API

Every Valyd account has its own **personal apps** and can create or join **any number of
organizations** — there is no "individual vs company" account type. An organization is a shared
tenant: a team, roles, a face-verified workforce, and public or private apps, governed by one
billing account.

**Note the different host:** the Members API lives on **`https://dev.valyd.id/api/sdk/*`**, not
`idp.valyd.id`.

## Roles

- **Owner / Admin** — sees and manages everything: the organization, team, members, apps and
  billing. Can assign roles and add people (admin, developer, or member).
- **Developer** — signs in, sees the organizations they belong to, creates and manages apps. No
  member or billing administration.
- **Member** — the workforce. Does **not** see the organization at all; a member exists only to log
  into the apps assigned to them, **by face**.

## What an organization gives you

- **Teams & roles** — clear separation between who builds and who administers.
- **Shared apps** — apps belong to the organization, not one person. Ownership does not leave when
  a person does.
- **Workforce by face** — add members by CSV, singly, or via the API. Each gets a link and joins by
  scanning their face; no passwords. **Only `active` (face-activated) members are billable.**
- **Public & private apps** — a **public** app lets anyone log in (the default). A **private** app is
  scoped to specific members; only assigned members can sign in, enforced at the login gate.
- **One billing account** — pay-as-you-go usage and per-seat subscriptions post to a single wallet
  and ledger.

## The Members API

Authenticated with your app's `client_id` + `client_secret` — headers `X-Client-Id` /
`X-Client-Secret`, or HTTP Basic — and scoped to the organization that owns the app. **Server-side
only.**

| Operation | SDK | REST |
| --- | --- | --- |
| List members (status + `valyd_id`) | `client.getMembers()` | `GET /api/sdk/members` |
| Add a member (emails a face-activation link) | `client.addMembers([{ email, firstName, lastName }])` | `POST /api/sdk/members` |
| Add many (bulk, up to 500; dupes → `skipped`) | `client.addMembers([ ...up to 500 ])` | `POST /api/sdk/members` |
| Invite silently (no email; returns each `activationLink`) | `client.addMembers([...], { notify: false })` | `POST /api/sdk/members` with `notify:false` |
| Look up one person (role + status, at any role) | `client.resolveMember({ valydId })` / `{ email }` | `POST /api/sdk/members/resolve` |
| Deactivate (revoke app login; recoverable) | `client.deactivateMember(memberId)` | `PATCH /api/sdk/members/{memberId}/deactivate` |
| Remove (default = deactivate; `permanent` deletes the membership) | `client.removeMember(memberId, { permanent: true })` | `DELETE /api/sdk/members/{memberId}?permanent=true` |
| Reactivate | `client.reactivateMember(memberId)` | `PATCH /api/sdk/members/{memberId}/reactivate` |
| Re-send an activation invite | `client.resendMemberInvite(memberId)` | `POST /api/sdk/members/{memberId}/invite` |
| Billing & seats | `client.getBilling()` | `GET /api/sdk/billing` |

```ts
import { ValydClient } from "@valyd/sdk";

// Server-side only — the client secret never touches the browser.
const client = new ValydClient({
  clientId: process.env.VALYD_CLIENT_ID,
  clientSecret: process.env.VALYD_CLIENT_SECRET,
});

// Add members — Valyd emails each a face-activation link.
const { created, skipped } = await client.addMembers([
  { email: "jane@acme.com", firstName: "Jane", lastName: "Doe" },
  { email: "sam@acme.com" },
]);

// List the roster with status + valyd_id.
const members = await client.getMembers();
// [{ id, firstName, lastName, email, status, valydId, active }]
const activated = members.filter((m) => m.status === "active");

// Seats + wallet.
const billing = await client.getBilling();   // { seats, pricePerSeat, balance, ... }
```

Raw REST:

```bash
curl -X POST https://dev.valyd.id/api/sdk/members \
  -H "X-Client-Id: $VALYD_CLIENT_ID" \
  -H "X-Client-Secret: $VALYD_CLIENT_SECRET" \
  -H "Content-Type: application/json" \
  -d '{"members":[{"email":"jane@acme.com","first_name":"Jane","last_name":"Doe"}],"notify":true}'
# -> { "success": true, "data": { "created": [...], "skipped": [], "notified": true } }
```

## Member lifecycle

| Status | Meaning |
| --- | --- |
| `invited` | created, no email sent yet (the `notify:false` path) |
| `link_sent` | activation email sent, awaiting the person |
| `active` | face-activated and bound to a Valyd identity — **the only billable state** |
| `deactivated` | removed from the workforce; not billable |

- `deactivateMember` / `reactivateMember` accept the member's `memberId` (`vmem_…`), their Valyd ID,
  or their email.
- `removeMember(memberId)` is the same deactivation **by default**; pass `{ permanent: true }` to
  delete the membership row outright — the seat and history go, and the email can be re-invited
  cleanly.
- `reactivateMember` restores a deactivated member to `active` (or `invited` if they never
  activated).
- `resendMemberInvite(memberId)` issues and emails a fresh activation link (and returns it),
  superseding the old one. It **refuses** for already-active or deactivated members.
- **None of these ever touch the person's Valyd identity** or their membership in any other org.
- Only `member`-role people are affected. Use `resolveMember({ valydId })` to tell a workforce
  member apart from a developer/admin, or from someone not in your org at all.
- **Result sync is by polling `getMembers()`.** There is no member webhook.

In the portal, the owner/admin sees the full roster with each member's status on
**Organization → Members**, and can re-send invites, deactivate/reactivate, or **Remove** outright
(permanent, same as the API's `permanent: true`).

## What is portal-only

Creating an organization, inviting teammates (developer/admin roles), CSV upload, deactivation via
the UI, and private-app member assignment are all human web steps. Adding/listing members and
reading billing **are** a real API.

## Getting started

1. Sign in at `https://dev.valyd.id` and open the **Organizations** tab.
2. Create an organization from the selector — you become its owner.
3. Invite teammates (developer or admin) and create apps under the organization.
4. Add members (your workforce) by CSV, singly, or with `addMembers()`; they join by scanning their
   face.
5. Mark apps public or private, and assign members to the private ones.

## Notes for integrators

- Organizations **do not change the login or verification API surface**. Your app still uses the
  same OAuth `client_id`/`client_secret` for Connect with Valyd and the same App API key for the
  Verification APIs. An organization governs *who owns the app*, *who may log into it*, and gives
  you the Members API for workforce onboarding.
- For a **private** org app, a user who is not an assigned member is **refused at the OAuth authorize
  step**. Public apps behave exactly as before.
