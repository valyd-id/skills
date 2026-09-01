# OAuth2 scopes

Scopes are space-separated on the authorization request. Users see them on the consent screen, in
plain language. **Every scope must first be enabled for your app in the Developer Portal** —
requesting one that isn't enabled fails authorization before the consent screen even renders.

The SDK adds the required `openid` scope automatically.

## Summary

| Scope | Required | Description | Grants access to |
| --- | --- | --- | --- |
| `profile` | **Mandatory** | Legal name, username, country, and verification status. No photo is shared. | `/userinfo` |
| `verifications` | optional | Liveness (human) check, ID/KYC status, and linked professional licenses | `/verifications` |
| `doctor_license` | optional | Medical / nursing license details for verified practitioners | doctor/nursing license endpoints |
| `zkp` | optional | Zero-knowledge age proofs | ZKP endpoints |
| `mcp` | optional | Model Context Protocol endpoints | MCP endpoints |

OIDC discovery advertises `openid`, `profile`, `email`, `verifications`, `zkp`. The MCP server
requires `openid mcp` — see [`mcp.md`](mcp.md).

## Requesting them

```javascript
const transaction = valyd.auth.createAuthorizationRequest({
  scope: ["profile", "verifications", "zkp"],
});

req.session.valydOidc = transaction;   // keep server-side
res.redirect(transaction.url);
```

`transaction.url` is a standard OIDC authorization URL containing `openid`, the requested scopes,
`state`, `nonce`, and an S256 PKCE challenge.

## Enforcement

```text
Authorization fails immediately, before the consent screen
  -> the scope is not enabled for your app. Enable it in the portal.
A scoped endpoint returns 403 with code "insufficient_scope"
  -> the token lacks that scope. Add it and re-authenticate the user.
Unsure which scopes a token carries
  -> re-run the flow and confirm the requested scopes match what the endpoint needs.
```

```json
{
  "success": false,
  "error": { "code": "insufficient_scope", "message": "The request requires the profile scope" }
}
```

## What each scope returns

### `profile` (mandatory) — `GET /userinfo`

| Field | Description |
| --- | --- |
| `sub` | Unique user identifier — the stable `valyd_…` id |
| `valyd_id` | Same value as `sub` |
| `preferred_username` | Pseudonymous username |
| `email` | Email address |
| `name` | Legal name |
| `id_verified` | boolean — KYC complete |

Relying parties receive the user's **real legal name**, not a pseudonym.

### `verifications` — `GET /verifications`

| Field | Description |
| --- | --- |
| `human_verified` | Passed a liveness / anti-spoof check. Falls back to `id_verified` when no explicit human check is on file. |
| `id_verified` | Completed KYC document verification |
| `licenses[]` | `license_type`, `verified`, `verified_from`, `expire_at` |

### `doctor_license`

Medical / nursing license details for verified healthcare practitioners, via the SDK helpers
`getDoctorLicense`, `getLicenses`, `getCprLicense`. Request it only when the user is expected to
hold such a license.

### `zkp`

Age proofs without revealing a birthdate: `is_18`, `is_21`, `is_25` — booleans.

### `mcp`

Lets AI agents retrieve the user's authorized identity and verification context through the MCP
interface, on the user's behalf.

## Related

- [`account-api.md`](account-api.md) — the endpoints these scopes unlock
- [`consent-attributes.md`](consent-attributes.md) — raw attributes, which scopes never grant
