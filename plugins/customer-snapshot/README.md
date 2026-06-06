# customer-snapshot

A skill for Bloomerang CS, Support, and AE folks: run it before any customer conversation and get a one-screen brief — account record, usage signals, recent touches, open items, suggested next action.

It's a **sketch**, not a finished workflow. The interesting part is what it demonstrates: how a Bloomerang-specific MCP composes with first-party connectors, and how an honest "Sources" footer in every brief tells the platform team which MCPs to wire next.

---

## Composes

- **[bloomerang-mcp](https://github.com/callanable/bloomerang-mcp)** — `find_constituent`, `get_constituent`, `list_giving`, `list_interactions`. Looks into the customer's Bloomerang account to surface what they're processing, what funds they run, and who they're interacting with.
- **Gmail** (optional) — recent threads with the customer's primary contacts.

What would be wired in production but isn't today (and the brief says so):

- **Zendesk** — open support tickets, severity, age
- **Salesforce** — renewal date, ARR, CSM owner, health score
- **Looker** — login frequency, feature adoption, processing volume

The "Sources" footer on every brief lists what was queried and what's stubbed. That's the platform-feedback-loop in miniature: every skill run tells the platform team what's missing.

---

## Example output

```
# Customer Snapshot — Acme Foundation
active · last touch 2026-05-12 · primary contact Jane Smith

## Account
- Lifetime CRM activity: $175,000 across 4 transactions
- Last activity: 2026-04-10 · $25,000 · Spring Appeal (Recurring Giving)
- Primary contact: Jane Smith, Director of Development

## Product signals
- Heavy use of recurring-giving program (restricted $25K to expansion)
- Trajectory growing — 22% YoY lift on spring appeal reported in April
- Engaged technical contact (Mark) actively integrating CSV exports

## Open items
- 2026-05-12: Jane asked about Q3 roadmap for recurring-giving widget — no reply
- 2026-04-12 QBR: confirmed Q3 expansion conversation for August

## Suggested next action
- Reply to Jane's May 12 thread — the recurring-giving roadmap update
  shipped last week. This is the highest-leverage touch.
- Confirm August Q3 expansion meeting is on her calendar.

---
Sources: bloomerang-mcp (4 calls).
Not yet wired in this demo: Zendesk (open tickets, severity, age),
  Salesforce (renewal date, ARR, CSM owner, health score),
  Looker (login frequency, feature adoption, processing volume).
```

---

## Install

Same as any plugin in this marketplace. See the root [README](../../README.md#install) for the install commands, or the [meeting-prep README](../meeting-prep/README.md#install) for the longer "production model vs local testing" framing.

This plugin requires the **bloomerang-mcp** server to be wired in your MCP config. See [bloomerang-mcp](https://github.com/callanable/bloomerang-mcp) for setup.

---

## Honesty about the demo data

The mock fixtures in `bloomerang-mcp` dress donor-CRM data up as CSM signals (treating "lifetime giving" as "lifetime CRM activity," for instance). That's metaphorical — the real internal version of this skill would compose against an internal customer DB, not the public donor CRM API. The point of the artifact is to show the *shape* of a customer-facing skill, not to be a production-grade CS tool.

The skill itself ([SKILL.md](skills/customer-snapshot/SKILL.md)) is the contract; the data layer is the bit a platform team would replace.
