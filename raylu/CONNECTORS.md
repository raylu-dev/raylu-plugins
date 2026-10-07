# Connectors

This plugin bundles a single connector: the **Raylu MCP server**.

| Connector | Type | URL | Auth |
|-----------|------|-----|------|
| `raylu` | Remote HTTP MCP | `https://mcp.raylu.ai/mcp` | OAuth (WorkOS) on first use |

When you install the Raylu plugin, the MCP server is registered automatically (see [`.mcp.json`](.mcp.json)). The first time a skill calls a Raylu tool, you'll be prompted to log in to your Raylu account via OAuth. After that, the connection persists.

The Raylu tools these skills rely on include:

- **Company data & search** — `search_companies`, `get_company`, `get_companies`, `firmographic_search`, `company_people_search`, `company_team_breakdown`
- **Lists and market maps** — `search_lists_and_maps`, `get_list_or_map`, `get_list_or_map_rows`, `create_list`, `add_companies_to_list_or_map`, `start_market_map`, `update_market_map`, `create_ai_enrich_column`, `enrich_list_or_map`, `create_ma_map`, `setup_ma_mapping`
- **Scoring** — `list_scoring_definitions`, `score_companies`
- **Outreach** — `find_company_contact`, `list_strategies`, `create_strategy`, `generate_campaign`, `start_campaign`
- **Meeting prep** — `meeting_prep_brief`
- **Automation** — `setup_sourcing_routine`, `get_run_status`

If a skill reports that a Raylu tool is unavailable, confirm the `raylu` MCP server is connected and that you've completed the OAuth login.

> **Enterprise note:** Some Claude Cowork organizations restrict which MCP servers can connect. If the Raylu server doesn't appear after install, an org admin may need to approve it.
