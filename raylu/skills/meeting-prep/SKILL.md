---
name: meeting-prep
description: Use when the user wants to auto-prep upcoming intro calls from their calendar — reading Google Calendar, filtering out internal teammates and PE/VC fund attendees, and generating a Raylu meeting brief for every startup on the invite. Trigger phrases include 'prep my upcoming meetings', 'brief me on today's intro calls', and 'what should I know before my calls'.
---

# Meeting Notes from Calendar

Walk through these steps ONE AT A TIME with the user. Do not skip steps.

## Step A: Confirm setup

Before doing anything, confirm two connectors are available:

1. A Google Calendar connector (e.g. the Claude Google Calendar connector). If it isn't connected, stop and ask the user to enable it — do not try to fake it.
2. The Raylu MCP (you should see `get_company`, `meeting_prep_brief`, `firmographic_search` available).

Then ask the user:

- "What email domain is your firm? (e.g. `acmecapital.com`)" — used to filter out internal attendees.
- "Where do you want the briefs delivered?" Options: print in this chat (default), send to your email, post to a Slack DM, save as a Notion note. Use whichever delivery connector the user has available.
- "How far ahead should I look?" Default: next 75 minutes (so briefs land roughly an hour before each meeting).

## Step B: Pull upcoming meetings

Query the Google Calendar connector for events starting in the configured window. For each event, capture: event ID, title, start time (in the user's local timezone), and the attendee list (emails + RSVP status).

Skip an event if any of the following are true:
- Every attendee is from the user's firm domain (purely internal).
- The user has declined.
- The event title indicates an internal-only meeting ("1:1", "team sync", "standup") AND only firm-domain attendees are listed.

## Step C: Classify each external attendee's company

Build a deduped list of external email domains. Drop the user's firm domain and free-mail domains (`gmail.com`, `outlook.com`, `yahoo.com`, `proton.me`, etc.).

For each remaining domain:

1. Call `get_company` with the domain.
2. If the returned profile indicates the company is an investment firm — a PE fund, VC, family office, or LP rather than an operating company — **skip it**. That's the co-investor / LP side of the table, not the meeting target.
3. If `get_company` returns no record AND the domain matches a fund pattern (`.vc`, `*capital.com`, `*ventures.com`, `*partners.com` without a known operating product), treat as PE/VC and skip.
4. Otherwise (operating company, or unknown but not fund-shaped), **keep it** as a research target.

If an event has no remaining research targets, skip the event — it isn't an intro call with a new startup.

## Step D: Generate the brief

For each event with at least one research target, for each unique target company (dedupe by domain):

1. Call `meeting_prep_brief` with the company domain.
2. Compose a brief containing:
   - **Meeting:** title + start time (local) + counter-party company
   - **One-line company summary**
   - **Funding snapshot** — latest round, total raised, lead investors
   - **Team** — founders + leadership relevant to the meeting
   - **Recent signals** — news, hiring trends, product launches in the last 90 days
   - **3–5 questions** you'd ask in this meeting, tailored to the company

Combine briefs for one event into a single message. Never call `meeting_prep_brief` twice for the same domain in a run.

## Step E: Deliver

Send each brief to the channel the user picked in Step A.

After delivery, print a compact summary: "Briefed N upcoming meetings, skipped M (internal / PE-VC side / no external attendees)."

## Step F (optional): Schedule it

To get briefs that land ~1 hour before each meeting automatically, suggest the user create a scheduled task (Claude Scheduled Tasks, or any scheduler that can invoke a workflow):

- Cadence: every 30 minutes
- Lookahead: 75 minutes
- Track delivered event IDs across runs so the same event isn't briefed twice

## What NOT to do

- Do not run `meeting_prep_brief` for every attendee — dedupe by company domain per event.
- Do not research the user's firm domain or `@gmail.com`-style personal addresses.
- Do not research PE/VC funds — those are peers/LPs, not the target of an intro call.
- Do not use `lookup_company` (it writes to the user's Raylu project) when `get_company` (read-only) is sufficient.
- Do not extend the lookahead beyond 90 minutes — accuracy of "intro call with a new startup" classification drops.
- Do not re-brief the same event in a later scheduled run — dedupe by event ID.
