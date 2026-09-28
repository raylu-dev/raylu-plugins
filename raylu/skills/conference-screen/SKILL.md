---
name: conference-screen
description: Use when the user is going to a conference and wants to know which attending companies to meet and what to talk about. Triggers on "who should we meet at [conference]", "who's worth a meeting at [conference]", "prep me for [conference]", "which attendees at [conference] are in our CRM", or a conference name plus a request for meetings, targets or prep pages.
---

# Conference Screen

Answer "who should we meet at this conference, and what do we talk about?" from the conference's attendee list in Raylu, the user's deal score, and the user's CRM. The result is a ranked table of companies worth meeting, the ones a teammate already owns set apart, and a one-page prep for each company kept.

Needs the Raylu conference tools, scoring and CRM search on the user's account. If `conference_search`, `conference_attendees`, `score_companies` or `crm_search` is missing from the tool list, say the account is not set up for this skill and stop.

Walk through these steps ONE AT A TIME. Ask one question per message. Skip any question the user already answered.

## Step A: Which conference

Ask: "Which conference and which year?"

Then check whether the attendee list was already pulled: `conference_lookups_list` (free). A `completed` lookup for the right edition is reused by its Lookup ID in Step C. A conference pulled in the past week is served from that same lookup at no extra charge; reuse the Lookup ID anyway, it is faster and skips the wait.

If no lookup exists, find the conference: `conference_search(query: "<name>")`. Show the matching editions (dates, city, size) and ask which one. Keep the edition's Event ID and Edition ID together; Edition ID alone can pull the wrong event.

## Step B: Which deal score, and how many

Ask: "Which deal score should I rank with?" after calling `list_scoring_definitions` (free). The user chooses, never you. If there is exactly one definition, name it and ask to confirm.

Ask: "How many companies do you want back?" Default: the top 25.

## Step C: The attendee list

If a completed lookup exists: `conference_attendees(lookup_id: "<id>")` (free).

If not, ask first: "Pulling the list costs 150 credits (the monthly allowance is 20,000 where one applies), takes a few minutes and adds every attendee to your Raylu project. Go ahead?" Only on a yes: `conference_attendees(edition_id: "<id>", event_id: "<id>")`.

A pull usually outlasts the tool's wait of about 50 seconds. When the tool says the lookup is still running, keep the Lookup ID, wait 60 to 120 seconds, and call `conference_attendees(lookup_id)` again. A running lookup is not an empty result.

Read every page: 100 companies per page (company, domain, tags, lists, team). The tool allows 5 calls per 5 minutes, and the pull, the polls and the pages all count against the same 5, so a 500-company list takes about 5 minutes and 2,000 about 20. Pages come alphabetically, so a partial read is biased: read them all, or say which letters you covered.

Tell the user how many attendees Raylu matched to companies and how many it could not match. A failed lookup is a failure, not a conference with no attendees.

## Step D: Score the whole list

`score_companies(definition_id: "<id>", domains: [...])` with at most 200 domains per call, 1 credit per call, 10 calls per 5 minutes. Domains past 200 come back as skipped, not scored. It returns a ranked table (rank, domain, score, label) and queues the companies with no score yet. Never call `get_run_status` for it.

Wait a minute or two, then read the new scores with `get_score_breakdown(definition_id: "<id>", domains: [only the unscored ones])`. Do not call `score_companies` again to fetch results; that starts scoring over.

A company with no score after that is "unscored", never zero. A domain the tool could not match to a company is "unresolved", and a "Disqualified" label is a real result, not a missing one.

## Step E: Ownership, from the top down

Walk down the ranking until you have enough companies, in two stages:

1. The list's Team column (free): a teammate assigned in Raylu.
2. `crm_search(question: "...")` with at most 40 domains per question and the question under 1,000 characters (25 credits per question, 100-row cap, 10 questions per 5 minutes). If the tool says the CRM schema is still being profiled, that is not an empty CRM: retry after the wait it names. Ask for: owner, whether the owner is the connected user, last touch, last note date. For TA: the Salesforce account owner and `Date_of_Last_Touch__c`; tasks of type Email - Sent, Note, Interest, Pass Lead. Not in the CRM means unowned.

The tool prints the SQL it wrote and the row count. Show both, because an AI wrote the query and the user needs to catch a wrong or empty answer.

Keep companies nobody owns or that the user owns. Companies owned by someone else go to a separate "coordinate first" table. Stop once the kept table is full. Walking down the ranking takes a handful of questions, against about 13 for a 500-company sweep.

## Step F: Prep pages

Write prep pages for the top 10 kept companies by default. Each `meeting_prep_brief(company: "<domain>")` costs 150 credits, so 25 pages is about a fifth of a month's budget. Ask before writing more than 10.

Call `meeting_prep_brief` one company at a time. It saves a company the project has not saved yet as a side effect; say so when that happens. The brief knows nothing about the conference; add that context yourself. Reuse the CRM history from Step E.

People: `company_people_search(company: "<domain>", seniority: "Founder")`, then `seniority: "C-Level"` if no founder appears (free, no emails; those two values are the ones the data uses). Zero people back means unknown, not none. The attendee list has no people or roles, so these are company executives, not confirmed attendees; say so. Emails only if the user wants them: `find_company_contact(query: "<domain>")`, 5 credits, 10 calls per 5 minutes, runs 1 to 3 minutes in the background; read the result with `get_company` afterwards.

## Step G: Deliver

Use the output contract below. Then offer the result as CSV, an email draft, or a Notion page, and offer more prep pages.

## Output contract

1. One line: the edition (name, dates, city), attendees on the list, how many matched and how many did not, the deal score used, how many companies have a score and how many are still queued.
2. Ranked table of kept companies, top N: Company, Domain, Score, Owner status (you, unowned, or the owner's name), Last touch, Why worth meeting (one clause from the score label, the CRM notes or the brief).
3. "Coordinate first": companies owned by someone else, same columns, owner named.
4. One prep page per kept company, top 10 by default: what they do, funding, growth, recent news, the user's CRM history, 5 to 8 questions to ask, 1 to 2 people worth talking to (title and why). Every fact comes from a tool result; say which.
5. Coverage line: pages read and letters covered, companies unscored, CRM questions asked and rows returned, prep pages written.
6. The export offer and the offer of more prep pages.

A blank score, a blank owner or a blank last touch stays blank and is labelled unknown. Never fill a cell from memory. Never say or imply that a company attends because you expect it to; only the list says who attends.

## Common mistakes

- Choosing the deal score instead of asking.
- Starting a pull without checking `conference_lookups_list` and asking for consent.
- Treating "still running" or "failed" as an empty list.
- Reading the first page and calling it the whole list.
- Writing an unscored company as zero.
- Putting a company a teammate owns in the main table.
- Writing more than 10 prep pages without asking.
- Presenting executives from `company_people_search` as confirmed attendees.
- Drafting or sending outreach. This skill stops at the prep pages.
