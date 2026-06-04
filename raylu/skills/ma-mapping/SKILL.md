---
name: ma-mapping
description: Use when the user wants to build an M&A map anchored on a specific acquisition target — choosing a lens (inline consolidation versus adjacencies), defining acquirability criteria, and producing a scored, AI-enriched table of candidates. Trigger phrases include 'build an M&A map for a company', 'find acquisition targets like X', and 'who could this acquirer buy'.
---

# M&A Mapping

Build an M&A map anchored on a specific target. Walk through these steps ONE AT A TIME. Do not skip steps.

## Step A: Identify the Target
Ask: "Which company is the M&A target? Give me a name or domain."
Call get_company to pull the full profile. Present the firmographic data. Confirm with the user.

## Step B: Choose the Lens
Ask: "What kind of targets are you looking for?"
1. Inline consolidation — companies that do the same thing as the target. Direct competitors.
2. Adjacencies — companies in neighboring spaces that complement the target.
Let the user pick one or both.

## Step C: Define Acquirable Criteria
Ask these questions one at a time:
1. "What headcount range? Should they be smaller than [target]?"
2. "Same customer size? (SMB / Midmarket / Enterprise)"
3. "Same go-to-market motion? (Sales-led, product-led, channel)"
4. "Same ideal customer profile?"
5. "Should the team be in the same region as [target]?"
6. "Funding stage preference?"
7. "Revenue range?"

From answers, derive firmographic filters + AI enrich columns for non-structural criteria.

## Step D: Build the Market Map
Call create_ma_map with:
- category, description, subcategories, regions derived from the target's space
- inScope/outOfScope based on acquirability criteria
- firmographics from Step C as natural language (e.g., "less than 250 employees", "Series A or earlier")
- enrichColumns for non-structural criteria (e.g., "Target Customer Segment", "GTM Motion", "Product Overlap with [Target]")

This creates the map, populates companies, scores them, creates AI columns, and runs enrichment — all in one operation. Use get_market_map_data to check results when enrichment completes.

## Step E: Filter and Present Results
Pull get_market_map_data with all relevant columns including AI enriched ones.
Filter to companies matching ALL acquirability criteria.
Present in a table:
| Company | Domain | Headcount | Location | Funding | Customer Segment | GTM | Product Overlap | Acquirability Notes |

The Acquirability Notes column should explain why the company is a fit and any red flags.
Offer: dig deeper on specific companies, start outreach, or save for reference.
