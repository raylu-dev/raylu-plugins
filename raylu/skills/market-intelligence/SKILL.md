---
name: market-intelligence
description: Use when the user wants to analyze a market map — distributions of scale, funding, geography, growth, M&A activity, or web traffic — or wants intelligence the existing columns do not cover (add an AI enrichment column on the fly). Trigger phrases include 'analyze my market map', 'what does the funding landscape look like', and 'which companies are growing fastest'.
---

# Market Intelligence

Raylu market maps contain structured data on hundreds to thousands of companies in a market — firmographic data, funding, growth metrics, scoring, and AI-enriched columns.

**Autonomy rule:** Once you know which market map to use, start proposing analyses immediately. Don't wait to be told what to analyze — suggest the most interesting cuts.

## Part A: Pick or Build a Market Map

1. Call list_market_maps to see what's available
2. If one matches, call get_market_map to see its columns
3. If none fit, use start_market_map / update_market_map to build one

## Part B: Suggest Macro Analyses

Propose analyses based on available columns:

| Analysis | Columns to pull | What to compute |
|----------|----------------|-----------------|
| Scale distribution | headcount or revenue | Histogram of company sizes |
| Funding landscape | lastFundingStage, lastFundingAmount, totalFundingAmount | Distribution by stage, total capital |
| Active investors | allInvestors, allFundingLeads | Top investors, co-investment patterns |
| Geographic concentration | country, state, city | Where companies cluster |
| Growth vs decline | employeesCountChange, monthlyVisitsChange | Fastest growers and decliners |
| M&A activity | status | Acquired vs active companies |
| Web traffic tiers | monthlyVisits, pagesPerVisit | Traffic distribution |

### How to run an analysis

1. Identify columns needed (2-5 max)
2. Call get_market_map_data with those specific columns — NEVER pull all columns
3. Compute statistics yourself from the raw data
4. Present with counts, percentages, top/bottom lists
5. Ask: "What else would you like to explore?"

## Part C: AI Enrich for Non-Column Intelligence

When the user wants intelligence that doesn't exist in current columns:

1. Write an enrichment prompt (direct instruction, specific, with output format)
2. Call create_ai_enrich_column with the prompt
3. Call enrich_market_map to trigger enrichment
4. Wait 1-5 minutes, then pull data via get_market_map_data

### Output types
- enum: Categorizing into buckets (MUST provide options array)
- text: Free-form analysis
- shortText: Brief labels
- number: Quantitative scores
- array: Lists of items

### Good enrichment prompt example
"Classify this company's primary target customer segment. Based on their product, pricing, and marketing, determine whether they primarily sell to SMB (<100 employees), Midmarket (100-1000 employees), or Enterprise (1000+ employees)."

### Bad prompt example
"What kind of customers does this company have?"
