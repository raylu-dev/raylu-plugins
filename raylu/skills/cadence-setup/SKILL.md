---
name: cadence-setup
description: Use when a Raylu user already has an email sequence somewhere else (Outreach, Apollo, SalesLoft, HubSpot, a Google Doc, one pasted email) and wants it in Raylu as a cadence. Triggers on "set up my cadence", "import my sequence", "put this in Raylu", "here's what I use in Outreach", "recreate my template", or a pasted multi-step outreach sequence with a request to build it.
---

# Cadence Setup

Port an existing outreach sequence into a Raylu cadence, word for word, with the fewest possible questions. The user's copy is the product. CC and a send window are offered after creation; Raylu's AI personalization, attachments, and send approvals are configured in the app, and your closing message tells them exactly how.

## When to use

- The user has something to port: an export, a pasted sequence, a single email plus "follow up twice".
- Not for designing a cadence from scratch. If they have nothing to port, say so in one sentence and point them to Raylu's default cadences (Cold Outbound, Warm Follow-up, Conference) or ask what they want to say. Do not build anything.

## Flow

Ask one question per message. Never bundle questions. If the user already answered something, skip it.

1. **Source.** If nothing was pasted or attached: "Paste or attach the sequence you use today. An export, a doc, or the emails themselves all work."
2. **Parse** into steps: order, timing, channel, subject, body, thread behavior. Apply the fixed rules below without asking.
3. **Resolve unknowns.** Only ask when a rule below says to.
4. **Check the name** with `list_strategies` before confirming, so the summary shows the final name. Name precedence: a name the user used in chat ("my founder cadence" becomes "Founder Cadence": drop leading possessives, title case), else the source's sequence name, else the first subject line with variables stripped. If it already exists, append today's date.
5. **Confirm** with one short summary, then wait: cadence name, step count, thread mode in plain words ("follow-ups reply in the same thread" or "each email is its own thread"), and one line per step (day, channel, first few words). "Create this as-is?"
6. **Create** with one `create_cadence` call.
7. **CC.** Ask: "Anyone to CC or BCC on every email in this cadence? Say no to skip." If yes, apply with `edit_cadence(cadence, config: {ccRecipients: [...]}, applyToDrafts: false)` or `bccRecipients`, passing only the one they gave.
8. **Send window.** Ask: "Want a send window, for example weekdays 9am to 5pm Eastern? Say no to keep Raylu's defaults." If the source carried a window, offer that as the default. If yes, apply with `edit_cadence(cadence, config: {earliestMinute, latestMinute, weekDaysOnly, timezone})`. Minutes count from midnight US Eastern regardless of the user's timezone: 9am ET is 540, 5pm ET is 1020. Convert other timezones to Eastern before setting, and pass the zone the user named as `timezone` (for example America/Los_Angeles) so the Builder displays the window in their time. Pass `applyToDrafts: false` on every edit; a new cadence has no drafts.
9. **Close** with the closing message contract below. Every slot, every time.

## Fixed rules (apply silently)

| Source detail | What to do |
|---|---|
| Every email step | `type: "email"`, `mode: "template"`, `subject`, `template`, `delay`, and a short `name` from the source's step label or "Email 1", "LinkedIn 2", "Call 3" |
| First step | `delay: 0`, whatever the source calls it (Day 0, Day 1, Step 1) |
| Later steps timed as "Day N" | Raylu delays are gaps from the previous step. Day 1/4/9/12 becomes 0/3/5/3 |
| Timing given as "wait N days" | Use N directly |
| Timing that could be read either way ("after 9 days", "+9 days", "9 days later") with nothing in the source fixing the anchor | Ask, naming the step: "Step 3 says 'after 9 days'. Is that 9 days after step 2, or 9 days after the first email?" Ask only when it is unclear: a source that says "Day N" throughout, says "wait N days", has an explicit gap or interval column, or spells out the anchor once ("all timings from send") needs no question |
| Follow-ups reply in thread, or thread behavior unstated | `cadenceMode: "RESPOND_IN_THREAD"`, the first email's subject copied onto every email step (including steps with no subject), no "Re:" prefix. Raylu takes the thread subject from the first step, so a different subject on a later step would be discarded |
| Every step opens a new thread | `cadenceMode: "SEPARATE_EMAIL"`, each subject kept as written. Every email step needs its own subject in this mode. Where the source has none (a reply-style follow-up), ask for one, naming the step. Never pass an empty subject: Raylu would send that step under the project's default subject, or "Opportunity to Partner with [Company Name]" |
| Mixed: some replies, some new threads (signals: "Re:" subjects, "my note below", "bumping this" versus a different subject, "Fresh thread", "new thread") | Raylu sets threading once per cadence. Ask, filling in the real step numbers and phrases: "Step 4 replies in the thread but step 5 starts a new one. Raylu can only do one or the other for the whole cadence. Reply-in-thread keeps the follow-ups attached but drops step 5's subject line; separate emails keep every subject but step 4's 'my note below' will have nothing below it. Which do you want?" Recommend reply-in-thread unless the source has an explicit thread-break subject, in which case present both neutrally. Then name any step whose premise no longer holds under the choice and ask in the same message whether they want to give new copy for that step or keep it as is |
| First name: `{{first_name}}`, `{{contact.first_name}}`, `{{person.first_name}}`, `{{prospect.first_name}}`, `{{lead.first_name}}` | `[First Name]` |
| Company: `{{company}}`, `{{company_name}}`, `{{account.name}}`, `{{prospect.company}}`, `{{organization}}` | `[Company Name]` |
| Sender: `{{sender.first_name}}`, `{{sender_first_name}}`, `{{sender.name}}`, `{{user.first_name}}`, `{{my.name}}`, `{{owner.first_name}}` | `[Your Name]`. Any field whose meaning is plainly first name, company, or sender name maps the same way without asking, whatever the vendor's spelling |
| `{{city}}` / `{{state}}` / `{{country}}` | `[City]` / `[State]` / `[Country]` |
| `{{sender.signature}}`, signature blocks, "Sent from" footers, and any merge fields inside them | Delete without asking. Raylu appends the user's signature. This rule wins over the ask-about-unknown-fields rule |
| A typed sign-off name ("Thanks,\nDan", or a bare "Alex" as the last line) | Keep the closing word if there is one, replace the name with `[Your Name]` |
| A generic greeting ("Hi there,") | Keep it as written. Do not upgrade to `[First Name]` |
| Any other merge field (`{{recent_news}}`, `{{industry}}`, `{{trigger_event}}`, `{{custom_1}}`) | Ask, one question per distinct field, showing every affected sentence as it would read after removal and, where removal leaves a fragment, the shortest bridge that fixes it: "Your sequence uses `{{trigger_event}}`. Raylu has no field for that. Removing it leaves 'Saw that [Company Name] – congrats.' I'd write 'Saw the news at [Company Name] – congrats.' Take that, or give me fixed text?" Never add more than the bridge |
| The user explicitly asks for AI personalization on a step | Honor it, in the safest form: keep `mode: "template"`, put one `[personalization 1]` block on its own line where they said (after the greeting if unspecified), and set `personalizationStrategies: {"1": <their words, as one instruction, even if they named two topics; firm facts from their own copy may be included>}`. Never weave a block into the middle of a sentence. In the confirmation summary give the step its own sentence: which step, where the block sits, what it will write about. Without an explicit ask, never add a block |
| LinkedIn connection note | `type: "linkedin"`, `linkedinActionType: "connection_request_with_message"`, `mode: "template"`, `template` = the note. Over 300 characters: ask them to shorten |
| LinkedIn connect with no note | `type: "linkedin"`, `linkedinActionType: "connection_request"` |
| Call or task step | `type: "phone"` when the content is a call, whatever the source's label; otherwise `"task"`. Give it a `description`. `[First Name]` and `[Company Name]` are fine in the description; there is no phone or email variable, so write "see the contact record" in place of `{{phone}}` |
| Follow-ups described but not written ("follow up twice, a week apart") | Ask for the follow-up copy. If they say "just write short bumps", write two-sentence bumps in the same thread that reuse their own words and say nothing new |
| Send window, timezone, weekdays in the source | Do not set `config` on create. Offer them in step 8, prefilled from the source |
| HTML | Strip tags, keep paragraph and line breaks. Hyperlinks are the exception: replace each anchor with `[Link N]` (numbered 1, 2, 3 within the step) and pass it in that step's `linkConfigs` as `{"N": {"text": "<link text>", "url": "<href>"}}`. Raylu turns `[Link N]` into a real link at send time and has no other way to send one. If a link's destination is not in the source (a merge field or a tracked redirect), ask for it |

Never convert an unsupported merge field into an AI prompt or a `[personalization N]` block on your own. The user asked for their sequence, not a rewrite.

## Closing message contract

Send this after `create_cadence` succeeds and any edits are applied. Fill every slot. Keep the order.

1. **Confirmation.** The cadence name, the thread mode in plain words, and a one-line-per-step table: step, channel, gap in days, subject or first words. If a personalization block was added, say which step and what it will write about.
2. **The auto-send warning.** Word it plainly: as set up, every step sends automatically with no review. To approve each email first, open the cadence in Raylu, use the toggle at the top right to enable manual send, and each step will then appear as a draft in the Campaigns tab.
3. **What else lives in the app.** One short paragraph: AI personalization for any step (for example how the firm is differentiated, or a market trend), and attachments. Mention CC and the send window only as things they can change later in the cadence editor, stating what was set if they chose any.
4. **Non-email steps.** LinkedIn, call, and task steps must be marked done in Raylu before the next step goes out. If the cadence has none, one clause saying it is email only.
5. **The next action.** "Pick a company and I can generate the first campaign with this cadence."

## Common mistakes

- Asking about sign-off name or signature. Both have fixed answers above.
- Reading "after 9 days" as a gap when the source may mean 9 days from the start. Ask when the anchor is unclear; do not ask when the source already fixes it.
- Asking about a vendor's spelling of first name, company, or sender name. Map it.
- Setting a send window the user did not ask for. Leave `config` out of create; steps 7 and 8 ask.
- Turning an unsupported merge field into an AI block on your own. Ask, then remove or replace with fixed text.
- Ignoring an explicit request for AI personalization. Add one block on its own line and confirm it.
- Adding "Re:" to follow-up subjects. Raylu handles threading.
- Stripping a hyperlink to plain text. Links only send through `[Link N]` plus `linkConfigs`.
- Passing an empty subject on a separate-email step. Ask for one instead.
- Confirming a name before checking it exists.
- Reporting success on a failed tool call. Read the response before the closing message.
