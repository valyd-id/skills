# Valyd MCP server

One hosted server gives an agent two capabilities:

- **Human-in-the-loop verification** — ask the signed-in user to approve an action with a face scan
  on their Valyd app (`verification_request` → `verification_status`).
- **The Valyd web agent** — run a browser/web task on the user's behalf, reusing their secure
  browser profile (`do_task`).

| | |
| --- | --- |
| Endpoint | `https://mcp.valyd.id/verification/mcp` |
| Transport | Streamable HTTP |
| Auth | OAuth 2.1 Bearer token (RFC 9728 discovery) |
| Authorization server | `https://idp.valyd.id` |
| Required scopes | `openid` `mcp` |
| Token audience (`aud`) | `https://mcp.valyd.id` |
| The user | the access-token `sub`; tools act on their behalf |
| Tools | `verification_request`, `verification_status`, `do_task` |

**Use no trailing slash on the URL.** There is nothing to register per request and no shared secret
to paste — a modern MCP client performs the OAuth login once, after which every tool call carries
the user's token automatically.

## Authentication (OAuth 2.1)

The server **does not issue tokens — it only validates them.** The flow is standards-based (OAuth
2.1 + PKCE, RFC 9728 Protected Resource Metadata, RFC 8707 resource indicators), so any
OAuth-capable MCP client connects with zero custom code.

1. The client calls the MCP endpoint with no token, gets **401** with a `WWW-Authenticate` header
   pointing at `https://mcp.valyd.id/.well-known/oauth-protected-resource`.
2. That metadata names the authorization server and scopes:

```json
{
  "resource": "https://mcp.valyd.id",
  "authorization_servers": ["https://idp.valyd.id"],
  "scopes_supported": ["openid", "mcp"],
  "bearer_methods_supported": ["header"]
}
```

3. The client discovers the IdP, registers (Dynamic Client Registration), and opens a browser. The
   user logs into Valyd and consents.
4. The client gets an access token bound to `resource = https://mcp.valyd.id` with scope `mcp`,
   and sends `Authorization: Bearer <access_token>` on every call.

### Token validation — all must hold

| Claim | Requirement |
| --- | --- |
| signature | RS256, verifiable against `https://idp.valyd.id/api/auth/oidc/jwks.json` |
| `iss` | `https://idp.valyd.id` |
| `aud` | `https://mcp.valyd.id` |
| `scope` | must include `mcp` |
| `exp` / `iat` / `sub` | present and not expired |

**Code-first clients** (LangChain, OpenAI SDK, scripts): run the OAuth 2.1 **authorization-code +
PKCE** flow against `https://idp.valyd.id` with `scope=openid mcp` and the resource indicator
`resource=https://mcp.valyd.id`, then pass the token as a Bearer header. The token is
**user-bound**, so it must come from a user login — **not** client-credentials.

> The legacy `X-MCP-Client-Id` / `X-MCP-Client-Secret` / `X-MCP-Webhook-Url` header scheme and the
> `user_id` tool parameter are **removed**. OAuth 2.1 Bearer only; the user comes from the token.

## Tools

The user is the token `sub` — no user id is passed. On failure every tool returns
`{"status": "error", "message": "..."}`.

### `verification_request`

Ask the signed-in user to approve a sensitive action with a face scan. **Call before anything risky**
— delete, payment, sharing data.

| Name | Type | Required | Description |
| --- | --- | --- | --- |
| `action_type` | string | yes | `"delete"`, `"payment"`, `"update"`, … |
| `title` | string | yes | Short title shown on the approval prompt |
| `description` | string | yes | Context the user reads before deciding |

```json
{
  "valyd_session_id": "uuid-string",
  "status": "PENDING",
  "expires_at": "2026-06-24T09:10:46+00:00"
}
```

### `verification_status`

Poll until `status` is no longer `PENDING`.

| Name | Type | Required | Description |
| --- | --- | --- | --- |
| `valyd_session_id` | string | yes | The id from `verification_request` |

```json
{
  "valyd_session_id": "uuid-string",
  "status": "APPROVED",
  "result": "...",
  "assurance_level": "high",
  "expires_at": "2026-06-24T09:10:46+00:00"
}
```

### `do_task`

Run a web/browser task for the signed-in user — open sites, fill forms, complete actions. The
browser profile **persists per user across calls**. May take several minutes.

| Name | Type | Required | Description |
| --- | --- | --- | --- |
| `task` | string | yes | What the agent should do, in plain language |
| `start_url` | string | no | Page to open before starting |
| `user_uuid` | string | no | Profile override; defaults to the signed-in user |

```json
{ "uuid": "user-or-profile-uuid", "response": "what the agent did / found", "success": true }
```

If the agent needs a login, card, or personal detail, it fetches it securely from Valyd (the user
approves on their phone) — **secrets are never returned to the calling agent in plain text**.

## Status reference

| Status | Meaning |
| --- | --- |
| **PENDING** | Waiting for the user on their Valyd app. Keep polling. |
| **APPROVED** | The user approved (face-verified). Proceed. |
| **DENIED** | The system/policy denied the request. |
| **DECLINED** | The user explicitly declined. **Do not proceed.** |
| **EXPIRED** | `expires_at` passed without a decision. |

`assurance_level` (e.g. `high`), when present, describes how strongly the user was verified.

## Agent flow

**Verification (human-in-the-loop)**

1. `verification_request` with `action_type`, `title`, `description`.
2. Poll `verification_status(valyd_session_id)` until not `PENDING`.
3. **APPROVED** → perform the action. **DENIED / DECLINED / EXPIRED** → abort and tell the user.

**Web task**

1. `do_task` with `task` (and optional `start_url`).
2. Read `response` and `success`.

Combine them: gate a `do_task` behind a `verification_request` approval for sensitive actions.

## Client setup

### Claude Code

```bash
claude mcp add --transport http valyd https://mcp.valyd.id/verification/mcp
```

Then run `/mcp` and choose **Authenticate** for `valyd` — a browser opens for the Valyd login.

```json
// .mcp.json (project-scoped)
{
  "mcpServers": {
    "valyd": { "type": "http", "url": "https://mcp.valyd.id/verification/mcp" }
  }
}
```

**No secrets go in this file.** Claude Code performs the OAuth flow and stores the token. Re-run
`/mcp` → Authenticate if it expires.

### Codex (OpenAI Codex CLI)

```toml
# ~/.codex/config.toml
[mcp_servers.valyd]
url = "https://mcp.valyd.id/verification/mcp"
```

```bash
codex mcp login valyd
```

If your build lacks interactive OAuth login:

```toml
[mcp_servers.valyd]
url = "https://mcp.valyd.id/verification/mcp"
bearer_token_env_var = "VALYD_MCP_TOKEN"
```

```bash
export VALYD_MCP_TOKEN="<access_token from https://idp.valyd.id>"
```

### Cursor / Claude Desktop

```json
{ "mcpServers": { "valyd": { "url": "https://mcp.valyd.id/verification/mcp" } } }
```

Cursor: `.cursor/mcp.json` or Settings → Tools & MCP. Restart the client; it prompts to authorize on
first use. **No trailing slash.**

### LangChain

```bash
pip install langchain-mcp-adapters langgraph langchain-openai
```

```python
import asyncio
from langchain_mcp_adapters.client import MultiServerMCPClient
from langgraph.prebuilt import create_react_agent

TOKEN = "<access_token from https://idp.valyd.id, scope: openid mcp>"

async def main():
    client = MultiServerMCPClient({
        "valyd": {
            "transport": "streamable_http",
            "url": "https://mcp.valyd.id/verification/mcp",
            "headers": {"Authorization": f"Bearer {TOKEN}"},
        }
    })
    tools = await client.get_tools()   # verification_request, verification_status, do_task
    agent = create_react_agent("openai:gpt-4o", tools)
    result = await agent.ainvoke(
        {"messages": "Ask the user to approve deleting the prod database, then tell me the outcome."}
    )
    print(result["messages"][-1].content)

asyncio.run(main())
```

### OpenAI Agents SDK

```bash
pip install openai-agents
```

```python
import asyncio
from agents import Agent, Runner
from agents.mcp import MCPServerStreamableHttp

TOKEN = "<access_token from https://idp.valyd.id, scope: openid mcp>"

async def main():
    async with MCPServerStreamableHttp(
        name="valyd",
        params={
            "url": "https://mcp.valyd.id/verification/mcp",
            "headers": {"Authorization": f"Bearer {TOKEN}"},
        },
    ) as valyd:
        agent = Agent(
            name="Assistant",
            instructions="Use Valyd tools for approvals and web tasks.",
            mcp_servers=[valyd],
        )
        result = await Runner.run(agent, "Book a table for two on the user's behalf at example.com.")
        print(result.final_output)

asyncio.run(main())
```

Alternative — OpenAI Responses API hosted MCP tool:

```python
resp = client.responses.create(
    model="gpt-4o",
    tools=[{
        "type": "mcp",
        "server_label": "valyd",
        "server_url": "https://mcp.valyd.id/verification/mcp",
        "headers": {"Authorization": "Bearer <access_token>"},
        "require_approval": "never",
    }],
    input="Ask the user to approve the payment, then proceed if approved.",
)
```

## Common errors

| Symptom | Cause | Fix |
| --- | --- | --- |
| `401` with `WWW-Authenticate` on connect | No / expired token | Let the client run the OAuth login (`/mcp` → Authenticate). Code clients: fetch a fresh token. |
| `invalid_token` | Wrong `aud`, `iss`, `scope`, or signature | Token needs `aud=https://mcp.valyd.id`, `iss=https://idp.valyd.id`, scope including `mcp` |
| `invalid_scope` | Requested a scope the IdP won't grant | Request only `openid mcp` |
| `missing_token` | No `Authorization: Bearer` header | Send it on every request |
| `do_task` returns `success: false` | The web agent couldn't complete the task | Read `response`; refine `task` / `start_url` and retry |

## Machine-readable resources

For an agent that wants the docs rather than the tools:

```bash
curl -sL https://docs.valyd.id/llms.txt              # index: hierarchy, base URLs, credential rules
curl -sL https://docs.valyd.id/llms-full.txt         # full corpus, one file
curl -sL https://docs.valyd.id/openapi/valyd-id.json      # OpenAPI 3.1 — TPSSO + OIDC
curl -sL https://docs.valyd.id/openapi/valyd-verify.json  # OpenAPI 3.1 — sessions, core, webhooks
curl -sL https://docs.valyd.id/valyd-postman-collection.json
```

Fetch `llms.txt` first — it lists every page as a clean Markdown URL. Fetch pages on demand rather
than loading the full corpus unless you need everything, and prefer those `.md` URLs over scraping
the HTML.
