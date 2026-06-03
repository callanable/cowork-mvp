# Design — why this plugin, and how it generalizes

## Thesis

The interesting question isn't "what AI feature should we build?" It's "what does the *unit of capability* look like in an internal AI platform?"

This plugin is one answer: a **skill, packaged as a plugin, composing MCPs that the org already has**. Install once. Every employee gets it. Zero per-user configuration. The skill is the marketplace unit; the plugin is the install primitive; the MCPs are the governed-data primitive.

`meeting-prep` is deliberately small. It's a demo of the *shape*, not the destination.

## Why a skill, not a chatbot

A chatbot is a UI. Anyone can build one. The platform problem isn't the UI — it's making capability *installable, discoverable, and governable*. Skills are all three:

- **Installable**: shipped as plugins, dropped into a marketplace, surfaced in every employee's Claude Code session without them touching settings.
- **Discoverable**: trigger conditions live in the skill description; Claude routes to them based on what the user is doing, not what they remember to invoke.
- **Governable**: the MCP layer underneath is where RBAC actually lives. The skill is portable across permission scopes.

A chatbot doesn't get installed. A skill does.

## How this pattern generalizes to the full platform

### 1. The skills marketplace

The plugin layout in this repo (`.claude-plugin/plugin.json` + `skills/<name>/SKILL.md`) is the install primitive Cowork already supports. A Bloomerang skills marketplace is, mechanically, a private git repo of these plugin directories that every employee's Cowork is configured to sync from.

Day-1 work: stand up `bloomerang-skills-marketplace` as a private repo, configure Cowork's `/plugin marketplace add` to point at it for the org, set up CI that lints new plugins on PR. From that day forward, *shipping a skill = merging a PR*.

### 2. Memory that works

Today this skill rebuilds context per invocation. Every time you `/meeting-prep` the same customer, it re-reads the same email history.

Next iteration: after each run, the skill writes a small per-person summary file to a memory store (`memory/contacts/jane@acmefoundation.org.md`) — relationship state, last touch, open commitments. Next invocation reads that file first and only fetches what's *changed* since. The brief now shows "what's new since last time you talked to Jane."

That memory layer is the same primitive other skills will need. It belongs in the platform, not the skill. The platform owns the memory store; skills declare what they want to write to it and read from it via a memory MCP.

### 3. Governed access via MCP scoping

The skill doesn't know or care which user is running it. It asks for the Google Calendar / Gmail / Drive MCPs and uses whatever's wired.

That's where RBAC lives. The platform decides — per user, per Okta group, per role — which MCPs are wired into that user's Cowork environment:

- Sales reps get Gmail + Calendar + Salesforce.
- CSMs get Gmail + Calendar + Bloomerang's own product DB (read-only, tier-1 customer data only).
- Engineers get Drive + GitHub + Sentry.
- Finance gets Drive + Calendar + the GL system (just-in-time, approval-gated).

The same `meeting-prep` skill works for all of them; what it can *see* depends entirely on what MCPs are scoped to that user. The skill is portable; the data governance is centralized.

This is the "tiered data model with role-based, just-in-time access" from the JD — implemented at the MCP layer, not duplicated in each skill.

## What it took to ship this

One evening. Four files: a plugin manifest, a skill, a README, this doc. No new infrastructure, no custom server, no model fine-tuning. The platform is mostly *composition*, not invention — most of the leverage is in deciding what to wire to what and shipping the resulting capability as something employees can actually find and run.

The bias I'd bring to the role: ship small, ship often, let usage data tell you what to invest in next.

## What I'd build week 1 at Bloomerang

1. **Audit the wired MCPs** — what's already available in Cowork org-wide? What's available per-team? Which connectors are paid, which need IT setup? Document the current surface.
2. **Sit with three departments** — sales, CS, support. Two hours each. Watch them work. Find the 3–4 friction points where a skill would shave 15+ minutes off a recurring task.
3. **Ship one skill per friction point** — even if rough. Push to the marketplace. Tell people.
4. **Stand up `bloomerang-skills-marketplace`** as a private repo with PR-based contribution flow, so by end of month builders elsewhere in the org can contribute.

The platform isn't a thing you build before users see anything. It's the residue of shipping skills people use.
