# Raylu Plugins

A Claude plugin marketplace for [Raylu](https://raylu.ai). Adds Raylu's company intelligence, market mapping, sourcing, and outreach workflows to Claude — as skills you can run in **Claude Desktop, the web app, Claude Cowork**, and **Claude Code**.

The marketplace hosts one plugin, **Raylu**, which bundles:

- **7 workflow skills** that walk Claude through high-leverage Raylu tasks end to end.
- **A `start` skill** to get oriented and confirm the connection.
- **The Raylu MCP connection** (`https://mcp.raylu.ai`), wired up on install — you just log in via OAuth the first time a tool runs.

## Install

### Claude Desktop / web / Cowork

1. Open **Customize** (left sidebar) → **Plugins**.
2. Under **Personal plugins**, click **+** → **Add marketplace**.
3. Enter this repository: `raylu-dev/raylu-plugins`.
4. Install the **Raylu** plugin.
5. The first time a Raylu skill runs, complete the OAuth login to your Raylu account.

### Claude Code

```bash
/plugin marketplace add raylu-dev/raylu-plugins
/plugin install raylu@raylu
```

## Skills

| Skill | What it does |
|-------|--------------|
| Companies | Company lookup, firmographic search, investor portfolios, team analysis, meeting prep |
| Market Intelligence | Analyze a market map; add AI enrichment columns |
| Sourcing | Stand up a recurring, automated sourcing routine |
| M&A Mapping | Build a scored M&A map anchored on an acquisition target |
| Outreach | Build and launch a multi-touch outreach sequence |
| Meeting Prep | Auto-prep intro calls from your calendar |
| Convert Skill | Rewrite any custom skill/prompt as a Raylu workflow |

## Requirements

- A **Raylu account** (log in via OAuth on first use).
- Installing marketplace plugins in Claude requires a **Pro or Team** plan.

See the plugin [README](raylu/README.md) and [CONNECTORS.md](raylu/CONNECTORS.md) for details.
