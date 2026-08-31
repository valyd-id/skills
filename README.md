# Valyd Skills

Agent Skills for building on [Valyd](https://docs.valyd.work) — verified identity, sign-in, and
verification APIs.

A skill teaches a coding agent (Claude Code, and anything else that reads `SKILL.md`) how to
integrate a product correctly on the first attempt, instead of guessing from a docs site.

## Skills

### [`valyd-integration`](skills/valyd-integration/)

Login with Valyd (OAuth2/TPSSO + OIDC), the Verification APIs (KYC, liveness, anti-spoof, face
match, age, professional licenses, geolocation), `@valyd/sdk`, signed webhooks, the workforce
Members API, consent-based attribute release, and the Valyd MCP server.

## Install

Using the [skills CLI](https://www.skills.sh) — run from your project:

```bash
npx skills add valyd/valyd-skills --skill valyd-integration
```

Add `-g` to install globally (`~/.claude/skills/`) instead of into the project
(`.claude/skills/`). Use `--skill '*'` to install everything in this repo.

Or install by hand:

```bash
git clone https://github.com/valyd/valyd-skills /tmp/valyd-skills
cp -r /tmp/valyd-skills/skills/valyd-integration ~/.claude/skills/
```

Either way, restart your agent session — `SKILL.md` is picked up on start.

## Using it

Nothing to invoke. The skill activates on its own when a task mentions Valyd, `valyd_id`, TPSSO,
"Login with Valyd", `@valyd/sdk`, `idp.valyd.work`, `workflow_id`, verification sessions, or
verifying someone's identity or license through Valyd.

Ask for what you want and the agent handles the rest:

> Add Login with Valyd to this Express app.
>
> Verify a doctor's license before letting them book a shift.
>
> Wire up hosted KYC with a signed webhook.

## What's in it

`SKILL.md` is a router, not the documentation. It carries the eleven rules that cause almost every
Valyd integration bug, the decision tree for picking an integration shape, the hosts and response
envelope, and three snippets that are correct as written. Everything else loads only when a task
needs it:

```text
skills/valyd-integration/
├── SKILL.md
└── references/
    ├── portal-and-accounts.md          Getting credentials; passwordless dev sign-in
    ├── login-oauth.md                  The TPSSO flow end to end
    ├── login-sessions-csrf.md          The CSRF mechanism (read this one)
    ├── scopes.md                       Which scope returns what
    ├── tpsso-endpoints.md              The five endpoints, request/response shapes
    ├── oidc.md                         Enterprise SSO, Mendix, discovery, JWKS
    ├── consent-attributes.md           Getting raw legal name / DOB with consent
    ├── verify-modes-and-account.md     Hosted vs Core, Account vs Non-account
    ├── verify-hosted.md                Session → redirect → webhook → decision
    ├── verify-core-apis.md             Every /api/v2/* check and its fields
    ├── webhooks.md                     Signatures, retries, dedupe, delivery log
    ├── statuses.md                     Status values and how to act on each
    ├── sdk.md                          @valyd/sdk — classes, methods, options, types
    ├── errors.md                       Every error code, both products, with fixes
    ├── organizations-members.md        Workforce, roles, billing, private apps
    ├── mcp.md                          Connecting an agent over MCP
    ├── recipes.md                      Four worked end-to-end builds
    └── gotchas-and-doc-conflicts.md    Where the published docs disagree
```

## Two things to know

**These references target the development environment.** Every host in them —
`idp.valyd.work`, `dev.valyd.work` — is development. `new Valyd({ ... })` defaults to
**production** (`valyd.id`), so dev credentials need `env: "development"` or OAuth fails with
`client_id/redirect_uri not allowed`. This is rule #3 in `SKILL.md` for a reason.

**Credentials are always a human step.** Nothing in Valyd mints a `client_id`, `client_secret`,
API key, or `workflow_id` over an API — they come from a person visiting
[dev.valyd.work](https://dev.valyd.work). The skill tells the agent to stop and ask rather than
invent them.

## Contributing

The reference files are derived from the published Valyd docs and will drift as those change. If
you hit something the skill gets wrong, or a place where the docs have since been corrected, open
an issue or a PR against the relevant file in `references/`.

`gotchas-and-doc-conflicts.md` tracks places the published docs contradict themselves. When one is
resolved upstream, delete the entry.

## License

[MIT](LICENSE)
