# Developer Portal, credentials & accounts

**Everything here is a human browser action at `https://dev.valyd.work`. There is no API.** When an
integration needs any of these values, ask the human for them.

## One console issues everything

| What you get | Where in the portal |
|---|---|
| `client_id` + `client_secret` — Login with Valyd (OAuth2 / OIDC) | your app → Credentials |
| Verify **API key** (`vrf_…`, shown once) | Verify projects |
| Webhook signing secret (`whsec_…`, rotatable) | Verify projects → Webhooks |
| `workflow_id` — create / edit / delete workflows | Verify projects → Workflows |
| Balance, top-ups, session history, webhook delivery log | portal home / app → Verification |

There is **no separate Verify dashboard**. One app carries both identities: the OAuth pair for login
and the API key for verification. Create several apps (Test, Production) rather than sharing one.

## Creating a project

Sign-up needs only a basic Valyd account — **no KYC required** to use the portal.

**Step 1.** Open `https://dev.valyd.work` and sign in.

**Step 2.** "Create Project", then fill in:

| Field | Required | Notes |
|---|---|---|
| Project Name | yes | Shown to users on the consent screen |
| Description | no | Helps users understand what they are authorizing |
| Allowed Web Origins | yes | Scheme + host of every domain you call from, e.g. `https://myapp.com, https://staging.myapp.com` |
| Redirect URL | yes | **Must NOT end with a trailing slash.** `https://myapp.com/callback` |
| Allowed Scopes | yes | See below |

Scope groups offered in the portal:

- **`profile`** (required) — core biometrics. `Face Vector` and `Face Match` are always enabled and
  cannot be turned off. Optional KYC fields: `Name`, `Age`, `Portrait`.
- **`verifications`** — `ID Verification` (government ID) and `Licenses` (professional / driver).
- **`zkp`** — `Age Verification` (prove age without birthdate) and `Country Verification`.

**Step 3.** Copy the credentials from the modal.

```text
Client ID     — 32 hex chars, e.g. 9357c59bc1794b4c9efe8823e5878147; viewable any time
Client Secret — e.g. sk_live_a1b2c3d4...; SHOWN ONLY ONCE
```

```bash
# .env — server-side only, never committed
VALYD_CLIENT_ID=9357c59bc1794b4c9efe8823e5878147
VALYD_CLIENT_SECRET=sk_live_a1b2c3d4e5f6g7h8i9j0...
VALYD_REDIRECT_URI=https://myapp.com/callback
```

Lost the secret? Regenerate it from project settings. **Rotating a secret kills the old one
immediately** — deploy the new value first.

## Verification credentials checklist

```text
No App API key yet?
  -> Sign in at https://dev.valyd.work, open your App, copy the key shown once at creation.
Lost the API key?
  -> Rotate it in the Console to generate a new one.
Using Hosted mode?
  -> Create a Workflow bundling the services you need; copy its workflow_id.
  -> Under Webhooks, set the endpoint URL and copy the signing secret.
Using Core APIs only?
  -> The App API key alone is enough. Workflows and Webhooks are not needed.
```

## Developer sign-in is passwordless

**There is no password. Never look for one, store one, or send one.** Two ways in:

1. **Magic link (email)** — enter your email, open the one-time link. The usual path.
2. **Face — "Login with Valyd"** — available once a verified Valyd identity is linked to the account.

Workforce **members** are different again: they activate and sign in **by face only**, from an
invite. See `organizations-members.md`.

### Linking a face to an email-only account

1. Sign in by magic link, open **Profile**.
2. Under your identity, choose **Connect your Valyd ID**.
3. Complete Login with Valyd (face). The identity attaches to the account you are already in.

Linking **requires an authenticated session** by design — you cannot attach a face to an account you
are not signed in to. This is unrelated to **Connected apps** on an end-user's Valyd account.

### One identity, several console accounts

A single Valyd identity can own multiple Developer Portal accounts — typically one per company.

- If your email or face maps to more than one, sign-in shows an **account picker**.
- Switch later with **Switch account** (`/switch`) — no re-scan, no new magic link.
- Kept **separate per account**: projects/apps and their credentials, Verify workflows and API keys,
  billing, team and members.

At the identity layer it is still **one face = one Valyd identity**. "Multiple accounts" is a portal
concept. To use one identity on multiple devices, pair each device — that extends the identity, it
does not create an account.

## Personal apps vs organization apps

- **Personal apps** live under your individual account; any signed-in developer can create them.
- **Organization apps** belong to a shared tenant and are role-gated (owner / admin / developer).
  Ownership stays with the org when a person leaves.

## Automating for CI

You cannot. Portal sign-in, linking and switching are human steps. For server-to-server automation
use an app's `client_id`/`client_secret` (Login, Members API) or a project API key (Verification).

## Common errors

| Symptom | Cause | Fix |
|---|---|---|
| Redirect rejected / mismatch at authorize | Registered Redirect URL has a trailing slash, or differs from what you send | Register with no trailing slash; match character for character |
| Lost the Client Secret | Shown once, modal closed | Regenerate in project settings, then update `.env` |
| Requests blocked from your domain | Origin missing from Allowed Web Origins | Add the exact scheme + host |
| Scope fails before the consent screen even renders | Scope not enabled for the project | Enable it in the portal first |
