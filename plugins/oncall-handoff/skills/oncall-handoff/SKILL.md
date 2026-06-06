---
name: oncall-handoff
description: Synthesize PagerDuty incidents, Slack #oncall threads, and recent commits into the handoff brief the next on-call needs. Use when the user is wrapping up a rotation, says "doing my handoff," "writing up oncall," "passing the pager," or runs /oncall-handoff.
user-invocable: true
argument-hint: [hours-back | "rotation"]
---

# /oncall-handoff — On-call handoff brief

The brief the next on-call needs to walk in already in context: what fired, what's still smoldering, what was changed in production this week, and what the incoming person should keep an eye on for the next 24 hours.

Triggered at the end of a shift. Composes three internal-tools MCPs the platform team would own.

---

## Composes

- **PagerDuty** — `list_incidents` (resolved + unresolved during window), `get_incident` (for the still-smoldering ones)
- **Slack** — `search_messages` against `#oncall` and any incident-channel that was spawned
- **GitHub** — `list_merged_prs` and `list_open_prs` for repos touched during the window
- **Optional: Sentry / Datadog** — for noisy error groups or saturation alarms that didn't page

None of those MCPs ship in this repo — they're sketches of the data primitives an internal AI platform would wire. The skill describes the contract; the platform team owns the wiring.

---

## When to trigger

User typed `/oncall-handoff`, or said:
- "writing up oncall," "doing my handoff," "passing the pager"
- "what happened on my shift"
- "summarize this week for the next on-call"

Don't trigger on general PagerDuty mentions — only when the user is explicitly closing out a rotation.

## Argument handling

`$ARGUMENTS` is:
- empty → default to last 7 days (a standard week-long rotation)
- a number (e.g. `48`) → that many hours back
- `"rotation"` → from the user's last-known rotation start (derived from PagerDuty schedule)

## Workflow

### 1. Define the window

Compute `since`. Default to 7 days ago at the rotation handoff time.

### 2. Pull incidents

- `pagerduty.list_incidents({ since, status: ["resolved","acknowledged","triggered"] })`
- Split into three buckets:
  - **resolved** — fired and closed during the window
  - **acknowledged or triggered** — still in flight; these get top billing in the brief
  - **auto-resolved noise** — fired and closed within < 5 minutes; bucket as "noise" and just count

For each still-in-flight incident, `pagerduty.get_incident(id)` for the full timeline.

### 3. Pull Slack context

For each still-in-flight incident, search `#oncall` and any auto-created `inc-*` channel for the most recent 2–3 substantive messages. Surface decisions, owners, current theory of cause.

### 4. Pull production changes

`github.list_merged_prs({ since, repos: <repos touched by incidents OR the team's primary repos> })`.
`github.list_open_prs({ awaiting_review: true })` — call out any PR sitting open that the incoming on-call should be aware of (likely landing during their shift).

### 5. Output the brief

```
# On-Call Handoff — <outgoing name> → <incoming name>
Rotation: <start> → <end>

## Still in flight (top of pager)
- INC-<id> · <title> · <severity>
  Last update: <time, short summary>
  Owner / next step: <owner — what to do next>
- ...

## Resolved this rotation
- <count> incidents · <noise-count> noise · top-3 by severity:
  - INC-<id> · <title> · <one-line cause + fix>
- ...

## Shipped to production
- <PR title> · <author> · <one-line impact>
- ...

## Open PRs that may land during your shift
- <PR title> · <author> · <state, e.g. "approved, blocked on CI">

## Watch list (next 24h)
- <thing>: <why to watch it>

---
Sources: pagerduty (<N> calls), slack (<M> threads), github (<K> PRs).
Stubs in this MVP: Sentry (error-group volume), Datadog (saturation alarms).
```

Like the other skills in this marketplace, the "Sources" footer is honest about what was queried vs stubbed. That's the platform-team feedback loop: the brief itself tells you which MCPs to wire next.

## Edge cases

- **No incidents in window** — still produce the "Shipped to production" + "Open PRs" + "Watch list" sections. A quiet week is information.
- **Many incidents (>15)** — group resolved incidents by service or root-cause cluster rather than listing each.
- **Cross-team incidents** — if multiple services / teams were involved, name the partner team contact under "Owner / next step" so the incoming on-call knows who to ping.
- **Sensitive content** (security, customer data) — summarize at higher level, don't quote message bodies.

## Tone

Compact. Operational. The reader is reading this 5 minutes before going on-call, possibly at 7am. Concrete > comprehensive. Use the imperative for "next step" lines ("Page Sarah if X recurs" beats "Sarah might be a good person to contact if...").
