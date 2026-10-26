---
name: companies
description: Use when the user asks about a specific company, wants to find or search companies by criteria (headcount, location, funding stage, industry, keywords), look up an investor's portfolio, analyze a company's team, prep for a meeting with a company, find conferences and their attendees, or check which conferences a company attends. Treats Raylu as the source of truth for company data. Trigger phrases include 'tell me about <company>', 'find companies that...', 'who has <investor> backed', and 'prep me for my call with <company>'.
---

# Raylu Companies

**Autonomy rule:** Once you know what company the user is asking about, look it up immediately. Do not ask if they want you to look it up.

## Mode A: Company Lookup

Use this when the user mentions a specific company by name or domain.

### Process

1. **Verify the company identity.** The user usually only has a name. Names are ambiguous — "Delve" could be delve.co, delve.ai, or delvehealth.com. Before looking anything up:
   - Call search_companies with the company name
   - If exactly one strong match, proceed with that domain
   - If multiple plausible matches, present the top candidates (name + domain + description) and ask which one they mean
   - Use conversation context to disambiguate when possible

2. **Fetch the full profile.** Once you have the verified name or domain:
   - Call get_company with the verified query
   - This returns firmographic data (headcount, location, funding, industry) and contacts if the company is saved

3. **Show where each fact comes from.** Show the structured data from Raylu. If you add anything from another source, say so.

4. **Suggest next actions only when appropriate:**
   - ONLY if the tool response shows the company is NOT saved: offer to save it
   - ONLY if the company is saved AND has zero contacts: offer to find a contact
   - Do NOT suggest saving a company that is already saved
   - Do NOT suggest finding contacts if contacts are already listed

### Listing Companies

When the user asks to list or show their companies:
- Call list_companies — it returns a table with name, domain, location, headcount
- Present the table directly. Do NOT summarize it into prose.
- If the user asks follow-up questions about specific companies, use get_company or get_companies

### Multi-Company Lookup

When the user mentions multiple companies, use get_companies (bulk lookup) — do NOT call get_company multiple times in parallel.

## Mode B: Firmographic Search

Use this when the user wants to find companies matching criteria.

### Process

1. Parse the request into structured filters
2. Call firmographic_search with the filters array
3. Present results: total count, preview companies, distribution stats
4. If result set is very large (1000+), suggest narrowing filters

### Column Reference

| Column | Type | Operators | Example Values |
|--------|------|-----------|----------------|
| headcount | number | gt, gte, lt, lte, eq | employee count, e.g. 50, 250, 1000 |
| country | string | eq, in, like | ISO 2-letter codes: 'US', 'GB', 'DE' |
| state | string | eq, in, like | 'California', 'New York', 'Texas' |
| city | string | eq, in, like | 'San Francisco', 'New York', 'Austin' |
| last_funding_stage | string | eq, in | 'Pre-Seed', 'Seed', 'Series A', 'Series B', 'Series C' |
| last_funding_amount | number | gt, gte, lt, lte | dollar amount |
| total_funding_amount | number | gt, gte, lt, lte | dollar amount |
| financing_status | string | eq, in | 'VC-backed', 'Bootstrapped', 'PE-backed' |
| founded_date | number | gt, gte, lt, lte, eq | year (e.g., 2020) |
| revenue | number | gt, gte, lt, lte | dollar amount |
| monthly_visits | number | gt, gte, lt, lte | visit count |
| status | string | eq | 'active', 'acquired' |
| industries | JSONB array | contains, contains_any | broad canonical labels: 'Financial Services', 'Software Development', 'Healthcare' |
| keywords | JSONB array | contains, contains_any | niche/product/category terms: 'fintech', 'saas', 'ai', 'payments', 'cybersecurity' |
| all_investors | JSONB array | contains, contains_any | 'Sequoia Capital', 'a16z' |
| all_funding_leads | JSONB array | contains, contains_any | 'Accel' |

### Filter Rules
CRITICAL: Only use column names EXACTLY as listed in the table above. Do not invent column names. Common mistakes: 'funding_stage' (wrong) → 'last_funding_stage' (correct), 'industry' (wrong) → 'industries' (correct).
- Headcount is numeric. For "less than 250 employees", use { column: 'headcount', operator: 'lt', value: 250 }.
- Country uses ISO 2-letter codes ('US', 'GB', 'DE'). State and city use full names ('New York', 'San Francisco').
- For "funded" or "has funding", use { column: 'total_funding_amount', operator: 'gt', value: 0 }.
- For specific funding stages, use { column: 'last_funding_stage', operator: 'in', value: ['Seed', 'Series A', 'Series B'] }.
- Funding amounts and revenue are raw numbers in dollars.
- Use 'in' for multiple values, 'gte'/'lte' for ranges.
- JSONB array columns (industries, keywords, all_investors, all_funding_leads) use exact, case-sensitive values.
- Use 'contains' when filtering on one exact JSONB array value.
- Use 'contains_any' when one concept may be tagged several ways. It ORs exact JSONB containment checks while staying index-friendly.
- Use keywords contains/contains_any for niche terms such as fintech, saas, ai, payments, cybersecurity, vertical software, and other product/category descriptors.
- Use industries contains <label> only when the user names a broad canonical industry label such as Financial Services, Software Development, or Healthcare.
- For compound concept searches, create one contains_any filter per concept and let filters AND together. Include compound tags in every concept group they satisfy.
- Example for "financial AI companies in NYC under 250 employees with funding":
  filters: [
    { column: 'keywords', operator: 'contains_any', value: ['financial ai', 'financial technology', 'financial services', 'finance/investment', 'fintech'] },
    { column: 'keywords', operator: 'contains_any', value: ['financial ai', 'artificial intelligence (ai)', 'artificial intelligence', 'ai', 'machine learning'] },
    { column: 'city', operator: 'eq', value: 'New York' },
    { column: 'headcount', operator: 'lt', value: 250 },
    { column: 'total_funding_amount', operator: 'gt', value: 0 }
  ]
- Columns NOT available: naics_codes, sic_codes, description

## Mode C: Conferences and company attendance

Use the single `conferences` MCP tool for conference discovery, company attendance, attendee rosters, and conference actions. Its input is `{"request":{"operation":"...", ...}}`. Inspect the live tool schema for the fields accepted by each operation. Do not look for separate conference tools.

| Operation | Purpose |
|---|---|
| `search` | Search or browse the full conference catalogue with filters, dates, sort, and paging. |
| `locations`, `sectors` | Read valid catalogue filter values. |
| `company_attendance` | Read the conferences a company attends by name, domain, or canonical ID. |
| `refresh_company` | Discover or refresh company attendance through the provider workflow. |
| `attendees` | Read a saved lookup, or start one using the exact event and edition IDs from search. |
| `submit` | Add a conference by source URL and available metadata, then read its lookup status. |
| `saved` | List saved conference lookups in the current project or for the current user. |
| `refresh` | Refresh a saved lookup or retry a failure with `mode: "refresh"` or `mode: "retry"`. |
| `create_list`, `export` | Create a Raylu list or return complete CSV for a frozen attendee selection. |
| `mark_seen` | Acknowledge the attendee snapshot the user has reviewed. |
| `delete` | Remove the user's saved lookup, preserving shared attendance data. |

### Read first, then act

- Read company attendance with `{"request":{"operation":"company_attendance","company":"example.com"}}`. Resolve ambiguous companies before continuing. A read does not save the company or start provider work.
- Search by conference name, topic, website, location, or filters. Use returned event and edition IDs together with a stable UUID `request_id` to start an attendee lookup. Reuse that UUID when retrying the same start. Reuse `lookup_id` for subsequent reads.
- Report pending, partial, failed, and genuinely empty results distinctly. Include source dates and freshness. Attendance indicates recorded participation, not a guaranteed future appearance.
- Continue paging when a complete result set is requested. Preserve `snapshot_revision` and `visit_token` from the same lookup and user. Do not acknowledge unseen rows.
- For lists and exports, use the selection schema exactly. Choose all rows in a snapshot, explicit canonical company IDs, or matching filters with exclusions. Do not treat the displayed page as the full selection.
- Starting an attendee lookup saves a personal conference lookup. Starting, refreshing, retrying, and submitting can start provider work and add resolved attendees to the project as companies. Submission also saves a personal conference. Reading an attendee page can add its returned companies to the project without starting a crawl. State these effects before acting. Use a stable request ID for retries of the same action.
- Use `delete` only for the saved lookup the user explicitly requested removing; clarify its identity if ambiguous. For a requested list or export, execute the requested action without adding a separate approval step.
- Export returns CSV content. If it exceeds the tool's limit, report the error and narrow the selection. Never present truncated CSV as complete or invent a download link.

If `conferences` is absent, report that conference access is unavailable for the connected account. Do not substitute general web knowledge for the user's Raylu data.
