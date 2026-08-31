# OAuth2 scopes

Scopes are space-separated and URL-encoded on the authorization URL. Users see them on the consent
screen. **Every scope must first be enabled for your project in the Developer Portal** — requesting
one that isn't enabled fails authorization before the consent screen even renders.

## Summary

| Scope | Required | Grants | Description |
|---|---|---|---|
| `profile` | **Mandatory** | `/userinfo` | Name and age-verification status. No photo is shared. |
| `verifications` | optional | `/verifications` | Liveness (human) check, KYC status, linked licenses |
| `doctor_license` | optional | doctor/nursing license endpoints | Medical / nursing license details |
| `zkp` | optional | ZKP endpoints | Zero-knowledge age proofs |
| `mcp` | optional | MCP endpoints | Lets an agent act on the user's identity |

OIDC advertises a different set — `openid`, `profile`, `email`, `verifications`, `zkp`. See `oidc.md`.
The MCP server requires `openid mcp`. See `mcp.md`.

## Requesting them

```js
const scopes = "profile verifications zkp";       // space-separated
const authURL =
  `https://idp.valyd.work/auth?client_id=${clientId}` +
  `&redirect_url=${redirectUrl}` +
  `&scope=${encodeURIComponent(scopes)}`;
// -> scope=profile%20verifications%20zkp
```

With the SDK, pass an array: `getAuthorizationUrl({ scope: ["profile", "verifications", "zkp"] })`.

## Enforcement

```text
Authorization fails immediately, before the consent screen
  -> the scope is not enabled for your project. Enable it in the portal.
A scoped endpoint returns 403 with code "insufficient_scope"
  -> the token lacks that scope. Add it to the authorize URL and re-authenticate the user.
Unsure which scopes a token carries
  -> re-run the flow and confirm the requested `scope` matches what the endpoint needs.
```

Missing-scope responses all look like this:

```json
{
  "success": false,
  "error": { "code": "insufficient_scope", "message": "The request requires the profile scope" }
}
```

## What each scope returns

### `profile` (mandatory) — `GET /userinfo`

| Field | Description |
|---|---|
| `sub` | unique user identifier |
| `email` | email address |
| `first_name` / `last_name` / `full_name` | name |
| `valyd_id` | the user's Valyd account identifier |
| `id_verified` | boolean — KYC complete |
| `created_at` | account creation timestamp |

Every relying party now receives the user's **real legal name**, not the pseudonym.

### `verifications` — `GET /verifications`

| Field | Description |
|---|---|
| `human_verified` | passed a liveness / anti-spoof check. Falls back to `id_verified` when no explicit human check is on file. |
| `id_verified` | completed KYC document verification |
| `licenses[]` | `license_type`, `verified`, `verified_from`, `expire_at` |

### `doctor_license`

Medical / nursing license details for verified healthcare practitioners. Reached through the SDK
helpers `getDoctorLicense`, `getLicenses`, `getCprLicense`. Request it only when the user is
expected to hold such a license.

### `zkp`

Age proofs without revealing a birthdate: `is_18`, `is_21`, `is_25` — booleans.

### `mcp`

Lets AI agents and tools retrieve the user's authorized identity and verification context through
the MCP interface, on the user's behalf. Response carries `tools` (what the agent may call) and
`context` (the identity/verification data exposed).
