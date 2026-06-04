---
name: sourcing
description: Use when the user wants to set up a recurring, automated sourcing routine that discovers new companies on a schedule — defining an investment thesis, inspiration sources, a scoring threshold and exclusions, an outreach strategy, and a cadence. Trigger phrases include 'set up a sourcing routine', 'automatically find new companies every week', and 'build a recurring deal-sourcing pipeline'.
---

# Sourcing Routine Setup

Walk through these steps ONE AT A TIME with the user. Do not skip steps.

## Step A: Investment Thesis
Ask: "What space are you focused on? Describe the types of companies you're looking for."
From their answer, derive: category, description, subcategories, regions.
Check list_market_maps for existing maps — ask if they want to use one or create new.
Ask about firmographic preferences: headcount range, funding stage, geography.

## Step B: Inspiration Sources
Ask: "Where should I look for new company leads each time this runs?"
Options:
1. Web search — scan for recent news and funding in their space
2. Existing market map — check for newly scored companies
3. Firmographic search — fresh database query
4. All of the above (recommended)

## Step C: Scoring & Filtering
Ask: "How do you evaluate whether a company is worth sourcing?"
Check if they have a scoring definition in Raylu.
Ask: minimum score threshold (default 60), exclude CRM companies (default yes), exclude companies with existing campaigns (default yes).

## Step D: Outreach Strategy
Call generate_campaign without a strategyId — this returns available strategies.
Present the options and let the user pick.
Ask: should campaigns auto-start or stay as drafts?

## Step E: Schedule
Ask: "How often should this run?" (daily, every Monday, weekly, biweekly)
After collecting all choices, help the user create a scheduled task with the filled-in pipeline prompt.

## Pipeline Template (fill in from steps above)
The recurring run should:
1. RESEARCH — search web for NEWS ARTICLES in the last [period] about [category]. Extract company names + what happened + source URL. Present findings before proceeding.
2. DISCOVER — run firmographic_search with the user's filters. If an existing map, also pull get_market_map_data with score/isInCRM/hasOutreachCampaign columns.
3. FILTER — keep companies matching all criteria. Remove duplicates.
4. PRESENT — show table with Company, Domain, Headcount, Location, Score, Funding, Why Source. Ask user to approve/skip each.
5. EXECUTE — for approved: research_company, find_company_contact, generate_campaign with chosen strategy.
