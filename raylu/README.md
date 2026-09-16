# Raylu

Company intelligence, market mapping, sourcing, and outreach workflows for Claude, powered by the [Raylu](https://raylu.ai) MCP server.

This plugin bundles the Raylu MCP connection plus a set of skills that drive Raylu's tools through high-leverage tasks. Once installed, describe what you want and Claude picks the matching skill — or invoke a skill explicitly from the `/` or `+` menu.

## Skills

- **start** — get oriented and confirm the Raylu connection.
- **companies** — company lookup, firmographic search, investor portfolios, team analysis, meeting prep.
- **market-intelligence** — analyze a market map; add AI enrichment columns.
- **sourcing** — set up a recurring sourcing routine.
- **ma-mapping** — build a scored M&A map anchored on a target.
- **outreach** — build and launch a multi-touch outreach sequence.
- **meeting-prep** — auto-prep intro calls from your calendar.
- **convert-skill** — convert a custom skill/prompt into a Raylu workflow.

## Connection

The plugin registers the Raylu MCP server (`https://mcp.raylu.ai/mcp`) automatically — see [CONNECTORS.md](CONNECTORS.md). You'll complete an OAuth login the first time a Raylu tool runs.

## Editing the workflows

Each skill is plain Markdown under `skills/<name>/SKILL.md`: a `description` (when Claude should use it) followed by step-by-step instructions. Copy, tweak, and reuse.
