# cowork-mvp — Bloomerang internal skills marketplace

A small Claude Code plugin marketplace built around a single thesis: the unit of capability in an internal AI platform is an **installable skill that composes existing MCPs**.

Two skills today:

- **`/meeting-prep`** — Pre-meeting brief from Calendar + Gmail + Drive.
- **`/customer-snapshot`** — One-screen view of a Bloomerang customer before any CS/Support/AE conversation: account record, usage signals, recent touches, suggested next action. Requires [bloomerang-mcp](https://github.com/callanable/bloomerang-mcp).

Both are small on purpose. The point is the *pattern*: installable, governable, shareable capabilities that compose primitives the platform team already owns.

See [`DESIGN.md`](DESIGN.md) for how this generalizes — memory layer, RBAC via MCP scoping, PR-based shipping. See the [marketplace site](https://callanable.github.io/cowork-mvp/) for a browseable view including the OIDC sign-in + provisioning demo.

### Companion repo

- [**bloomerang-mcp**](https://github.com/callanable/bloomerang-mcp) — MCP server that wraps Bloomerang's donor CRM API. CS/Support use it to look into a customer's account at how they're using the product. One of several MCPs an internal platform would wire up.

---

## meeting-prep

A Claude Code plugin that turns three Google Workspace MCPs into a single command: `/meeting-prep`.

Run it before any meeting and it produces a one-page brief — attendees, recent email context per person, related Drive docs, open threads, suggested talking points. Zero configuration once the plugin is installed.

---

## What you get

A single skill: **`/meeting-prep [event-title-or-keyword]`**

- No args → briefs your next meeting.
- With args → briefs the next upcoming meeting matching that title or attendee.

Example output:

```
# Meeting Prep — Quarterly review with Acme Foundation
Tuesday 2:00 PM · 30 min · meet.google.com/abc-defg-hij

## Attendees
- Jane Smith <jane@acmefoundation.org> — Director of Development, Acme Foundation
- Mark Lee <mark@acmefoundation.org> — Program Manager

## Context
Acme adopted Bloomerang in Q1 and ran their spring appeal through the
platform — Jane reported a 22% YoY lift two weeks ago. Last touch was Jane
asking about the new recurring-giving widget on May 12; no reply yet.

## Open threads
- Jane: asked about Q3 roadmap for the recurring-giving widget (May 12)
- Mark: requested CSV export format docs (May 8) — replied with link May 9

## Relevant docs
- "Acme Foundation — Onboarding Notes" — original implementation plan, has
  their custom field mappings
- "Q1 Customer Health — Top 50" — Acme is in green tier, NPS 9

## Suggested talking points
- Walk back to Jane's May 12 question; the recurring-giving roadmap update
  shipped last week
- Confirm Q3 expansion conversation is still on for August
- Ask about staffing changes — Mark's title in his sig changed since onboarding
```

---

## Install

> **Note for end users**: you should never have to do any of this. In a real deployment, IT registers the org marketplace once in Cowork settings, and `/meeting-prep` just appears in every employee's session. The steps below are for testing the plugin on your own machine, or for IT adding it to the org marketplace the first time.

### How an employee actually gets it (production model)

Zero steps. IT adds the marketplace to Cowork's org-wide config once; employees see `/meeting-prep` on next session start. No clones, no dialogs, no paths.

### How IT adds it to an org marketplace

Copy `plugins/meeting-prep/` into your org's private marketplace repo and add an entry to its `marketplace.json`. Push. Done — every Cowork user in the org gets it on next sync.

### How to test it locally (for evaluators)

```bash
git clone <this repo> ~/cowork-mvp
```

Then in Claude Code, run these as **two separate commands**:

1. Type or paste this and press Enter:
   ```
   /plugin marketplace add ~/cowork-mvp
   ```
   A dialog appears with the path pre-filled — press Enter again to confirm. (If you type a path manually in the dialog, don't escape spaces — it's a text field, not a shell.)

2. Then:
   ```
   /plugin install meeting-prep@cowork-mvp
   ```

Restart Claude Code, then `/meeting-prep` is available in any session.

---

## Prerequisites

This plugin assumes the following MCPs are available in the user's Cowork environment — most teams already have them via the Anthropic Connectors panel (Settings → Connectors):

- **Google Calendar** connector — enabled
- **Gmail** connector — enabled
- **Google Drive** connector — enabled

The plugin doesn't ship its own MCP servers because Cowork's first-party Google connectors already handle OAuth, refresh, and scope management correctly. If your org has Google Workspace + Cowork enabled, you're already done.

If a connector is missing when `/meeting-prep` runs, the skill will tell the user which tool it couldn't find rather than failing silently.

---

## Usage

In any Claude Code session:

```
/meeting-prep
```

or

```
/meeting-prep acme
/meeting-prep "quarterly review"
/meeting-prep jane@acmefoundation.org
```

The brief is printed inline. Copy to a doc, paste into a Slack thread, or just read it. The skill doesn't persist anything — it's stateless by design (see `DESIGN.md` for the memory layer plan).

---

## What's next

See `DESIGN.md` for the full thinking on how this plugin pattern scales into a real internal AI platform.

Near-term ideas already half-baked:

- **Bring-your-own context** — drop a PDF/markdown file in `context/` and have the skill include it when relevant (e.g. a customer's contract, a job description, a deal brief).
- **Memory layer** — write attendee summaries to a per-person memory file so the second prep for the same customer is faster, richer, and shows what's changed since last time.
- **Scheduled briefs** — cron 30 minutes before each meeting, drop the brief into a Drive doc the user already has open.
