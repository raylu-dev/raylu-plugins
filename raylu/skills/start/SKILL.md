---
name: start
description: Use when first getting oriented with the Raylu plugin, when the user asks what they can do with Raylu, or to verify the Raylu connection is working. Greets the user, checks that the Raylu MCP server is connected, and lists the available Raylu workflows.
---

# Raylu — Start Here

You are helping someone get oriented with the Raylu plugin. Walk through these steps in order.

## Step 1: Welcome

Display this message:

```
Raylu for Claude

Raylu brings live company intelligence, market mapping, sourcing, and outreach into Claude.
This plugin gives you ready-to-run workflows backed by Raylu's data and tools.
```

## Step 2: Check the Raylu connection

Confirm the Raylu MCP server is connected by checking that Raylu tools are available (for example `search_companies`, `firmographic_search`, `get_market_map_data`, `meeting_prep_brief`).

- If the tools are available, say: "Raylu is connected — you're ready to go."
- If they are not, tell the user to install/enable the Raylu plugin and complete the OAuth login the first time a Raylu tool runs, then try again. Do not fake tool calls or answer company questions from general knowledge.

## Step 3: Show what's available

Present the available workflows and what each one does:

- **Companies** — look up a company, run a firmographic search, explore an investor's portfolio, analyze a team, or prep for a meeting.
- **Market Intelligence** — analyze a market map and add AI enrichment columns on the fly.
- **Sourcing** — set up a recurring routine that discovers and reaches out to new companies on a schedule.
- **M&A Mapping** — build a scored, AI-enriched M&A map anchored on an acquisition target.
- **Outreach** — build and launch a multi-touch outreach sequence for a specific contact.
- **Meeting Prep** — auto-prep upcoming intro calls from your calendar.
- **Convert Skill** — turn an existing custom skill or prompt into a Raylu-optimized workflow.

Then ask: "What would you like to start with?" and hand off to the matching workflow.
