# Developer Portal, credentials & accounts

**Everything here is a human browser action at `https://dev.valyd.id`. There is no API.** When an
integration needs any of these values, ask the human for them.

Each environment has its own portal and its own credentials — see
[`environments.md`](environments.md).

## One console issues everything

| What you get | Where in the portal |
| --- | --- |
| `client_id` + `client_secret` — Connect with Valyd (OAuth 2.0 / OIDC) | your app → Credentials |
| App API key (`vrf_…`, shown once) | your project → Verification |
| Webhook signing secret (`whsec_…`, rotatable) | your project → Verification → Webhooks |
| `workflowId` — compose and edit workflows | your project → Verification → Workflows |
| Balance, top-ups, session history, webhook delivery log | portal home / app → Verification |

There is **no separate Verify dashboard**. One app carries both identities: the OAuth pair for
Connect, and the API key for verification.

## Creating an app

Sign-up needs only a basic Valyd account — **no KYC required** to use the portal. New accounts
start with a **$100 welcome credit**.

**Step 1.** Open `https://dev.valyd.id` and sign in.

**Step 2.** Create an app and fill in:

| Field | Required | Notes |
| --- | --- | --- |
| App Name | yes | Shown to users on the consent screen |
| Description | no | Helps users understand what they're authorizing |
| Allowed Web Origins | yes | Scheme + host of every domain you call from |
| Redirect URL | yes | **Must NOT end with a trailing slash.** `https://myapp.com/callback` |
| Allowed Scopes | yes | See [`scopes.md`](scopes.md) |

If you use RP-initiated logout, register your `post_logout_redirect_uri` as an **additional
redirect URI**.

Scope groups offered in the portal:

- **`profile`** (required) — core biometrics. `Face Vector` and `Face Match` are always enabled and
  cannot be turned off. Optional KYC fields: `Name`, `Age`, `Portrait`.
- **`verifications`** — `ID Verification` and `Licenses`.
- **`zkp`** — `Age Verification` (prove age without a birthdate) and `Country Verification`.

**Step 3.** Copy the credentials from the modal.

```text
Client ID     — 32 hex chars, e.g. 9357c59bc1794b4c9efe8823e5878147; viewable any time
Client Secret — sk_live_…; SHOWN ONLY ONCE
```

The app's **Quick setup** page shows the setup checklist, client credentials, and OIDC endpoints in
one place.

Lost the secret? Regenerate it from app settings. **Rotating a secret kills the old one
immediately** — deploy the new value first.

## Verification credentials checklist

```text
No App API key yet?
  -> Sign in, open your project's Verification tab, copy the key shown once at creation.
Lost the API key?
  -> Rotate it in the portal to generate a new one.
Running any verification?
  -> Compose a workflow (Verification -> Workflows) and copy its workflowId.
Want push results?
  -> Under Webhooks, set the endpoint URL and copy the signing secret.
     (Optional — polling sessions.decision(id) is a valid design.)
```

Workflows are **portal-only** — the SDK has no workflow CRUD. See [`workflows.md`](workflows.md).

## Developer sign-in is passwordless

**There is no password. Never look for one, store one, or send one.** Two ways in:

1. **Magic link (email)** — enter your email, open the one-time link. The usual path.
2. **Face — Connect with Valyd** — once a verified Valyd identity is linked to the account.

Workforce **members** are different: they activate and sign in **by face only**, from an invite.
See [`organizations-members.md`](organizations-members.md).

### Linking a face to an email-only account

1. Sign in by magic link, open **Profile**.
2. Under your identity, choose **Connect your Valyd ID**.
3. Complete Connect with Valyd (face). The identity attaches to the account you are already in.

Linking **requires an authenticated session** by design — you cannot attach a face to an account
you are not signed in to. This is unrelated to **Connected apps** on an end-user's Valyd account.

### One identity, several console accounts

A single Valyd identity can own multiple portal accounts — typically one per company.

- If your email or face maps to more than one, sign-in shows an **account picker**.
- Switch later with **Switch account** (`/switch`) — no re-scan, no new magic link.
- Kept **separate per account**: apps and their credentials, workflows and API keys, billing, team
  and members.

At the identity layer it is still **one face = one Valyd identity**. "Multiple accounts" is a
portal concept. To use one identity on multiple devices, pair each device.

## Automating for CI

You cannot. Portal sign-in, linking and switching are human steps. For server-to-server automation
use an app's `client_id`/`client_secret` (Connect, Members API) or the App API key (verification) —
never a portal login.

## Common errors

| Symptom | Cause | Fix |
| --- | --- | --- |
| Redirect rejected at authorize | Registered URL has a trailing slash, or differs from what you send | Register with no trailing slash; match character for character |
| Lost the Client Secret | Shown once, modal closed | Regenerate in app settings, then update `.env` |
| Requests blocked from your domain | Origin missing from Allowed Web Origins | Add the exact scheme + host |
| Scope fails before the consent screen renders | Scope not enabled for the app | Enable it in the portal first |
| 401 with a key that "should work" | Key is from a different environment | Match host and credentials — see [`environments.md`](environments.md) |
