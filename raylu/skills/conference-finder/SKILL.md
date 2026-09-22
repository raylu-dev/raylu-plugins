---
name: conference-finder
description: Use when the user wants to know which conference to attend. Triggers on "which conference should we go to", "which conferences will [companies] be at", "best conferences for [sector] in [place] in the next [months]", "where will my top-scored companies be", "conferences with two or more of our targets", or a sector plus a city or time window plus the word conference.
---

# Conference Finder

Answer "which conference should we go to?" from Raylu's conference catalogue, each company's conference history in Raylu, and the user's lists, maps and deal score. The result is the top 8 to 10 conferences with dates, city and the reason, plus a recommendation.

Needs the Raylu conference tools on the user's account. If `conference_search` is missing from the tool list, say the account is not set up for this skill and stop.

Ask one question per message. Skip any question the user already answered.

## First question

Ask: "What are you after? (a) Specific companies: which conferences will they be at. (b) The best conferences for a sector, place and time window. (c) The conferences where your top-scored companies from a list or market map will be." The user picks the path.

## Path A: specific companies

Ask which companies (a few; about 20 is the working limit), the time window (default the next 6 months), and any location limits.

1. `get_companies(queries: [...])`, at most 50 per call. Compare the returned count with what you asked for and read each company's own error block; a pending, warehouse-only or ambiguous company is not a saved one. For a company not saved in Raylu, ask before adding it, then `lookup_company(query: "<domain>")` (5 credits each); `company_conferences` only works on saved companies and never saves one itself.
2. `company_conferences(company: "<domain>")` per company (free, 30 per minute): conference, dates, location, industry, role (exhibitor, sponsor, speaker, visitor), official URL. The header says whether the history was fetched and when, or "never", or that a fetch is in progress (check again shortly). Never means "not fetched yet", not "attends nothing". You cannot start the fetch from here: tell the user to press Refresh on that company's Conferences tab in the Raylu app, then run again.
3. For a "never" company, search the web for "[company] 2026 exhibitor OR speaker" and label every result "unverified, from the web".
4. Build a grid of conferences by company, upcoming only, and rank conferences by how many of the companies attend. Default cut: 2 or more.
5. Confirm the top 1 to 2: `conference_search(query: "<name>")` for the edition ids, then `conference_lookups_list`, then `conference_attendees(lookup_id)` (free) if a list was already pulled. An exact domain match in that list is the only firm confirmation. Do not start a new pull here without asking; it costs 150 credits.

## Path B: sector, place and time

Ask, one at a time: the sector or thesis in two sentences; the countries (a city is optional); the time window (default the next 6 months); the event type (conference, tradeshow or workshop, or any); the purpose (attend, sponsor or source).

1. Map the thesis onto 1 to 4 of the catalogue's 30 fixed industries: Apparel & Clothing; Auto & Automotive; Building & Construction; Electric & Electronics; Agriculture & Forestry; Arts & Crafts; Medical & Pharma; Packing & Packaging; Industrial Engineering; Telecommunication; Travel & Tourism; Science & Research; Business Services; Education & Training; Miscellaneous; Logistics & Transportation; Environment & Waste; Fashion & Beauty; Food & Beverages; Baby, Kids & Maternity; Wellness, Health & Fitness; Power & Energy; Banking & Finance; Security & Defense; IT & Technology; Animals & Pets; Home & Office; Hospitality; Entertainment & Media; Music & Entertainment. There is no bucket for SaaS, AI or cyber, and fintech, insurance and private equity all fall under Banking & Finance, so the mapping is lossy. Show the mapping and ask the user to confirm it.
2. `conference_search` in filter mode, with no `query`: `industries`, `countries` (two-letter codes), date_from and date_to, event_type if given, and `limit: 100` so a page is not cut short. Industries times countries must stay at 12 or fewer. Page until the tool says no further page exists, or 5 pages, whichever first; the tool never reports a total. Remove duplicate editions. If the user named a city, keep the editions whose location names it; the catalogue filters by country only.
3. Rank on exhibitor count, footfall, date and city. Check the top 5 on the web (agenda, sponsors, speakers); the tool does not print the official site.

If the tool reports the provider unavailable, the results are saved catalogue entries only. Say so; it is not a short list of conferences.

## Path C: top-scored companies from lists or maps

Ask which lists or market maps (one or several), which deal score (`list_scoring_definitions`; the user chooses, never you), how many top companies (default 20), and the time window.

1. Resolve names with `list_market_maps(search: "...")` or `list_lists(search: "...")`.
2. Maps: `get_run_status(feature: "market_map", query: "<map>")` first, so you do not read a map mid-build; then `get_market_map_data(query: "<map>", columns: ["name", "domain", "score"])`. There is no server-side sort and no paging, and the call times out at 15 seconds, so ask for those three columns only and sort yourself. A map's score column is the map's own deal score; say which definition it comes from, and if it is not the one the user chose, score the domains with `score_companies` instead. Lists: `get_list(query: "<list>", limit: 50, page: n)` for names and domains, then `score_companies(definition_id, domains: [...])`, at most 200 per call. Merge by domain and take the top N. Scores from different deal-score definitions are not comparable: with several maps, take the top few per map, never one sorted list.
3. Continue with Path A steps 2 to 5. List and map members are already saved.

## All paths end

Deliver with the output contract. Then offer either a list of dates to hold on the calendar, or to run the conference-screen skill on the conference the user picks.

## Output contract

1. One line: the path taken, the inputs (companies, or sector and countries and window, or lists and score), how many companies or editions were read, how many histories were never fetched, and how many pages the search returned.
2. Table of the top 8 to 10 conferences: Conference, Dates, City, Why it fits (Path B) or Which of your companies attend (Paths A and C, with the role each has), Confidence (confirmed from the attendee list, from the company's fetched history, predicted from prior years, or unverified from the web).
3. One recommendation, with the reason in one sentence.
4. For Paths A and C: the companies whose history was never fetched, so the user can press Refresh in the app and run again.
5. The limits below that apply to this answer.
6. The offer: calendar dates, or conference-screen.

Everything in the table comes from a Raylu tool result, except rows labelled unverified from the web. No conference is placed in the table from memory.

## Limits to state plainly

- The catalogue filters by 30 fixed industries and by country only.
- A history that was never fetched means "not fetched yet", never "attends nothing". Matching is by exact website, so brands and regional domains are missed. History is a shared cache across Raylu.
- Confirmed attendance exists only 3 to 4 months out. Further out is predicted from prior years and must be labelled as such.
- A conference missing from the catalogue takes about 24 hours to request.

## Common mistakes

- Sorting scores from different deal-score definitions into one list.
- Reading "never fetched" as "attends nothing".
- Starting an attendee pull to confirm a conference without asking.
- Mapping a thesis onto industries without showing the mapping.
- Passing a name query together with dates or an event type; the tool rejects that pair.
- Trusting a page count as a total.
- Treating a provider outage as a short list.
- Placing a conference in the table from general knowledge.
