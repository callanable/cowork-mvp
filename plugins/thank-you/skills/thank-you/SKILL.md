---
name: thank-you
description: Draft personalized thank-you emails for recent donors by composing Bloomerang giving history with Gmail context. Use when the user asks to "draft thank-yous", "thank recent donors", "follow up on this week's gifts", or runs /thank-you. Aim to send within 48 hours of the gift.
user-invocable: true
argument-hint: [days-back | "today"]
---

# /thank-you — Draft thank-yous for recent donors

Pulls donations from the last N days out of Bloomerang, gathers per-donor context (giving history, prior interactions, recent email exchanges), and drafts a personalized thank-you email per gift — ready to review and send.

Composes:

- **bloomerang** (from [bloomerang-mcp](https://github.com/callanable/bloomerang-mcp)) — `list_recent_donations`, `get_constituent`, `list_giving`, `list_interactions`, `log_interaction`
- **Gmail** — `search_threads` (recent context), `get_thread` (substance), and `send_message` (optional, only on explicit user confirmation)

---

## When to trigger

User typed `/thank-you` (with or without an argument), or asked to "draft thank-yous", "thank recent donors", "follow up on this week's gifts". Don't trigger off general donor mentions — only when the user is explicitly working the thank-you queue.

## Argument handling

`$ARGUMENTS` is:

- empty → default to last 7 days
- a number (e.g. `3`) → that many days back
- `"today"` → just today
- an ISO date (e.g. `2026-06-01`) → since that date

## Workflow

### 1. Pull recent gifts

Call `bloomerang.list_recent_donations` with `since = <computed date>`. If the result is empty, tell the user "No new gifts since X — nothing to thank" and stop.

If 10+ gifts, ask the user whether to proceed with all or filter to a subset (largest first, or specific fund/campaign).

### 2. For each gift, gather context

Per gift, in parallel where possible:

- `bloomerang.get_constituent(constituent_id)` — name, email, lifetime giving, last gift before this one
- `bloomerang.list_giving(constituent_id)` — full history; you only need the count and the prior gift to know if this is first-time, second-time, lapsed-now-returning, or major-donor-repeat
- `bloomerang.list_interactions(constituent_id)` — surface any recent open thread or commitment the thank-you should acknowledge

If the donor has an email and the user wants high-context drafts, also:

- Gmail `search_threads` with `from:<email> OR to:<email>` (max 3 most recent)
- Read the most recent thread if it's likely substantive (length > 1 reply)

### 3. Classify the gift

Tag each gift with one of:

- **first-time** — no prior giving history
- **repeat** — has given before, last gift within 12 months
- **lapsed-returned** — has given before, but the prior gift was >12 months ago
- **major-repeat** — repeat donor whose lifetime giving is in the top 10% of the CRM (treat anything above $10K lifetime as major for the demo; in production this comes from a config)

The classification shapes the email tone:

- first-time → welcoming, brief, set expectation for future updates
- repeat → warm and specific, reference the relationship by name
- lapsed-returned → explicitly notice the return ("we missed you — your support means a lot")
- major-repeat → personal from the executive director / fundraiser by name, reference the relationship history, no boilerplate

### 4. Draft each email

Use this format. One block per gift.

```
─── Gift: <constituent_name>, $<amount> on <date> — <classification> ───
To: <email>
Subject: Thank you for your gift to <fund or campaign>

<2–4 sentence body that:
 - thanks them specifically for the gift (amount, fund, date)
 - references either: their giving history (if repeat/lapsed), the campaign
   context, or a recent interaction that's relevant
 - says what their gift will fund in concrete terms (NOT "general operating")
 - closes with a human sign-off, not a wall of legal/tax language>

— <user's name, inferred from Gmail signature; ask if unknown>
```

If a donor has no email on file, output the draft with `To: <no email on file — needs follow-up>` and a one-line note suggesting they get added.

### 5. Offer to send

After all drafts are displayed, ask:

> "Drafted N thank-yous. Want me to (1) send these via Gmail, (2) save as Gmail drafts, or (3) just copy them?"

- **send** — only on explicit confirmation. Use Gmail `send_message` per draft. After each send, `bloomerang.log_interaction({ constituent_id, type: "Email", subject, body })` so the CRM has the record.
- **drafts** — Gmail `create_draft` per email, no log_interaction yet (the user hasn't actually sent).
- **copy** — just print, do nothing. Default.

Never send without explicit confirmation. Even if the user said "just send them" earlier in the conversation, confirm once more at this step — these are real donor relationships and a misfire is expensive.

## Edge cases

- **MCP not wired**: if `bloomerang.*` tools aren't available, tell the user "The Bloomerang MCP isn't wired in this Cowork environment. Add it via [bloomerang-mcp](https://github.com/callanable/bloomerang-mcp) or ask your admin." Stop.
- **Recurring gifts**: don't thank for every monthly installment — group by donor and thank for the program once per quarter unless it's the very first installment.
- **Anonymous gifts**: skip silently. The skill is for relational thank-yous; anonymous donors are handled differently.
- **Refunds / chargebacks**: skip silently.
- **Sensitive notes on the gift** (e.g. memorial, in-honor-of): surface in the draft body explicitly and tonally appropriately. Don't bury the lede.

## Tone

Specific, warm, short. Avoid:

- "We are so grateful for your generosity" (every nonprofit email opens with this)
- "Your gift makes our work possible" (says nothing)
- Long paragraphs about the org's mission

Prefer:

- Naming the specific fund and what it does
- Referencing prior context if available
- Sounding like the user wrote it, not a template
