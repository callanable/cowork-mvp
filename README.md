# cowork-mvp — Bloomerang internal skills marketplace

A small Claude Code plugin marketplace built around a single thesis: the unit of capability in an internal AI platform is an **installable skill that composes existing MCPs**. The platform owns the identity layer, the MCP-wiring layer, and the marketplace; skill authors own the workflows.

## Live demo

**https://callanable.github.io/cowork-mvp/**

Sign in with Google → the page reads your identity, picks a role, and shows which MCPs are provisioned and which skills are available to you. The OIDC flow is real. The role-to-MCP mapping is mocked for the demo; in production it's driven by Workspace groups, SCIM, or HRIS, owned by the platform's identity broker.

What the demo shows end to end:

- **Identity** — real Google OIDC sign-in
- **Role** — derived from claims (mocked here as a picker)
- **MCPs** — which data sources are wired for that role
- **Skills** — which marketplace plugins that role can run

That's the JD's "Frictionless Access" and "Marketplace of Skills" bullets, working.

## Skills

Three plugins ship today. Each has its own README under `plugins/<name>/`.

- **[`/meeting-prep`](plugins/meeting-prep/README.md)** — Pre-meeting brief from Calendar + Gmail + Drive. The polished worked example: useful to anyone with meetings, composes three first-party Anthropic Connectors, no custom MCPs required.
- **[`/oncall-handoff`](plugins/oncall-handoff/README.md)** — Synthesize PagerDuty + Slack + recent commits into the handoff brief the next on-call needs. Internal-platform-engineering shape.
- **[`/customer-snapshot`](plugins/customer-snapshot/README.md)** — Pre-conversation brief for CS/Support/AE looking at a specific customer's Bloomerang account. Composes [bloomerang-mcp](https://github.com/callanable/bloomerang-mcp) with Gmail. Sketch — the brief is honest about which MCPs would be wired in production.

## Companion repo

- **[bloomerang-mcp](https://github.com/callanable/bloomerang-mcp)** — Example MCP server. Wraps Bloomerang's donor CRM API as an illustration of the data-primitive pattern. In production an internal AI platform wires up several such MCPs (customer DB, Salesforce, Zendesk, Looker) — each a small, governed tool surface that skills compose against.

## Install

In a real Bloomerang deployment, IT registers this marketplace once in Cowork settings and every employee gets the skills on next session start — no per-employee setup. The two commands below are for local testing or first-time IT setup.

```
/plugin marketplace add ~/cowork-mvp
/plugin install meeting-prep@cowork-mvp
```

(Substitute any plugin name. Restart Claude Code after installing.)

## Design

See [`DESIGN.md`](DESIGN.md) for the platform thinking — memory layer, RBAC via MCP scoping, PR-based shipping for skill contributions.
