---
name: conference-finder
description: Use when the user wants to know which conference to attend. Triggers on "which conference should we go to", "which conferences will [companies] be at", "best conferences for [sector] in [place] in the next [months]", "where will my top-scored companies be", "conferences with two or more of our targets", or a sector plus a city or time window plus the word conference.
---

# Conference Finder

Answer "which conference should we go to?" from Raylu's conference catalogue, each company's conference history in Raylu, and the user's lists, maps and deal score. The result is the top 8 to 10 conferences with dates, city and the reason, plus a recommendation.

Requires the single `conferences` tool. If it is unavailable, report that this account lacks conference access. Calls use `{"request":{"operation":"...", ...}}`. Read the live tool schema for each operation.

Ask one question per message. Skip any question the user already answered. The time window (next 6 months) and location (no limit) have defaults: use them without asking and state them in the output.

Reading conference data does not start provider work or save a company. Refreshing, submitting or starting a lookup can start provider work; report that effect and use stable request IDs. Do not invent a price or completion estimate.

## First question

Ask: "What are you after? (a) Specific companies: which conferences will they be at. (b) The best conferences for a sector, place and time window. (c) The conferences where your top-scored companies from a list or market map will be." The user picks the path.

## Before anything is saved or pulled

Confirm the project only if the intended workspace is ambiguous. A read does not require saving the company. Execute explicitly requested retrievals and list exports without an extra permission question.

## Path A: specific companies

Ask which companies (a few; about 20 is the working limit).

**If the user says "X or similar companies"**: build a peer set first. `get_companies` on the named company for its keywords, size and location, then `firmographic_search` on those keywords with a headcount band. The results are noisy (schools, charities, wrong-country records, inflated headcounts), so pick by hand and show the proposed set, grouped, for the user to edit before anything is saved.

1. `get_companies(queries: [...])`, at most 50 per call. Compare the returned count with what you asked for and read each company's own error block. A `[Preview]` tag, a pending or warehouse-only record, or an ambiguous match is not a saved company.
2. **Which website to use.** If the company came from a list or map, use the website the list or map stores for it, not the one the user typed, and say so when they differ (a company can be filed under a parent's or merger partner's site).
3. **Ambiguous matches.** When a tool answers "closest matches", do not drop the company. If one candidate's website is exactly the root domain asked for, use its ID. Otherwise show the candidates and ask once.
4. Use operation `company_attendance` with the company domain or canonical ID. Keep source and freshness information. A never-fetched or partial history does not mean the company attends nothing. Distinguish provider failure from genuinely empty data.
5. For requested discovery or refresh, use operation `refresh_company` with the same company and a stable `request_id`. It can start provider work without saving a warehouse company into the project. Read the history again to check published results.
6. Build a grid of upcoming conferences by company, using dates rather than an assumption that historical attendance guarantees future attendance. Rank by how many target companies have evidence. Label predictions separately from recorded participation.
7. Confirm promising editions with operation `search`, then `saved` and `attendees`. Reuse exact event and edition IDs. Roster rows include company roles and can be searched and filtered. Preserve `snapshot_revision` and `visit_token` across pages and state coverage.
8. A requested new attendee lookup uses `attendees` with both event and edition IDs, plus a UUID `request_id`. Reuse that UUID for retries of the same start. Existing `lookup_id` reads do not start a crawl. Partial rows can be useful while retrieval continues.
9. If Raylu has no evidence, say so. External research is optional supplementary evidence and must carry its source and date. Never label an external inference as confirmed Raylu attendance.

## Path B: sector, place and time

Ask, one at a time: the sector or thesis in two sentences; the countries (a city is optional); the event type (conference, tradeshow or workshop, or any); the purpose (attend, sponsor or source).

1. Use operation `sectors` for current industry labels and `locations` for stable location IDs. Avoid hardcoded vocabularies. Show a lossy thesis-to-sector mapping when it affects the result.
2. Use operation `search` with query, industries, countries, location IDs, dates, event types and other supported filters. Query can match names, websites, topics, audiences and locations. Filters work without a query and combine with it.
3. Read total count and page metadata. Use the requested sort and `page_size`, up to 100. Preserve zero, false and unknown values distinctly.
4. **Read the total first.** If it is over about 100, narrow before paging (a sub-topic keyword, one country, an event type, a shorter window) or ask the user which. Read at most 5 pages and tell the user "read X of Y".
5. **Clean the results, and say how many you removed of each kind:**
   - editions that have already ended (work it out from the dates; the Status column is unreliable);
   - listings longer than about two weeks (series and roadshows);
   - social and chapter events (happy hours, golf, tennis, wine tastings, coffee mornings, holiday parties), webinars and hobby or consumer events;
   - results whose organiser is plainly a different body with the same initials;
   - duplicates (same name and dates): keep the copy with dates and an exhibitor count.
   The event-type filter is loose, so check the titles yourself.
6. **City.** Use returned location IDs or supported city filters. Check the returned venue and city against the intended location; a text match alone is not a location guarantee.
7. **Rank** on exhibitor count, date and city. Attendance and exhibitor figures are catalogue estimates: label them so, do not rank on a figure that is out of line with the event type or venue (a hotel roundtable listed at 20,000), and flag a venue that contradicts the location. A blank cell is unknown, not zero.
8. Use the returned official and source URLs to check the top 5 when external verification is useful. If a big recurring event's next edition is missing from the catalogue, say "next edition not listed yet" and give the expected dates from the web as unverified.

If the tool reports the provider unavailable, the results are saved catalogue entries only. Say so; it is not a short list of conferences.

## Path C: top-scored companies from lists or maps

Ask which lists or market maps (one or several), then how many top companies (default 20).

1. Resolve names with `search_lists_and_maps(search: "...")`; it labels each match List, CRM list, or Market map.
2. **Deal score.** `list_scoring_definitions`. If there are several, the user chooses, never you. If there is exactly one, name it and ask to confirm. If there are none, say so and offer: rank by the market map's own score column (say that it is the map's score and that its source is not named), or have the user name the companies.
3. **Maps:** `get_list_or_map_rows(map: "<map>", columns: ["name", "domain", "score"])`. If the map is still building, the reply says so above the rows; tell the user the scores are not final. There is no server-side sort, so ask for those three columns only and sort yourself. Pages hold up to 500 rows: while the count line reads "page X of Y" with X below Y, ask for the next `page`.
4. **Lists:** `get_list_or_map_rows(list: "<list>", columns: ["name", "domain"], page: n)` for names and domains, reading every page. If the list cannot be found, check with `search_lists_and_maps` that the connector is on the project that holds it before telling the user it is missing; a connector pointed at another project gives exactly this error. With a deal score: `score_companies(definition_id, domains: [...])`, at most 200 per call.
5. **Taking the top N.** Leave out negative scores and "Disqualified" and say how many. If the cut falls inside a group of tied scores, take the whole tied group when that is 30 companies or fewer; otherwise ask the user how to narrow it. Scores from different deal scores or different maps are not comparable: with several maps, take the top few per map, never one sorted list.
6. Continue with Path A steps 2 to 9. Company attendance reads and refreshes do not require a separate save.

## All paths end

Deliver with the output contract. Then offer a list of dates to hold on the calendar. Offer the conference-screen skill only if the account has a deal score; say that without CRM search it runs with ownership unknown.

## Output contract

1. One line: the path taken, the inputs (companies, or sector and countries and window, or lists and score), how many companies or editions were read out of how many, how many histories were never fetched or not saved, and what was removed in cleaning.
2. Table of the top 8 to 10 conferences: Conference, Dates, City, Why it fits (Path B) or Which of your companies attend (Paths A and C, with the role each has where the history gives one), Confidence (confirmed from the attendee list, from the company's fetched history, partial history, predicted from prior years, or unverified from the web).
3. Paths A and C only, when few or no conferences reach the cut: a second table, "Fits the sector, no company evidence yet", of catalogue events found by search. Label it plainly; it says nothing about who attends.
4. One recommendation, with the reason in one sentence. If the evidence is too thin to choose, say that and say what would firm it up.
5. For Paths A and C: the companies whose history was never fetched or that are not saved, and whether a requested refresh is still running or needs retry.
6. The limits below that apply to this answer, and provider actions started this run.
7. The offer: calendar dates, or conference-screen.

Everything in the tables comes from a Raylu tool result, except rows labelled unverified from the web. No conference is placed in a table from memory.

## Limits to state plainly

- Preserve source, event date and observation date. Historical participation is not a promise of future attendance.
- Never-fetched, partial, failed and empty histories are different states. Company matching can miss aliases or regional sites.
- Size figures may be estimates. A missing count is unknown rather than zero.
- Use operation `submit` with a source URL and available metadata when the user asks to add an unlisted conference. Do not promise a fixed turnaround time.
- Use `refresh` for a saved lookup, `mark_seen` for a reviewed snapshot and `delete` only for a saved lookup the user explicitly requested removing. Deletion retains shared attendance data.

## Common mistakes

- Sorting scores from different deal scores or maps into one list.
- Reading "never fetched" or "not saved" as "attends nothing".
- Starting unrequested provider work, or silently saving a company just to read attendance.
- Mapping a thesis onto sectors without showing the mapping, or using a sector label the tool does not list.
- Treating an empty keyword search as "no such conference" after one wording.
- Paging a 600-result search instead of narrowing it, or not saying how much was left unread.
- Ranking on an attendance figure that cannot be right.
- Leaving ended events, happy hours and duplicates in the results.
- Dropping a company because the tool returned "closest matches".
- Looking a list member up by the website the user typed when the list stores a different one.
- Treating a provider outage as a short list.
- Placing a conference in the table from general knowledge.
