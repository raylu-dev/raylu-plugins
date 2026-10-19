---
name: travel-meetings
description: Use when a Raylu user is travelling to a city and wants to know which companies or CEOs from their own pipeline are there so they can book meetings. Triggers on "I'm in Dallas next month, who should I meet", "who from my CRM is in Chicago", "fill my trip with meetings", "anyone near Boston I should see", or any city plus a request to find companies or people to visit.
---

# Travel Meetings

Answer "who should I meet while I'm in [city]?" from the user's own companies in Raylu, using Raylu's location data and never the CRM's. Companies come from a Raylu CRM list, market map or list the user already has. Location comes from Raylu: company HQ, or where the CEO actually lives (from LinkedIn), which is often a different city.

## Data source rule (read first, never break)

- **Companies come only from Raylu.** The source is always a Raylu CRM list, market map or list, read with `get_list_or_map_rows`. Never call `crm_search` in this skill, not for names, not for domains, not for owners, not for locations. When the user says "my CRM", "Salesforce", "Affinity", "Attio", "DealCloud", "my callback list", "my covered list", "my touched companies" or anything like it, they mean a Raylu CRM list, not a CRM lookup.
- **Location comes only from Raylu.** The `ceoLocation`, `ceoName`, `city` and `state` columns from `get_list_or_map_rows`; the Location line from `get_companies`; person locations from `company_people_search`. Nothing else.
- **Never use a CRM address field** (BillingCity, BillingPostalCode, BillingStreet, Shipping*, or any CRM city, state or ZIP) to place, match, include, exclude or narrow a company. A CRM address filter drops every company whose CEO lives in the trip city but whose CRM address is elsewhere or blank, which is exactly the case this skill exists to catch. If you are about to filter anything by a CRM address, stop.
- Lists and market maps return the same location columns. Do not look up HQ or CEO company by company when the list already carries `city`, `state`, `ceoName` and `ceoLocation`.

## When to use

- The user names a city and wants meetings from companies they already track.
- Not for finding new companies in a city. That is a market map. If they want new targets, say so and offer to start one.

## Flow

Ask one question per message. Skip any the user already answered. If everything was answered in the first message, send one short restatement of the assumptions and start.

1. **City and dates.** Usually given. A region name (Bay Area, DFW, Tri-State) is not a city: use its principal city as the centre, say so, and invite a correction. Do not ask for dates up front; if you need a month for naming, infer it ("the 20th" means the next 20th) and say "roughly". Dates are asked only at step 9, and only if outreach is generated.
2. **Where should I pull from?** Build the menu below, then ask. Skip it only when the user already named a specific list or map; then resolve that name (see "Resolving a named source").

   **Building the menu.**
   - CRM lists: call `search_lists_and_maps` with `search` set to "CRM", then "Touched", then "Team". Keep only matches labelled "CRM list". The same name can appear many times (one copy per user); treat copies with the same name as one entry. Search never shows a CRM list's company count, so read one copy of each name with `get_list_or_map_rows(list: <ID>, columns: ["name"], limit: 1)` and take the total from its first line. Drop any CRM list that reads 0 companies.
   - Market maps and lists: one `search_lists_and_maps` call with no `search`, `limit: 10`. It returns the whole account, not only the user's own, so show the first ten of each kind as returned and give the rest as a count.

   **The message**, in this shape:

   > We can pull from your CRM list:
   > - **[CRM list name]** ([N] companies): [description]
   > - …
   >
   > We can also pull from market maps: [name], [name], [name] … (and [N] more; name any of them).
   >
   > Or lists: [name], [name], [name] … (and [N] more; name any of them).

   Show one bullet per CRM list that has companies. When exactly one CRM list has companies, still show it as a bullet. Describe each CRM list by its name, with exactly these words:
   - **My CRM**: every account in your firm's CRM that has an owner, across the whole firm, not only yours.
   - **All CRM Companies**: every company in your firm's CRM.
   - **CRM Companies**: the accounts you own in the CRM.
   - **My Touched Companies**: accounts where you logged a touch in the CRM in the last 5 years (at TA: a Note, Pass Lead or Interest activity; calls and visits logged as meetings do not count).
   - **Team CRM Companies**: accounts owned by the people on your team, as set up by Raylu.
   - Any other name on a CRM list: "a list synced from your CRM". Do not guess what it contains.

   **When every CRM list reads 0 companies**, the connected user has no CRM identity in this project (their email does not match a CRM user). Say so in one line, leave the CRM section out of the menu, and offer the market maps and lists.

   **When the user's answer is just "my CRM" (or Salesforce, Affinity, callback, covered…) without naming one**, use the largest of CRM Companies, My Touched Companies and Team CRM Companies, and say which you used. Never default to My CRM or All CRM Companies; those are firm-wide. Use them only when the user names them.

   **Resolving a named source.** Resolve names with `search_lists_and_maps(search: <the distinctive word from the user>)`; it labels each match List, CRM list, or Market map. Exact match wins; mention near-misses in one clause. No exact match and more than one near-miss: ask which, and say which one you would guess. Several regular lists or maps with one name: never pick by name; ask for the link from Lists in the app and use the ID in it. Several CRM lists with one name: read each one's count as above; equal counts are the same list for this user, so use any; different counts (Team lists can differ per team): ask for the link. Read the company count from the tool's own count line, never from a number inside the list's description.

   When the source is a market map or a non-CRM list whose rows carry `isInCRM`, ask as part of this step: "Only companies already in your CRM, or everyone on the map?"
3. **How wide a net?** "How far are you willing to drive? Any number works; people usually say somewhere between 25 and 100 miles." Use the figure and the unit they give, miles or kilometres, throughout.
4. **Which location?** Offer both, recommend one:
   - "CEO location: where the CEO actually lives, from LinkedIn. I recommend this if you want to meet the CEO."
   - "Company HQ: I recommend this if any executive is fine."
   - If they cannot choose, treat it as CEO location. Both come back together anyway; the choice only sets which column leads.
5. **Metro set.** Now that you have the radius, build the set of place names inside it from your own geography: the city, its boroughs or districts, its suburbs, neighbouring towns. Measure straight-line, centre to centre; the distance band is the one judgment in the table that comes from you, not a tool, so state the set in the output (one line per band, or counties and regions with named exceptions for a large metro) so the user can correct it. "In city" means the city and its boroughs or districts (the five boroughs for New York, the London boroughs for London). Match on city plus region together: the state for US trips; for other countries, any sub-national label of that country (home nation, county, borough, district, postcode fragment), since the data's region column holds whichever it has. A row whose city slot holds a street, a county, or a region name has an unknown city and a known region. A city name shared across countries (London, Cambridge, Perth, Richmond, Windsor, Kingston) never matches on city alone; a blank region with a shared name is unknown. A city string that begins with the city name matches ("London Ec V Nx", "London Sw Bz" are London). A region value that is a borough of the trip city ("Lambeth", "Westminster", "Sutton") counts as in city; say so. A returned city you cannot place ("Glenview, New York") goes to "Checked, not in range" with a flag and still counts as checked. For up to five rows with a blank region and a name that could be inside the radius, spend one `get_companies` call to resolve them. A list name and a resolved company name that disagree, or a domain the tool remapped, get one clause; that comes from the tool, not from memory.
6. **Run** the reading path below. Say what you are doing and roughly how long.
7. **Deliver** with the output contract.
8. **Contacts?** After the list: "Want email and phone for these?" (or "for [Company]?" when there is one match). On a yes, use any Contacts row already returned by `get_companies` that is not the user themself; that is a vendor contact record and already counts as a contact, though a row with no title does not identify who the CEO is. Run `find_company_contact` only for companies with no such row. When the contact record names a CEO who differs from the CEO the list or people search showed, the list's record is probably stale: the list and people search come from scraped LinkedIn titles, while contact records come from vendors that track the current officeholder. Say so. If the company matched on CEO location, re-run `company_people_search(company)` for the new name. In range: keep the CEO match. Out of range or not found: drop the CEO leg only; the company stays if its HQ is in range, otherwise it moves to "Checked, not in range" with the reason. Then ask which person the outreach should go to; if the user says "the CEO", use the contact record's name and say so.
9. **Actions.** Offer, do not do: save as a list named "[City] trip [Month Year]"; generate campaigns with a "City Visit" cadence. Only when the user says yes, call `list_strategies(includeTemplates: true)`. If the user already owns a copy such as "City Visit (Copy)", use it; otherwise copy the template with `copy_cadence` first, since templates cannot be generated from directly. Ask for the exact trip days if you do not have them; "the week of Oct 20" is not enough. Always pass `contactEmail` (the person chosen at step 8) to `generate_campaign`; without it the campaign goes to the company's primary contact, which in some projects is the user's own CRM record. `generate_campaign` cannot fill the template's custom fields (trip city, trip dates), so each draft comes back in "needs attention". Clear it yourself: confirm generation with `get_run_status(feature: "company", query: <domain>, type: "outreach")`, find the campaign ID with an unfiltered `list_campaigns` (the name search misses new campaigns), call `get_campaign(campaignId)`, and for every email step whose subject or body still contains `[Trip city]` or `[Trip dates]`, call `update_email(emailId, subject, bodyText)` with the city and dates written in. Leave `[Your Name]`; it is a system variable filled at send. Read the campaign back and confirm no custom-field brackets remain in any email step. The LinkedIn step's note cannot be read or edited through the connector and its prompt also names the trip fields, so tell the user to check that one step in the app. The campaign keeps showing "needs attention" until someone opens it in the app; say so. Generate as drafts, never auto-start. If the user did not give dates, leave the drafts as they are and tell them the two values to type in the Campaigns tab. There is no email notification in Raylu; do not offer one.

## Reading path (CRM lists, market maps and lists alike)

1. `get_list_or_map_rows(<list or map>: <ID>, columns: ["name","website","primaryDomain","city","state","ceoName","ceoLocation","isInCRM"])`. Pass `list` for a CRM list or list, `map` for a market map. Pages hold up to 500 rows: while the count line reads "page X of Y" with X below Y, ask for the next `page`. Read every page; there is no size gate, since one call covers 500 companies. If the source is still building, the reply says so above the rows; say the list may still grow. If the row count is a few short of the listed count, report the gap in the coverage line; do not retry.
2. HQ is `city` + `state`. The `state` column holds whatever sub-national label the data has: a US state, a country, a county, or a borough.
3. CEO is `ceoName` + `ceoLocation`. CEO names arrive as scraped LinkedIn strings with credentials or taglines ("FRCS Dr. Jane Doe MDCM", "James Weir - ask me about NDA automation"); keep the honorific and name, drop the rest.
4. `isInCRM` comes back as "Yes, in CRM" or "No, not in CRM". When the user chose the in-CRM filter, rows flagged No are neither matches nor "not in range": give their count in one clause and leave them out of every table. (Every row of a CRM list is in the CRM; skip this for CRM lists.)
5. **People-search fallback**, only for rows the columns cannot settle: in-range companies (by HQ) with a blank `ceoName`, and rows with a blank or country-only `ceoLocation` whose HQ is in the trip's region or an adjacent one. Call `company_people_search(company, seniority: "Founder")`; if no row has a CEO title, including when zero people come back, call again with `seniority: "C-Level"`. Those two values are the ones the data uses; "C-Suite" is a department, not a seniority. Cap: 40 companies. Above 40, take the in-range ones first, then list the companies skipped. People-search locations come back as "City, Country" with no state; an ambiguous city name ("Lewisville, United States") stays uncounted and is flagged. Zero people in both calls means unknown. People returned but none titled CEO also means unknown: write "unknown (no CEO title returned)" in the CEO column, and any of those executives who live in range get an "other exec" row.
6. When the location choice is HQ and more than ten companies are in range with no CEO name, the table has no names for those; say so and offer a capped CEO pull: "I can look up who runs up to 40 of these. Which ones, or shall I take the first 40?"

Reading `ceoLocation`: a city string matches on city and region; any string that names the trip's metro ("San Francisco Bay Area", "Greater Seattle Area", "Dallas-Fort Worth Metroplex", "Austin, Texas Metropolitan Area", "Greater London", "London Area, United Kingdom", "Greater Paris Metropolitan Region", "Zürich Metropolitan Area", "Metro Vancouver") counts as matched with band "metro, city not given"; a country-only or blank string is unknown, never "not there"; when the country named is not the trip's country, write "unknown, different country".

## Output contract

1. One line: source name and kind (CRM list, market map or list), radius and centre city, location field, how many companies were checked, and the words "location only, not ranked". Then the place set used for the bands.
2. Table of matches only, sorted by distance band, with columns: Company, HQ city, CEO name, CEO city, Matched on (HQ / CEO / both / other exec), Distance band. Bands use the user's unit and radius: "in city", "within half of it" ("within 15 mi" for 30 miles, "within 40 km" for 80 km), "within [radius]", and "metro, city not given". When HQ and CEO land in different bands, the band column shows the one for the location field the user chose, and the other appears in its own column's text. A co-founder, chair, or other executive who lives in range gets their own row with "other exec" and their title, since the user may still want that meeting. When CEO data was not pulled, the CEO columns say "not pulled". More than 40 matches: give counts per band and a compact name list per band instead of one row each.
3. A separate short table titled "CEO nearby, HQ elsewhere" when any row matched on CEO only. These are the ones a CRM address would have missed.
4. "Just outside the radius", with the outer bound stated ("40 to 60 mi"): companies within roughly half the radius beyond it, one line each, so a boundary call is the user's.
5. "Checked, not in range": the remaining companies with their HQ city, so the user can see the whole source was covered. Up to 50 companies: list them all. More than 50: list only those in the trip's region (or with a CEO in the trip's region) and any whose name suggests the trip's country but whose data places them elsewhere, and give the count of the rest in one clause.
6. One line on coverage: rows read versus the source's count, how many had no HQ, how many had no CEO location, how many the people-search fallback resolved, and any skipped by the 40 cap.
7. The contacts question, then the actions offer.

Everything in the table comes from a Raylu tool result, except the distance band, which is stated as your estimate. No company is placed in a city from memory or from the CRM, and no "they have a Dallas office" from general knowledge. If a fact did not come back from a Raylu tool, it is not in the table.

## Common mistakes

- Calling `crm_search` for anything. The CRM list in Raylu is the source; the CRM itself is never queried.
- Narrowing, matching or excluding companies on a CRM address field (billing city, ZIP, street) instead of Raylu's `city`, `state` and `ceoLocation` columns.
- Reading only `name` and `domain` from a list and then looking up HQ and CEO company by company. The list already has those columns.
- Defaulting "my CRM" to the firm-wide My CRM or All CRM Companies list.
- Describing a CRM list from its name alone when the name is not one of the five above.
- Bundling the source, radius and location questions into one message.
- Asking about sector, goal, or check size. Not this skill's job.
- Resolving a list name from the first ten rows only.
- Matching "London" without checking the country.
- Ranking or annotating companies from outside knowledge. The table is location data only.
- Treating a blank CEO location or an empty people search as "not in the city".
- Putting the Contacts block from `get_companies` in the CEO column.
- Offering an email digest. There is no tool for it.
