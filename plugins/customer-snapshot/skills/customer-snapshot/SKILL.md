---
name: customer-snapshot
description: One-screen snapshot of a Bloomerang nonprofit customer — account record, usage signals, recent touches, open items, and a suggested next action. Use when a CSM, AE, or Support engineer asks to "pull up <customer>", "what's the latest on <customer>", "snapshot of <customer>", or runs /customer-snapshot. For internal use before any customer-facing conversation.
user-invocable: true
argument-hint: <customer-name-or-domain>
---

# /customer-snapshot — Pre-conversation customer brief

The single most-used skill on a customer success team. Run it before any call, email, or QBR with a Bloomerang customer (a nonprofit on the platform) and get a one-screen view of:

- Who they are and how big
- How they're using Bloomerang
- What's happened recently — touches, tickets, signals
- What's open and unresolved
- What the next action should be

The skill is for **Bloomerang employees** (CSM, AE, Support, Onboarding). The "customer" is a nonprofit using Bloomerang. This is the employee-facing read of the [bloomerang-mcp](https://github.com/callanable/bloomerang-mcp) data primitive — see that repo's README for the framing.

---

## Composes

- **bloomerang** ([bloomerang-mcp](https://github.com/callanable/bloomerang-mcp)) — `find_constituent`, `get_constituent`, `list_giving`, `list_interactions`
- **Gmail** (optional) — recent email threads with the customer's primary contacts
- **Hypothetical, not yet wired in this MVP** — see footer of every brief:
  - Zendesk MCP — open support tickets, severity, age
  - Salesforce MCP — renewal date, ARR tier, CSM owner, health score
  - Looker / Sigma MCP — feature-adoption metrics, login frequency, processing volume
  - Slack MCP — recent mentions in customer-channels

The skill works against just `bloomerang` + Gmail today; the brief footer flags which sections would be richer once the other MCPs are wired by the platform team. Honest about what's stubbed.

---

## When to trigger

- User typed `/customer-snapshot <name>` (with or without an argument).
- User says "pull up Acme Foundation", "what's the latest on Acme", "snapshot of Acme", "give me the Acme account".
- User is about to send an email or join a call where the recipient is a known customer — Cowork may proactively suggest this, but only with explicit opt-in.

Don't trigger from automation, channel messages from non-employees, or context that's clearly about an external (non-customer) entity.

## Argument handling

`$ARGUMENTS` is the customer name (e.g. "Acme Foundation"), an email domain ("acmefoundation.org"), or an email of a primary contact ("jane@acmefoundation.org"). If empty, ask which customer in one line and stop.

## Workflow

### 1. Resolve to a customer record

- Call `bloomerang.find_constituent` with the argument. Filter results to organization-type records first; if none, fall back to the highest-lifetime-giving individual whose organization name or email domain matches.
- If 2+ ambiguous matches, list them by name + email and ask the user to disambiguate. Don't guess.
- If zero matches, say so in one line ("No customer found matching <arg>") and exit.

### 2. Pull the account picture

Per resolved customer, in parallel:

- `bloomerang.get_constituent(id)` — full record, lifetime giving, last activity, primary contact
- `bloomerang.list_giving(id)` — full transaction history (stands in for "usage signals" — recurring fund tells you they're using the recurring-giving features, large gifts to restricted funds tell you what they care about)
- `bloomerang.list_interactions(id)` — every logged touch (emails, calls, meetings), most recent first

If the customer has known contacts with email addresses, optionally:

- Gmail `search_threads` for `from:<contact-email> OR to:<contact-email>`, last 30 days, max 3 threads per contact
- Pull one-line summaries — surface anything unanswered

### 3. Synthesize signals

From the data above, infer:

- **Engagement state** — active (touched within 30 days) / quiet (30–90 days) / lapsed (90+ days)
- **Product signals** — which Bloomerang capabilities they're using, inferred from fund names, gift methods, campaign structure
- **Relationship trajectory** — growing (recent gifts/touches increasing) / steady / declining
- **Open threads** — questions the customer asked that no one replied to, commitments we made that haven't been delivered

This synthesis is the *point* of the skill — raw data is one click away in the admin panel; the brief earns its keep by saying what the data *means*.

### 4. Output the brief

Use this exact format. Keep total length under one screen. Omit any section that has no content (don't print "None").

```
# Customer Snapshot — <customer name>
<engagement state> · last touch <date> · primary contact <name>

## Account
- <one line on size: lifetime CRM activity, # transactions>
- Last activity: <date> · <amount> · <fund/campaign>
- Primary contact: <name>, <title>

## Product signals
- <inferred from fund names + gift patterns: which capabilities they use>
- <relationship trajectory in one line>

## Open items
- <date>: <one-line summary of unresolved thread or commitment>
- <date>: <one-line summary>

## Suggested next action
- <one or two specific actions, derived from open items or the data>

---
Sources: bloomerang-mcp (<N> calls)<, Gmail if used>.
Not yet wired in this demo: Zendesk (open tickets, severity, age),
  Salesforce (renewal date, ARR, CSM owner, health score),
  Looker (login frequency, feature adoption, processing volume).
```

The "Sources" footer is non-negotiable. It tells the user (a) what the brief is grounded in, (b) what's missing because the platform hasn't wired it yet, and (c) what to push the platform team to add next. That feedback loop is the whole game.

## Edge cases

- **MCP not wired**: if `bloomerang.*` tools aren't available, say "The Bloomerang MCP isn't wired in this Cowork environment. Add via [bloomerang-mcp](https://github.com/callanable/bloomerang-mcp) or ask the platform team." Stop.
- **Ambiguous customer**: list candidates, ask. Don't guess.
- **Zero data**: if find_constituent returns a record but get_constituent / list_giving / list_interactions are all empty, output a minimal brief with just the account row and a note that there's no activity to summarize.
- **Internal "customer" mismatch**: if the argument resolves to a known internal employee or a non-customer entity, push back rather than producing a confusingly-framed brief.
- **Sensitive context** (collections, churn, legal): if recent interactions contain obvious sensitive material, summarize at a higher level rather than quoting, and flag for human review in the suggested-next-action line.

## Tone

Neutral, specific, factual. The reader is 90 seconds away from a real conversation with a real customer. Brevity beats completeness; concrete signals beat hand-wavy synthesis; "I don't know" beats a guess.

## RBAC note

In a production deployment, what this skill *sees* depends entirely on which MCPs are scoped to the calling user. A Tier-1 Support engineer might see the bloomerang record but not the Salesforce health score; a CSM sees both; an exec sees everything plus revenue. Same skill, different data view, governed at the MCP layer — not duplicated in the skill code. See [cowork-mvp's DESIGN.md](https://github.com/callanable/cowork-mvp/blob/main/DESIGN.md#3-governed-access-via-mcp-scoping) for the full framing.
