# Valyd Skills

Agent Skills that teach your coding agent to build on [Valyd](https://docs.valyd.work) — sign-in,
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

- **Login with Valyd** — OAuth2 / TPSSO and OIDC
- **Verification APIs** — KYC, liveness, anti-spoof, face match, age, professional licenses, location
- **`@valyd/sdk`** — every class, method and option
- **Webhooks** — signatures, retries, deduplication
- **Members API** — organizations, roles, workforce
- **MCP** — connecting an agent to Valyd's hosted MCP server

Plus the traps: the `state` parameter that isn't echoed back, the SDK environment default, raw-body
signature verification, and a dozen more that break integrations on the first try.

## Good to know

The skill targets Valyd's **development** environment (`idp.valyd.work`). With dev credentials you
need `env: "development"` — the SDK defaults to production.

Credentials always come from a human at [dev.valyd.work](https://dev.valyd.work). No API mints
them, and the skill tells your agent to ask rather than invent them.

## License

[MIT](LICENSE)
