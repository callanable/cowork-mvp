---
name: meeting-prep
description: Generate a one-page brief for an upcoming calendar meeting by pulling related Gmail threads and Drive docs. Use when the user asks to "prep for a meeting", "get ready for a call", "what do I have coming up", or wants context on a specific upcoming event.
user-invocable: true
argument-hint: [event-title-or-keyword]
---

# /meeting-prep — Pre-meeting brief

Pulls the user's next calendar event (or one matching a keyword), gathers related Gmail threads and Drive docs, and outputs a single-page brief.

The skill composes three MCPs that are expected to be wired in the user's environment via Cowork's Anthropic Connectors panel:

- **Google Calendar** — `list_events`, `get_event`
- **Gmail** — `search_threads`, `get_thread`
- **Google Drive** — `search_files`, `read_file_content`

Tool names above are semantic; the actual MCP-prefixed names depend on which Google MCP server is wired. Use the available tool whose suffix matches.

---

## When to trigger

User typed `/meeting-prep` (with or without an argument). Don't trigger from inbound channel messages or automation — pre-meeting context belongs to the user.

## Argument handling

`$ARGUMENTS` is either empty (→ next upcoming event) or a keyword/phrase (→ search upcoming events for a title match, then fall back to attendee name match).

## Workflow

### 1. Identify the target meeting

- No args: call Calendar `list_events` with a window of `now` → `now + 7 days`. Pick the soonest event that has at least one attendee other than the user.
- With args: call `list_events` over the same window, filter for case-insensitive substring match on summary/description; if none, match on attendee email or display name. If multiple match, pick the soonest and note the others at the bottom of the brief.
- If nothing matches: tell the user briefly ("No upcoming events match X in the next 7 days") and stop.

Call `get_event` on the chosen event to retrieve full attendees, description, attached files, conferencing link.

### 2. Identify external attendees

Split attendees into **internal** (matching the user's own email domain) and **external** (everyone else, excluding the user themselves and any `resource.calendar.google.com` rooms).

For the brief, focus context-gathering on **external** attendees. List internal attendees by name only — no context-gathering — unless there are zero external attendees, in which case treat internals as the targets.

### 3. Pull email context per attendee

For each target attendee (cap at 5 to keep latency sane; if more, pick the organizers and most-recently-emailed):

- Gmail `search_threads` with query `from:<email> OR to:<email>`, max 5 most recent threads.
- For the top 1–2 most relevant threads (recency + length), `get_thread` and pull a 2–3 sentence summary of the substantive content.

Note any **open asks** — questions the attendee asked that the user hasn't replied to, or commitments the user made that haven't been delivered.

### 4. Pull Drive context

- If the calendar event has attached files: read each via `read_file_content`, summarize in 1–2 sentences.
- Then `search_files` with the event title as query; include up to 3 recent matches not already attached. Skim each, include only if clearly relevant to the meeting topic.

### 5. Output the brief

Use this exact format. Keep total length under one screen.

```
# Meeting Prep — <event title>
<start time, local> · <duration> · <conferencing link if present>

## Attendees
- <Name> <<email>> — <one-line role/context inferred from email signature or domain>
- <Name> <<email>> — internal

## Context
<2–4 sentence synthesis of relationship state across the email history.
What is this meeting likely about? What's the latest substantive exchange?>

## Open threads
- <Attendee>: <one-line summary of an open ask or pending commitment>
- <Attendee>: <one-line summary>

## Relevant docs
- <Doc title> — <one-line why it matters>

## Suggested talking points
- <Point 1 — derived from open threads or doc content>
- <Point 2>
- <Point 3>
```

If a section has no content, omit the heading entirely rather than printing "None." Don't pad.

## Edge cases

- **No upcoming meetings in window**: say so in one line and exit.
- **Only-internal attendees**: still produce a brief, but lean on Drive docs and the event description rather than email history.
- **No matching emails or docs**: produce a minimal brief with just the event details and a note that there's no prior context to draw on.
- **Recurring events**: prep the *next* instance only. If prior instances of the recurring event have notes in Drive (e.g. a "<series-name> notes" doc), surface the most recent.
- **Sensitive content**: if email threads contain obvious sensitive material (compensation, legal, HR keywords), summarize at a higher level rather than quoting.

## Tone

Neutral, factual, no speculation. If you don't know something, omit it — don't guess. The user is reading this 5 minutes before a call; clarity beats completeness.
