---
name: conference-finder
description: Use when the user wants to know which conference to attend. Triggers on "which conference should we go to", "which conferences will [companies] be at", "best conferences for [sector] in [place] in the next [months]", "where will my top-scored companies be", "conferences with two or more of our targets", or a sector plus a city or time window plus the word conference.
---

# Conference Finder

Answer "which conference should we go to?" from Raylu's conference catalogue, each company's conference history in Raylu, and the user's lists, maps and deal score. The result is the top 8 to 10 conferences with dates, city and the reason, plus a recommendation.

Needs the Raylu conference tools on the user's account. If `conference_search` is missing from the tool list, say the account is not set up for this skill and stop.

Ask one question per message. Skip any question the user already answered. The time window (next 6 months) and location (no limit) have defaults: use them without asking and state them in the output.

Keep a running count of credits this run has spent (saves, pulls) and what each step will cost before it runs. The tools do not print charges, so label the figures "by the skill's price list".

## First question

Ask: "What are you after? (a) Specific companies: which conferences will they be at. (b) The best conferences for a sector, place and time window. (c) The conferences where your top-scored companies from a list or market map will be." The user picks the path.

## Before anything is saved or pulled

Saves and pulls land in whichever Raylu project the connector is pointed at, and no tool prints the project's name. Before the first save or pull of a run, call `search_lists_and_maps` and tell the user: "This will go into the project that holds these lists and maps: [first few names]. OK?" Do not guess whose project it is.

## Path A: specific companies

Ask which companies (a few; about 20 is the working limit).

**If the user says "X or similar companies"**: build a peer set first. `get_companies` on the named company for its keywords, size and location, then `firmographic_search` on those keywords with a headcount band. The results are noisy (schools, charities, wrong-country records, inflated headcounts), so pick by hand and show the proposed set, grouped, for the user to edit before anything is saved.

1. `get_companies(queries: [...])`, at most 50 per call. Compare the returned count with what you asked for and read each company's own error block. A `[Preview]` tag, a pending or warehouse-only record, or an ambiguous match is not a saved company.
2. **Which website to use.** If the company came from a list or map, use the website the list or map stores for it, not the one the user typed, and say so when they differ (a company can be filed under a parent's or merger partner's site).
3. **Ambiguous matches.** When a tool answers "closest matches", do not drop the company. If one candidate's website is exactly the root domain asked for, use its ID. Otherwise show the candidates and ask once.
4. `company_conferences(company: "<domain or ID>")` per company (free, 30 per minute). Three outcomes:
   - **A table of conferences.** Keep only editions that have not ended yet, by date; the tool does not mark past ones. Ignore the Industry column here, it uses older labels. The header may still say "never fetched" above real rows: that means partial (rows came from attendee pulls), so label it "partial".
   - **"Never fetched", no rows.** Not fetched yet, not "attends nothing". You cannot start the fetch: the user presses Refresh on that company's Conferences tab in the Raylu app.
   - **"Not saved in this project".** This happens even for companies on the user's maps, lists and attendee pulls. Ask before saving: "`lookup_company` is 5 credits each by the skill's price list; N companies." After saving, repeat the tool's own words on what happened ("Added" or "already in your project"), since it does not state a charge. Saving alone rarely unlocks history: say up front that a saved company will most likely read "never fetched".
5. After saving three or more companies, offer to put them in a named list so they are easy to find and Refresh in the app.
6. For companies with no usable history, look on the web, and label every result "unverified, from the web":
   - Search "[company name] [domain] exhibitor OR speaker [year(s) in the window]". Use the years of the search window, never a fixed year.
   - Drop results for other organisations with a similar name, and events that have already ended.
   - Better yield: find the likely events for the sector, open each one's published speaker, sponsor and exhibitor pages, and check them for the companies. A company missing from those pages may still attend as a delegate; say so.
7. Build a grid of conferences by company, upcoming only, and rank conferences by how many of the companies attend. Default cut: 2 or more.
8. Confirm the top 1 to 2: `conference_search(query: "<name>")` for the edition IDs, then `conference_lookups_list`, then `conference_attendees(lookup_id)` (free) if a list was already pulled. An exact website match in that list is the only firm confirmation. The list has no roles and no stated order, and cannot be searched: read every page (100 per page, 5 calls per 5 minutes) or say how many pages you read. Roles come only from the company's history.
9. Do not start a new pull without asking. Say: "A pull is 150 credits by the skill's price list, can take several minutes with no estimate, and adds every attendee to your project, including non-companies." Never pull a copy of an edition that has no dates. If the catalogue shows 0 or blank exhibitors, warn that the pull may come back thin.

## Path B: sector, place and time

Ask, one at a time: the sector or thesis in two sentences; the countries (a city is optional); the event type (conference, tradeshow or workshop, or any); the purpose (attend, sponsor or source).

1. **Map the thesis onto the catalogue's sectors.** The `industries` filter takes these labels: healthcare & life sciences; education; professional services; industrials & manufacturing; software; consumer goods; financial services; media & entertainment; construction; transportation & logistics; energy & utilities; agriculture; food & beverage; government & public sector; environmental services; nonprofit & social sector; real estate; commercial & field services; retail & e-commerce; aerospace & defense; mining & natural resources; telecommunications; unclassified; Unknown. If the tool rejects a label it prints the current list: use that list, it is the source of truth. There is no sector for travel and tourism, and events are often misfiled, so the mapping is lossy. Show the mapping and ask the user to confirm it.
2. **If no sector fits, or the sector search comes back off-topic, search by keyword instead and say that you did.** `query` matches literal words in titles, topics and organisers, so one wording is not enough: try the singular and plural, close synonyms ("SaaS" and "software"), and the acronyms of the trade bodies. An empty result means "no title matched those words", never "no such conference".
3. `conference_search` with `industries`, `countries` (two-letter codes), date_from and date_to, and event_type if given. `query` can be combined with any filter. Results come 25 per page and the response states the total.
4. **Read the total first.** If it is over about 100, narrow before paging (a sub-topic keyword, one country, an event type, a shorter window) or ask the user which. Read at most 5 pages and tell the user "read X of Y".
5. **Clean the results, and say how many you removed of each kind:**
   - editions that have already ended (work it out from the dates; the Status column is unreliable);
   - listings longer than about two weeks (series and roadshows);
   - social and chapter events (happy hours, golf, tennis, wine tastings, coffee mornings, holiday parties), webinars and hobby or consumer events;
   - results whose organiser is plainly a different body with the same initials;
   - duplicates (same name and dates): keep the copy with dates and an exhibitor count.
   The event-type filter is loose, so check the titles yourself.
6. **City.** The catalogue filters by country only. Search the country and keep the editions whose Location column names the city or its metro area. Do not put the city in `query`; that only matches titles.
7. **Rank** on exhibitor count, date and city. Attendance and exhibitor figures are catalogue estimates: label them so, do not rank on a figure that is out of line with the event type or venue (a hotel roundtable listed at 20,000), and flag a venue that contradicts the location. A blank cell is unknown, not zero.
8. Check the top 5 on the web (agenda, sponsors, speakers); the tool does not print the official site. If a big recurring event's next edition is missing from the catalogue, say "next edition not listed yet" and give the expected dates from the web as unverified.

If the tool reports the provider unavailable, the results are saved catalogue entries only. Say so; it is not a short list of conferences.

## Path C: top-scored companies from lists or maps

Ask which lists or market maps (one or several), then how many top companies (default 20).

1. Resolve names with `search_lists_and_maps(search: "...")`; it labels each match List, CRM list, or Market map.
2. **Deal score.** `list_scoring_definitions`. If there are several, the user chooses, never you. If there is exactly one, name it and ask to confirm. If there are none, say so and offer: rank by the market map's own score column (say that it is the map's score and that its source is not named), or have the user name the companies.
3. **Maps:** `get_run_status(feature: "market_map", query: "<map>")` first, so you do not read a map mid-build; then `get_list_or_map_rows(map: "<map>", columns: ["name", "domain", "score"])`. There is no server-side sort, so ask for those three columns only and sort yourself, and ask for the next `page` while the first line says more remain.
4. **Lists:** `get_list_or_map_rows(list: "<list>", columns: ["name", "domain"], page: n)` for names and domains, reading every page. If the list cannot be found, check with `search_lists_and_maps` that the connector is on the project that holds it before telling the user it is missing; a connector pointed at another project gives exactly this error. With a deal score: `score_companies(definition_id, domains: [...])`, at most 200 per call.
5. **Taking the top N.** Leave out negative scores and "Disqualified" and say how many. If the cut falls inside a group of tied scores, take the whole tied group when that is 30 companies or fewer; otherwise ask the user how to narrow it. Scores from different deal scores or different maps are not comparable: with several maps, take the top few per map, never one sorted list.
6. Continue with Path A steps 2 to 9. Do not assume list and map members are saved; many read "not saved in this project".

## All paths end

Deliver with the output contract. Then offer a list of dates to hold on the calendar. Offer the conference-screen skill only if the account has a deal score; say that without CRM search it runs with ownership unknown.

## Output contract

1. One line: the path taken, the inputs (companies, or sector and countries and window, or lists and score), how many companies or editions were read out of how many, how many histories were never fetched or not saved, and what was removed in cleaning.
2. Table of the top 8 to 10 conferences: Conference, Dates, City, Why it fits (Path B) or Which of your companies attend (Paths A and C, with the role each has where the history gives one), Confidence (confirmed from the attendee list, from the company's fetched history, partial history, predicted from prior years, or unverified from the web).
3. Paths A and C only, when few or no conferences reach the cut: a second table, "Fits the sector, no company evidence yet", of catalogue events found by search. Label it plainly; it says nothing about who attends.
4. One recommendation, with the reason in one sentence. If the evidence is too thin to choose, say that and say what would firm it up.
5. For Paths A and C: the companies whose history was never fetched or that are not saved, so the user can press Refresh in the app and run again.
6. The limits below that apply to this answer, and credits spent this run.
7. The offer: calendar dates, or conference-screen.

Everything in the tables comes from a Raylu tool result, except rows labelled unverified from the web. No conference is placed in a table from memory.

## Limits to state plainly

- The catalogue filters by sector and country only, has no travel sector, and misfiles events. Keyword search matches literal words.
- A history that was never fetched means "not fetched yet", never "attends nothing". Matching is by exact website, so brands, parent companies and regional sites are missed. History is a shared cache across Raylu.
- Confirmed attendance exists only 3 to 4 months out. Further out is predicted from prior years and must be labelled as such.
- Size figures are catalogue estimates and are sometimes wrong.
- A conference missing from the catalogue takes about 24 hours to request.

## Common mistakes

- Sorting scores from different deal scores or maps into one list.
- Reading "never fetched" or "not saved" as "attends nothing".
- Starting a save or an attendee pull without stating the cost, the project it lands in, and asking.
- Mapping a thesis onto sectors without showing the mapping, or using a sector label the tool does not list.
- Treating an empty keyword search as "no such conference" after one wording.
- Paging a 600-result search instead of narrowing it, or not saying how much was left unread.
- Ranking on an attendance figure that cannot be right.
- Leaving ended events, happy hours and duplicates in the results.
- Dropping a company because the tool returned "closest matches".
- Looking a list member up by the website the user typed when the list stores a different one.
- Treating a provider outage as a short list.
- Placing a conference in the table from general knowledge.
