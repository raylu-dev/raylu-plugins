---
name: convert-skill
description: Use when the user wants to convert an existing custom skill, prompt, or workflow into a Raylu-optimized version — mapping external lookups (Crunchbase, LinkedIn, web search, spreadsheets) to Raylu MCP tools and flagging steps with no Raylu equivalent. Trigger phrases include 'convert this skill to use Raylu', 'turn this prompt into a Raylu workflow', and 'Raylu-ify this'.
---

# Custom Skill Converter

Take any custom Claude skill, prompt, or workflow and rewrite it as a Raylu-optimized version that uses our MCP tools. Walk through these steps ONE AT A TIME. Do not skip steps.

## Step A: Ingest the Source Skill
Ask: "Paste the skill, prompt, or workflow you want to convert. It can be a Markdown file, a system prompt, a set of instructions, or rough notes — anything that describes what you want Claude to do."

Read the full text carefully. Do NOT skim. Identify:
- The skill's name and primary objective
- The trigger conditions ("use when...")
- The discrete steps or stages
- The questions the skill asks the user along the way
- The external tools, websites, APIs, or manual processes the skill currently calls (Crunchbase, LinkedIn, Pitchbook, Google search, spreadsheets, etc.)
- Any output formats, tables, or artifacts the skill produces

Present a brief summary back to the user before doing anything else:

> "Here's what I understood from your skill: **{objective}**. It has **{N}** steps. It currently uses **{list of external tools or manual processes}**. Confirm before I convert it, or correct anything I got wrong."

## Step B: Map External Actions to Raylu Tools

For each step in the source skill, identify which Raylu MCP tools can replace or enhance it. The full Raylu tool catalog is below — use these as the building blocks:

**Company data & lookup**
- `search_companies` — disambiguate by name across the warehouse
- `get_company` / `get_companies` — full firmographic profile (single or bulk)
- `research_company` — research and start tracking a company
- `list_companies` — list all saved companies

**Firmographic search**
- `firmographic_search` — structured search with filters: headcount, country, state, city, last_funding_stage, last_funding_amount, total_funding_amount, financing_status, founded_date, revenue, monthly_visits, industries, keywords, all_investors, all_funding_leads

**Market maps**
- `start_market_map`, `update_market_map`, `get_market_map`, `get_market_map_data`, `list_market_maps`
- `create_ai_enrich_column`, `enrich_market_map` — add and run AI enrichment columns
- `create_ma_map` — build an M&A-anchored map in one operation
- `add_company_to_market_map`

**Lists**
- `create_list`, `get_list`, `list_lists`, `add_company_to_list`

**Contacts & outreach**
- `find_company_contact` — verified email + phone for the primary contact
- `list_strategies`, `create_strategy` — outreach cadences
- `generate_campaign` — generate a full sequence
- `start_campaign`, `pause_campaign`, `cancel_campaign`, `complete_campaign_step`
- `update_email`, `rewrite_email` — edit individual emails
- `get_campaign`, `list_campaigns`
- `get_outreach_schedule`, `get_daily_send_count`

**Other**
- `setup_sourcing_routine`, `setup_ma_mapping` — bootstrap recurring workflows
- `get_run_status`

For each source step, output a mapping in this table format:

| Source step | Raylu tool | What changes |
|---|---|---|
| "Look up the company on Crunchbase" | `get_company` | Pulls Raylu's enriched profile instead of opening Crunchbase manually |
| "Find a contact on LinkedIn" | `find_company_contact` | Returns verified email + phone in one call |
| "Filter to Series A SaaS companies" | `firmographic_search` | Structured filter on `last_funding_stage` and `keywords` |

Present the full mapping table to the user. Explicitly flag any source steps that have NO Raylu equivalent — those should stay as-is or be handled outside the tool chain.

## Step C: Identify Optimization Opportunities
Look beyond 1-to-1 replacement. Suggest improvements the original skill couldn't access:
- Can a market map replace a multi-step research loop?
- Can an AI enrichment column automate something the user was doing manually per company?
- Can `research_company` + `find_company_contact` + `generate_campaign` be chained instead of run separately?
- Can a scheduled routine (`setup_sourcing_routine`) replace something the user runs ad-hoc?
- Can a saved list (`create_list` + `add_company_to_list`) replace a spreadsheet?

Present the optimizations and ask the user which to apply. Do NOT silently change the skill's intent — flag every meaningful structural change before making it.

## Step D: Rewrite in Raylu Workflow Style
Generate the new skill using this exact structure (matches the existing Raylu workflows):

1. **Title** — preserve or refine the original skill's name
2. **One-line objective** — what the skill does, written as an imperative
3. **Top-level rule** — e.g., "Walk through these steps ONE AT A TIME. Do not skip steps."
4. **Optional autonomy rule** — if Claude should act without confirmation in certain cases, state it up front
5. **Steps** — `## Step A`, `## Step B`, etc. Each step must:
   - Lead with the user-facing question (if any), formatted as `Ask: "..."`
   - Name the exact Raylu tool calls in order
   - Explain what to do with the output and when to confirm with the user
6. **Output format** — if the original produced a table, preserve the column schema. If it produced a recommendation, preserve the framing.

Tone match: direct, imperative, no fluff. Read M&A Mapping, Sourcing Routine Setup, and Outreach Sequence Builder for reference — match their cadence and structure.

## Step E: Validate and Hand Off
Present the converted skill to the user inside a fenced code block (so it copy-pastes cleanly). Then ask:
1. "Does this preserve the intent of your original skill?"
2. "Any steps you'd like to drop, reorder, or rename?"
3. "Want me to save this as a Markdown file you can share with your team or add to the Raylu MCP Workflows library?"

If the user approves, format the final version as Markdown with the same conventions used by the workflows in the Raylu MCP Workflows library.
