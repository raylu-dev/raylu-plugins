---
name: outreach
description: Use when the user wants to build and launch a multi-touch outreach sequence for a specific contact at a specific company — verifying the company, finding the contact, choosing or creating a cadence, setting the personalization angle, generating the campaign, and queueing the first send. Trigger phrases include 'reach out to this company', 'build an outreach sequence', and 'draft and queue emails to...'.
---

# Outreach Sequence Builder

Build and launch an outreach sequence for a specific contact at a specific company. Walk through these steps ONE AT A TIME. Do not skip steps.

## Step A: Identify the Target Company
Ask: "Which company do you want to reach out to? Give me a name or domain."
- Call search_companies if the name is ambiguous (e.g., "Delve" could be delve.co, delve.ai, or delvehealth.com). Present top candidates and let the user pick.
- Call get_company with the confirmed name/domain to pull the firmographic profile. Present headcount, location, funding, industry. Confirm with the user.
- If the company is not yet tracked, call lookup_company.

## Step B: Find the Right Contact
Ask: "Who should we reach out to? Do you have a specific person in mind, or should I find the best contact?"
- If the user names someone: confirm the role/title and use that.
- If not: call find_company_contact to retrieve and verify the primary contact (email + phone + LinkedIn).
- Present the contact and confirm before proceeding: "I found {name}, {title} at {company}. Email: {email}. LinkedIn: {url}. Use this contact?"

## Step C: Choose or Build the Sequence
Ask: "What kind of outreach sequence are you running?"
- Call list_strategies to show available cadences (e.g., "Founder Intro", "Diligence Outreach", "Follow-up After Event").
- Present the options with step counts and intervals so the user can pick.
- If none fit, ask the user to describe the cadence they want and call create_strategy with:
  - Number of steps (e.g., 4-touch sequence)
  - Days between steps (e.g., Day 0, Day 3, Day 7, Day 14)
  - Channel mix (email only, email + LinkedIn, etc.)
  - Tone (formal, casual, urgent, warm intro)

## Step D: Set the Personalization Angle
Ask these one at a time. The answers feed into the campaign generation as context:
1. "What's the hook? What about this company is interesting to you right now? (recent funding, product launch, hiring spike, market trend, intro from a mutual)"
2. "What's the ask? (intro call, diligence access, market education chat, founder dinner, etc.)"
3. "Any context about your firm or thesis we should reference?"
4. "Anything to AVOID mentioning?"

## Step E: Generate the Campaign
Call generate_campaign with:
- companyId (from Step A)
- contactId (from Step B)
- strategyId (from Step C)
- personalization context (from Step D)

This produces the full sequence — first email plus every follow-up, each personalized per step. Present the first email (subject + body) to the user immediately. Tell them how many follow-ups are queued.

## Step F: Review and Edit Each Email
For each email in the sequence:
- Present subject + body
- Ask: "Send as-is, edit specific lines, or rewrite from scratch?"
- If edit: call update_email with the specific changes (e.g., "shorten the opening", "swap the case study").
- If rewrite: call rewrite_email with a directional note (e.g., "make it more direct", "drop the buzzwords", "lead with the mutual connection").
- Repeat until the user approves the email.

## Step G: Queue and Launch
Ask: "Ready to queue the first email and start the sequence?"
- Call get_daily_send_count to confirm the user isn't over their daily send cap.
- Call start_campaign to activate the sequence.
- Call complete_campaign_step to queue the first email for send.
- Call get_outreach_schedule to show when the next emails in the sequence will go out.

Confirm: campaign is live, first email queued, follow-ups scheduled. Tell the user how to pause or cancel if needed (`pause_campaign`, `cancel_campaign`).
