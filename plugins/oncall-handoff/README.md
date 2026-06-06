# oncall-handoff

A Claude Code plugin for engineering teams: `/oncall-handoff` synthesizes the brief the next on-call needs from PagerDuty + Slack + GitHub.

Run it at the end of your rotation. The next person walks in already in context — what fired, what's still smoldering, what shipped to prod this week, what to keep an eye on.

This is the **internal-platform-engineering-shape** example in the marketplace. Skills like this are the lowest-hanging fruit for a team like Bloomerang's: every engineering team already does handoffs, and they're almost always done badly.

---

## What it composes

Three internal-tools MCPs the platform team would own:

- **PagerDuty** — incidents during the rotation window
- **Slack** — `#oncall` thread context and incident-channel decisions
- **GitHub** — merged PRs (what shipped) and open PRs (what's landing during the next shift)

The skill describes the contract; none of those MCPs ship in this repo. That's the point of the marketplace pattern: skills are portable across whatever MCPs the platform has wired. When the platform team wires Sentry or Datadog next, this skill picks them up via the same composition.

---

## Example output

```
# On-Call Handoff — Sarah → Marcus
Rotation: 2026-05-30 → 2026-06-06

## Still in flight (top of pager)
- INC-4012 · Donation processing webhook retry storm · P2
  Last update: 14:22 — backoff deployed, queue draining, ETA 2h.
  Owner / next step: Marcus — verify queue is empty by 17:00; page Sarah if not.

## Resolved this rotation
- 11 incidents · 3 noise · top by severity:
  - INC-4001 · Bloomerang admin 500s on org search · P1 · stale index, reindexed
  - INC-3998 · Stripe webhook auth rotation · P2 · key rotated, runbook updated
  - INC-3994 · CSV export job OOM · P2 · bumped worker memory tier

## Shipped to production
- "Add idempotency keys to donation processor" · @amaya · reduces dupe-charge risk on retry
- "Migrate constituent search to OpenSearch 2.x" · @rj · should fix INC-4001 class
- "Cache health-check responses" · @sarah · -40% load on /healthz endpoint

## Open PRs that may land during your shift
- "Recurring-giving widget v2 rollout" · @platform-team · approved, blocked on CI flake (rerunning)
- "Notification preferences UI" · @growth-team · waiting on design review

## Watch list (next 24h)
- INC-4012 retry storm: if it recurs, the new backoff isn't enough — escalate to @reliability.
- Donation processor (just shipped idempotency): watch for any odd duplicate-charge reports.
- OpenSearch 2.x migration is at 60% rollout; tomorrow goes to 100%.

---
Sources: pagerduty (4 calls), slack (7 threads), github (12 PRs).
Stubs in this MVP: Sentry (error-group volume), Datadog (saturation alarms).
```

---

## Install

Same as any plugin in this marketplace — see the root [README](../../README.md#install) or the [meeting-prep README](../meeting-prep/README.md#install) for the install commands.

This plugin assumes PagerDuty, Slack, and GitHub MCPs are wired in the user's Cowork environment. If a connector is missing when `/oncall-handoff` runs, the skill will tell the user which tool it couldn't find rather than failing silently.
