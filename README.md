# Valyd Skills

Agent Skills that teach your coding agent to build on [Valyd](https://docs.valyd.id) — sign-in,
KYC, and verification APIs.

## Install

```bash
npx skills add valyd-id/skills
```

Add `-g` to install globally instead of into the current project. Then restart your agent session.

Works with Claude Code and any other agent that reads `SKILL.md`.

## Use it

Nothing to invoke — it activates on its own when you mention Valyd. Just ask:

> Add Login with Valyd to this Express app.
>
> Verify a doctor's license before letting them book a shift.
>
> Wire up hosted KYC with a signed webhook.

## What it covers

- **Connect with Valyd** — OpenID Connect sign-in, tokens, sessions
- **Reusable Verification** — workflow sessions for KYC, liveness, face match, age, licenses, location
- **Unique Human API** — liveness and face uniqueness, with no user login
- **`@valyd/sdk`** — every class, method and option
- **Webhooks** — signatures, retries, deduplication
- **Organizations** — roles, workforce, the Members API
- **MCP** — connecting an agent to Valyd's hosted MCP server

Plus the traps: the CSRF check that silently does nothing, the endpoints that now return `410`, the
methods removed from the SDK, raw-body signature verification, and a dozen more that break
integrations on the first try.

## Good to know

The references target Valyd's **development** environment (`idp.valyd.id`). Production and
testing mirror it on their own domains with their own credentials — set the host per environment
via `VALYD_IDP_URL`, and never mix a key from one environment with the host of another.

Credentials always come from a human at [dev.valyd.id](https://dev.valyd.id). No API mints
them, and the skill tells your agent to ask rather than invent them.

## License

[MIT](LICENSE)
