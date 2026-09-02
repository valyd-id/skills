# Environments & credentials

Valyd runs the same product on independent environments. **These references describe the
development environment (`*.valyd.id`).** Each environment has its own hosts, its own Developer
Portal, and its own credentials — an app registered in one environment does not exist in another.

## Hosts per environment

Every environment exposes the same three hosts under a different domain:

| Role | Development | What it is |
| --- | --- | --- |
| **API** (`idp`) | `https://idp.valyd.id` | The API your app calls — verification, the Account API, and sign-in (OAuth 2.0 / OIDC) |
| **Developer Portal** (`dev`) | `https://dev.valyd.id` | Where a human creates apps, gets keys, and composes workflows. No API automates app creation. |
| **Docs** (`docs`) | `https://docs.valyd.id` | Docs, the API Playground, and live demos |

Production and testing mirror this layout on their own domains. Point your integration at the API
host for the environment you registered your app in, and **never mix a key from one environment
with the host of another.**

## Credential formats

Everything comes from the Developer Portal for the matching environment.

| Credential | Format | Issued for | Used as |
| --- | --- | --- | --- |
| Client ID | `9357c59b…` | Connect with Valyd (OAuth 2.0 / OIDC) | `client_id` in authorize + token requests |
| Client Secret | `sk_live_…` (shown once) | Connect with Valyd | `client_secret` in the server-side token exchange |
| App API key | `vrf_…` (shown once) | Verification API | the `X-API-Key` header on every verification call |
| Webhook signing secret | `whsec_…` | Verification sessions | verifying the HMAC on incoming webhooks |
| Workflow ID | `wf_…` | Verification sessions | the `workflowId` you pass to `sessions.create()` |

**The two credential families are independent.** The App API key never authenticates Connect, and
`client_secret` never authenticates verification calls. Connect-only integrations hold the
`client_id` + `client_secret`; verification integrations hold the App API key (plus a workflow ID
and webhook secret for sessions). A full Reusable Verification build holds both.

## Environment variables

The quickstarts and the SDK read these from your server environment. Keep a separate `.env` per
environment and never expose a secret in frontend code.

```bash
# .env (server-side only)
VALYD_IDP_URL=https://idp.valyd.id      # the API host for THIS environment

# Connect with Valyd (OAuth 2.0 / OIDC)
VALYD_CLIENT_ID=9357c59bc1794b4c9efe8823e5878147
VALYD_CLIENT_SECRET=sk_live_a1b2c3d4e5f6g7h8i9j0...
VALYD_REDIRECT_URI=https://yourapp.com/callback   # exact registered callback, no trailing slash

# Verification API (independent of the two values above)
VALYD_API_KEY=vrf_...
VALYD_WEBHOOK_SECRET=whsec_...            # verification sessions only
VALYD_WORKFLOW_ID=wf_...                  # verification sessions only
```

**Setting `VALYD_IDP_URL` per environment is how the same code deploys everywhere:** change the
host, supply that environment's credentials, and nothing else moves. In the SDK this is the
`baseUrl` constructor option, which defaults to `https://idp.valyd.id`.

> Some older material (notably the EVV page) shows an `env: "development"` constructor option on a
> `Valyd({ ... })` client. The current documented mechanism is `VALYD_IDP_URL` / `baseUrl`. Prefer
> that; see [`gotchas-and-doc-conflicts.md`](gotchas-and-doc-conflicts.md).

## Separate test and production apps

Even inside one environment, create **separate apps** — for example "Test" and "Production" — so
each has its own `client_id` / `client_secret`, App API key, workflows, and webhook endpoint. You
can rotate or revoke the test app's key without touching production.

Keep separate **workflows** per app too, rather than mutating one in place: a session runs the
checks of the workflow as it was when the session was created.

## Testing costs real money

Verification runs for real — there are no simulated results. Checks are billed against your app's
wallet. New accounts start with a **$100 welcome credit**.

This is why [reading the account first](account-api.md) matters: a read is free and instant, a
check is not.

## Related

- [`portal-and-accounts.md`](portal-and-accounts.md) — the portal walkthrough and passwordless sign-in
- [`products.md`](products.md) — which credentials each product needs
