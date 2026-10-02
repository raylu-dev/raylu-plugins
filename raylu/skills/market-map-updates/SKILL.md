---
name: market-map-updates
description: Update one or more Raylu market maps by rerunning them, then report the website domains added grouped by map, score the newly added companies with the fund's deal score, and surface standout highlights with the specific scoring factors behind each score. Always generates the update on the day it is invoked, whether the user wants a one-time update or a recurring schedule (set up via Claude scheduled tasks) on top of it. Optionally emails the results as a designed HTML email with one CSV per market map, and labels companies with a "Date added" field. Never launches outreach. Use when the user asks for market map updates, new companies/domains after a rerun, what's new in a market map, recurring/scheduled market-map updates, or delivery of update results by email.
---

# Market Map Updates

## Overview

Update market maps by rerunning them and compare membership by website domain across the before/after, for one or more maps at once. Treat the domain set as the source of truth for the diff unless the user explicitly asks for richer company-level analysis. Once the diff is computed, automatically score the newly added companies and surface standout highlights, grouped by market map. Optionally email the results, and as the last step label every company in each map with a "Date added" field. **This skill never removes companies from a market map.** It reruns maps with all existing companies kept, and never deletes or prunes anything. This skill also never launches outreach.

This skill is used by many different funds, each with their own custom deal-scoring definition — different node names, different weightings, different DAG structures. Nothing in this skill should assume a specific scoring definition's internals. Everything below that touches scoring works off the *shape* of the data (a node has a point value and a descriptor, regardless of what it's called), not hardcoded node names.

Walk through Steps 0, 0.5 and 1 with the user directly. Steps 2–8 are a **fixed procedure** — identical whether it runs live in this conversation or is embedded as the prompt for the recurring scheduled task.

---

## Step 0: Select Market Map(s) to Watch

Ask: "Which market map(s) do you want me to update? You can pick one or several."

- Call `list_market_maps` to present candidates.
- If the user names one or more by name/description, use `get_market_map` to confirm each exact match.
- If any name is ambiguous, ask a short clarification rather than guessing.
- Store the confirmed map ID(s) — everything below runs per map, then combines for reporting.

## Step 0.5: Email Check (before any rerun is started)

Check whether the user's account is connected to an email service (Gmail, Outlook, or similar) by looking at the tools actually available (load deferred tools with ToolSearch or list connectors if needed). Never send a test email to find out.

- **If an email connector is connected:** ask exactly one delivery question: "Do you want me to also email the report, or just deliver it in chat?" If they want the email, confirm the address (default: the account's own address).
- **If none is connected:** skip this step silently and do not mention email.

This is the only question about delivery. Never ask about Slack channels, other destinations, or where the report should go. The report is always delivered in chat (or in the scheduled task's own output), plus the email if requested.

Settle this now, before `start_rerun_market_map`: the rerun takes a while and nobody should be left waiting on a question afterwards. Record the answer (yes/no, address, connector) for Step 7.

## Step 1: One-Time, or Also Recurring?

**The update always runs today.** Whatever the user answers here, run Steps 2–8 live, in this conversation, for the map(s) selected in Step 0. Never replace the live run with a scheduled one, and never wait until tomorrow.

Ask: "Do you want just this update today, or this update today plus a recurring schedule?"

**If "just today":** run Steps 2–8 and do not mention scheduling at all.

**If "today plus a recurring schedule":** run Steps 2–8 now, then set up the recurring task after the report is delivered.
- Ask: "How often do you want this to run? I'd recommend quarterly at most — more frequent than that and it's unlikely new companies will have surfaced. Quarterly, semiannual, or annual?"
- Based on the answer, suggest anchor dates and let the user adjust:
  - Quarterly → Jan 1, Apr 1, Jul 1, Oct 1
  - Semiannual → Jan 1, Jul 1
  - Annual → Jan 1
- Ask preferred time of day if it matters to the user; otherwise default to 9:00 AM local.
- Build the **fixed Steps 2–8 prompt** as self-contained text: hardcode the map ID(s)/name(s) from Step 0, the scoring definition, and the Step 0.5 email answer (email address and connector, or "chat only"). Scheduled runs have no memory of this conversation — nothing about map selection or delivery can be left implicit. Do not ask any further delivery questions.
- Call `create_scheduled_task` once: `taskId`: `<slug>-recurring`, `cronExpression`: built from the chosen anchor dates (e.g., quarterly on the 1st at 9am → `0 9 1 1,4,7,10 *`; semiannual → `0 9 1 1,7 *`; annual → `0 9 1 1 *`), `description`: "[Frequency] update for [map names]." There is no separate first-check task, because the first update already ran today.
- Confirm the task was created and show its next run date.

---

## Steps 2–8: The Fixed Procedure

*(This section is the entire content of the scheduled-task prompt when Step 1 sets up a recurring schedule. In a live run, follow it directly in this conversation.)*

### Step 2: Per-Map Rerun and Diff

For **each** selected map:
1. Before reading, call `get_run_status` with `feature: "market_map"` to confirm no build is already in flight for that map.
2. **Pre-rerun snapshot.** Call `get_market_map_data` with columns `["name", "website", "headcount", "employeesCountChangeYearly", "totalFundingAmount", "lastFundingAmount", "lastFundingStage", "lastFundingDate"]`. Use `website`, not `primaryDomain` (systematically empty). Note any rows with empty websites as a caveat. Pulling these extra firmographic columns alongside name/website costs nothing extra — it's the same single call already being made for the diff, just wider rows — so there's no reason to fetch them separately later. This is the cheap path; keep it this way rather than reaching for a per-company scoring call just to get a headcount number. **Keep this pre-rerun domain set — Step 8 needs it.**
3. **Rerun the map.** Call `start_rerun_market_map` with the map ID to get a `sessionId`. For a plain as-is rerun, call `update_rerun_market_map` with that `sessionId` and `confirm: true` (default `keepCompaniesMode: "all"`). Always keep every existing company: never use `keepCompaniesMode: "score"` or any pruning option, even if asked mid-run. Nothing is ever removed from a market map by this skill. The confirmed rerun runs asynchronously — do not treat the confirm response as finished.
4. **Post-rerun snapshot.** Check completion via a single `get_run_status` call (`feature: "market_map"`) — do not poll in a loop, and verify the completed task matches the rerun just started, not a prior one. Once complete, fetch rows again with `get_market_map_data` using the same columns as the pre-rerun snapshot.
5. **Compute the domain diff** for this map. Normalize domains (lowercase, strip protocol/`www.`/trailing path/query/hash, trim whitespace, ignore empty). `added = post - pre`, `removed = pre - post`, `unchanged = |pre ∩ post|`.
6. Because the rerun keeps all companies, `removed` should be empty. If it is not (the platform dropped something on its own), mention it in one line in the chat report only. Never act on it, and leave it out of the email and CSVs.

### Step 3: Report Results (grouped by market map)

Present results grouped by map, one subsection per map, in this order:

**1. One-line summary.** State the total added and, once scoring (Step 5) has run, the breakdown by score tier, using whatever tier labels the scoring definition actually returns — don't assume "High/Medium/Disqualified" are universal, since different funds name their tiers differently. Example shape: "<N> companies added — <count> <tier label>, <count> <tier label>, ...". If scoring hasn't completed yet, give the added/removed counts only and say scoring is still in progress.

**2. Standouts.** One bullet per standout company — see Step 6 for the selection rule and bullet format. This is the section people will actually read; keep it tight and skip anything not worth a human's attention.

**3. Removed domains.** Nothing is ever removed by this skill, so normally say nothing here. If the diff unexpectedly shows removed domains, list them in one line as informational, with no action taken.

If a map had zero net differences, say so explicitly in its subsection (e.g., "No new companies were added to this map in this run") — never silently omit it. If **no map in the set** had any additions, state that plainly as the final result and stop here — do not proceed to Step 4.

After all maps' subsections, give a combined total across all maps (companies added).

### Step 4: Deliver the Added-Companies CSV

Produce one CSV across all maps with exactly these columns, in this order: `market_map, company_name, website_domain, score, headcount, description`. Nothing beyond these six columns — the full market map has dozens of columns, but this export is meant to be scanned quickly, not to replace the map itself. Use the normalized domain; keep the display company name and firmographic values from the post-rerun snapshot. Include additions with a blank website as empty `website_domain` rows. Descriptions are one complete sentence with no ellipses and no cut-off text (see the description rule in Step 7). Never include revenue.

Sort/group by `market_map`. If a company was newly added to more than one selected map in this run, include one row per map it was added to (so grouping by map stays accurate), but dedupe it in Steps 5–6 so it's only scored/highlighted once.

Name the file `market_map_updates_added_<YYYY-MM-DD>.csv` (or the single map's name if only one was selected).

### Step 5: Score Added Companies

Runs automatically — never ask whether to score.

- Call `list_scoring_definitions`. If exactly one exists, use it. If more than one, you MUST ask the user which to use — never choose on their behalf. (In a scheduled run with no live user, use the definition specified in the embedded prompt at setup time.)
- Deduplicate added companies across maps, then call `score_companies` with the full deduped set (`company_ids` or `domains`) and the resolved `definition_id`. This is async for unscored companies and returns no run to poll — do not call `get_run_status` for it. Wait roughly a minute or two, then call `get_score_breakdown` to read scores + tiers. Do not re-call `score_companies` just to check results.
- Use the scores/tiers from this step to populate the one-line tier summary in Step 3 and the `score` column in the Step 4 CSV — this covers every added company and doesn't require anything beyond what this step already fetches.
- **Do not run the full node-level scoring breakdown (below) for every added company.** That's the expensive part — `get_score_breakdown` returns a company's entire scoring logic tree, which can be a very large response across even a few dozen companies. Reserve it for the standout companies selected in Step 6, not the full added list.

### Step 6: Standout Highlights

**Selecting standouts:** take the top 5 added companies by score (or all of them, if fewer than 5 were added). This keeps the section short on a small diff and bounded on a large one. Skip disqualified/lowest-tier companies here — they belong in the CSV, not the highlights.

**For each standout**, pull `get_score_breakdown` for that company (if not already fetched in Step 5) and build one bullet with two parts:

*What it is:* one to two sentences — what the company does and its headcount. Never mention revenue. Pull these from the Step 2 firmographic columns already in hand — no extra call needed.

*Why it scored high:* this is where the generic, definition-agnostic scan happens. In `effectiveResult.nodeResults`, every node that actually contributes to the score carries a numeric point value — either directly (range and decision-table nodes) or inside a `{score, value, reasoning}` object (AI-judgment nodes). Skip any node whose value is a plain boolean; those are disqualification-gate checks, not score contributors, regardless of what the fund named them.

From that filtered set:
1. Take the **top 3 nodes by points contributed** — this is what "why it scored high" means, and it works on any DAG because it's driven by the actual point values, not by node names.
2. Separately, always also surface the firmographic nodes tied to headcount, revenue, and total funding — find these not by node ID (which varies per fund) but by matching on the field the node reads from (`metadata.field` or the node's input field equal to `headcount`, `revenue`, or `totalFundingAmount`). These three fields are part of Raylu's standard company schema, so this hook holds across every fund's map even though node IDs and labels differ. If a fund's DAG has no node scoring on one of these fields, just omit it — don't force it.
3. If a node appears in both sets, only mention it once.

**Format each factor descriptively, never with a vague adjective:**
- Range/table nodes: `<field> of <raw value> (<matched range or row label>, +<points> points)` — e.g., "Headcount of 82 employees (25–500 employee range, +8 points)" or "No institutional funding on record (+6 points)" for a zero/default match. The raw value comes from `metadata.inputValue`; the bucket label comes from `metadata.matchedRange.label` or `metadata.rowLabel`.
- AI-judgment nodes: `<classified value> (+<points> points)` followed by a short (one-sentence, trimmed) excerpt of that node's own `reasoning` field — this is the model's actual justification generated at scoring time, not a re-interpretation of the company description layered on afterward.

If scoring hasn't completed by report time, say so explicitly rather than showing a blank score. If scoring nodes return errors (for example a failing lookup node), say so in the in-chat report instead of presenting the scores as final.

### Step 7: Deliver

Deliver the report (Step 3) and CSV (Step 4) in the conversation (or, in a scheduled run, as the task's own output).

**If email delivery was chosen in Step 0.5**, also send an HTML email through the connected email tool to the confirmed address.

- **Subject:** `We found <N> more companies in markets you're tracking (<date>)`, with a straight apostrophe, never a curly one or a "?". For a single map, use `<Map name> updated: <N> companies added (<date>)`.
- **Attachments:** one CSV per market map (same six columns as Step 4, only that map's rows), named `market_map_updates_added_<map-slug>_<YYYY-MM-DD>.csv`. Include a plain-text alternative body.
- **Links must be real.** Each market map title links to the map in Raylu (`https://app.raylu.ai/market-map/<map id>`, built from the ID returned by `get_market_map` or `update_rerun_market_map`, only for maps that belong to the connected account). **Each company name links to that company's own website** (`https://` + the normalized domain). Never use `app.raylu.ai/companies/...` links. Never put sample or simulated data in a real email.
- **Nothing extra:** no warning banner, no data-caveats section, no "How this was generated" appendix, no outreach line, no "Sent by Claude via Raylu", no "Forward to a colleague", no "Unsubscribe" link. Caveats stay in the in-chat report. (If the email is sent through an email-marketing tool such as MailerLite that requires an unsubscribe tag, add the tool's own `{$unsubscribe}` link as one small line in the footer and nothing else.)
- **Encode every non-ASCII character as an HTML entity** (`&middot;`, `&quot;`, `&rsquo;`, `&#8482;`, etc.) so no symbol can ever render as a "?" box. Use a straight apostrophe in "you're".

**Look: match Raylu's MailerLite campaigns.** Table layout with inline CSS and `bgcolor` attributes, one 600px card, centered, 16px rounded corners, 1px border `#DCD6CA`.

- Page background `#E4DCCF`; card `#F3ECE5`; headline text `#14210F`; green `#235C23`; body text `#3A4232` / `#55604A`; muted `#6B7360` / `#8A917C`; dividers `#DCD6CA`. Headings in 'PP Eiko' falling back to Georgia; body in 'PP Neue Montreal' falling back to Arial.
- **Header:** the Raylu logo at the top left (`https://storage.mlcdn.com/account_image/1455915/rUkvJyLgPXNUne7W37JU1oljTUNEKRS4TBkDRNDP.png`, 132x50), then the date in muted 14px, then the headline **"We found <N> more companies in markets you're tracking"** in 30px serif (N = total added across all maps). No stat tiles.
- **Top-right green glow:** a soft radial gradient (`radial-gradient(130% 90% at 100% 0%, rgba(35,92,35,0.16) 0%, rgba(35,92,35,0.06) 40%, rgba(243,236,229,0) 72%)`) on the header cell. Gmail removes gradients and every `<img>` from messages sent through its connector, so for those sends use the fallback: a text logo (a bold green "//" followed by "raylu" in 36px Georgia) and, to the right of it, a row of six narrow cells stepping from `#EFE9E1` to `#DADBCE` to hint at the glow.
- **One section per market map**, in the order selected:
  1. The map name as a 22px serif heading in green, hyperlinked to the map.
  2. **Directly under the title, one light-grey (`#A3A89A`), italic, 14px line:** "Every company is rated out of 100 using your <scoring definition name>." Use the actual name of the fund's scoring definition from Step 5.
  3. A counts line in 14px `#55604A`: "<N> added · <count> <tier>, <count> <tier>, ..." using the scoring definition's own tier labels.
  4. Up to 5 numbered rows (01–05), each separated by a 1px divider: the **company name in bold, underlined, linked to its own website**, with its score beside it in green (" · 90"); then one complete-sentence description (see below); then, **on its own line** in 12px muted text, the firmographics.
  5. If more companies were added than shown, a small muted "+ N more in the attached CSV".
  If a map had zero additions, still include its section and say so.
- **Description rule:** one complete sentence on what the company does, written as a finished thought. **No ellipses, no "...", no trailing fragments** such as "that automates." If the source description is long, rewrite it as one shorter complete sentence instead of cutting it.
- **Firmographics line (its own line, never merged into the description):** as many of these as have data, separated by " · ": "<n> employees", "+X% yearly headcount growth" (from `employeesCountChangeYearly`), "$X raised" (total funding), and "last round $X Stage YYYY" (from `lastFundingAmount`, `lastFundingStage`, `lastFundingDate`). **Never include revenue.** Omit any figure with no data; never write "not available".
- **Do not mention deal-score points in the email.** The score number beside the name is enough.
- **Footer:** a 1px dark divider, then "**Raylu**" with the tagline "The AI origination engine for private markets.", then "369 Lexington Ave, New York, United States", then links to X (`https://x.com/raylu_dev`), LinkedIn (`https://www.linkedin.com/company/raylu-ai/`) and `https://raylu.ai/`.

If the email fails to send, say so in one line and continue — never block Step 8 on it.

### Step 8: Label the Maps (last step)

Use the `set_field_values` tool. If it is not available in this session (it may not exist yet — check with ToolSearch), skip this step, say in one line that the "Date added" labeling was skipped because the tool is unavailable, and never emulate it with other tools.

For **each** selected map:
1. Use a custom field named exactly **"Date added"** on the map. `set_field_values` creates the field if it does not exist yet, in the same call that writes the values; do not look for a separate field-creation tool. Follow the tool's own schema for the exact arguments.
2. Every company in this run's **added** set gets today's date, formatted `YYYY-MM-DD` in the user's timezone.
3. Every company in the Step 2 **pre-rerun snapshot** gets the value **"In original map"**. Do not overwrite a date written by an earlier run: if a company already has a "Date added" date, leave it alone and only fill blank values with "In original map".
4. A company added to more than one map is labeled once per map. 
5. Write in batches, confirm the counts, and report one line per map (e.g., "Labeled 60 new and 193 original companies in <map>"). If a write fails, do not retry or work around it: skip the rest of that map's labeling, say so in one line in the chat report, and continue. Labeling never blocks the report or the email.
