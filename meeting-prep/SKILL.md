---
name: meeting-prep
description: Prepare briefings for today's meetings — attendee research, email history (Microsoft 365), past meeting notes (Circleback), LinkedIn, and company context. Use when running the daily meeting prep cron, or when user asks to prepare for meetings, review who they're meeting with, or get context on upcoming calls.
---
# Daily Meeting Prep

## Config — read before starting
Read `../config/user.json` (resolves to `~/executive-assistant-skills/config/user.json`).
Extract and use throughout:
- `name`, `full_name` — user's name
- `primary_email`, `work_email` — Microsoft 365 accounts to check
- `notification_email` — inbox that receives the meeting-prep email
- `timezone` — IANA timezone (e.g. America/Argentina/Buenos_Aires)
- `workspace` — absolute path to Hermes workspace (e.g. /home/user/.hermes)
- `obsidian_vault_path` — absolute path to your local Obsidian vault
- `obsidian_meeting_notes_dir` — vault-relative folder where briefs are archived (e.g. `Meetings`)

Do not proceed until you have these values.

## Debug Logging (MANDATORY)
Read `../config/DEBUG_LOGGING.md` for the full convention. Use `python3 {user.workspace}/scripts/skill_log.py meeting-prep <level> "<message>" ['<details>']` at every key step. Log BEFORE and AFTER every MCP call (ms365, circleback), every vault file write, and every web search. On any error, log the full tool args and error before continuing.

## Scope
- Timezone: {user.timezone}
- Calendars: primary of {user.primary_email} AND {user.work_email} (via `ms365` MCP)
- Today's meetings only

**Timezone note:** Use explicit timezone-bounded ISO8601 timestamps for calendar queries. Example: call `ms365.list-calendar-events` with `startDateTime: "2026-03-03T00:00:00-03:00"` and `endDateTime: "2026-03-04T00:00:00-03:00"`. Do NOT rely on implicit "today" boundaries — they use UTC.

- **ALL meetings with attendees** — both external and internal
- Skip personal/solo events with no attendees (e.g. "Personal Trainer", "Golf", all-day reminders)

## Meeting types

### External meetings (attendees outside your email domains)
Full research brief (email context, Circleback, LinkedIn, company research) — see below.

### Internal meetings (all attendees from your email domains)
Lighter brief — no LinkedIn/company research needed, but still include:
- Attendee list
- Circleback context from previous instances of this recurring meeting
- Recent email threads related to the meeting topic/agenda
- Any open action items from last time
- Format: same structure but skip LinkedIn/company sections

### Recurring collaborative meetings (e.g. podcasts, content sessions with external co-hosts)
These are external meetings — give them full briefs. Don't skip recurring meetings just because they're familiar.

## Error handling
- If `ms365.list-calendar-events` fails for one account: continue with the other account, note "⚠️ [account] calendar unavailable" in output.
- If Circleback fails: continue without meeting history, note it per meeting.
- If `ms365.send-mail` fails: save the assembled email body to `{user.workspace}/state/undelivered/meeting-prep-YYYY-MM-DD.md`, log ERROR with the full error, and retry once after 30s. If still failing, leave the file in place so the next run can pick it up.

## For each meeting

### 1. Event basics
Title, local time ({user.timezone}), attendees.

**RSVP status (MANDATORY):** For each attendee, inspect `attendees[].status.response` from the Outlook event:
- `accepted` → no flag needed
- `notResponded` → flag as "⚠️ hasn't responded"
- `declined` → flag as "❌ declined"
- `tentativelyAccepted` → flag as "❓ tentative"

If ANY non-organizer external attendee has NOT accepted, add a visible line in the brief:
> ⚠️ *RSVP:* <name> hasn't accepted yet

This is informational — it doesn't mean they won't join, but it's useful to know ahead of time, especially for first calls or important meetings.

### 2. Email context (90-day lookback, with historical fallback)
Search Outlook on both accounts for exchanges with attendees using the `ms365` MCP. For EACH attendee, try these strategies in order:

1. **Email address** (primary — from calendar invite):
   `ms365.search-mail-messages --args '{"query": "from:<email> OR to:<email>", "top": 20}'`
2. **Full name**: `"firstname lastname"`
3. **First name + "intro"**: `"intro firstname"` (catches informal intro subjects)
4. **First name + company**: `"firstname companyname"`

The attendee's email from the calendar invite is the most reliable identifier — always start there.

**Intro discovery** (after general email search):
5. Search for intro emails involving the attendee: `subject:intro <email>`, `subject:intro <firstname>`, `subject:introduction <email>`
6. Also check threads where a third party CC'd/introduced the attendee

**Recent email context** (after intro discovery):
7. Pull the most recent threads with this attendee (by email address) to surface any recent updates, asks, or context leading into today's call.

**Historical fallback** (if no results from 90-day search):
8. Run a broader search with NO date filter — `ms365.search-mail-messages` without a date scope. This catches long-standing relationships where the last email was months/years ago. If older threads exist, this is NOT a first call — note the relationship history.

- First call vs follow-up? Base this on ALL email history found (including historical), not just 90-day window
- **If first call: MUST include "who introduced + when" (date) if found in email; if not found, explicitly say "No intro trail found in email"**
- If follow-up: extract updates since last call
- **If email contains a concrete commercial trigger** (pricing, deliverables, scope, budget, urgency, timeline, decision-maker request), include it explicitly in the brief

### 3. Circleback context
**Search by ATTENDEE, not by meeting title.** The same recurring meeting may have different titles week to week. Always search by the attendee's name or email to find all past meetings with them.

Call the Circleback MCP:

```
circleback.search_meetings --args '{"query": "meetings with <attendee full name>", "limit": 10}'
```

Fallback: if the attendee name yields no results, search by company.

```
circleback.search_meetings --args '{"query": "meetings with <company name>", "limit": 10}'
```

Then fetch the full transcript / AI notes for the best match:

```
circleback.get_meeting --args '{"meeting_id": "<id>"}'
```

Circleback covers meetings + calendar + email in one connector, so you can also cross-reference its calendar view for recent events the attendee joined:

```
circleback.list_calendar_events --args '{"attendee_email": "<email>", "since": "<ISO date 3 weeks ago>"}'
```

- **Recent (< 3 weeks):** Provide a richer summary (not one-liner): decisions made, key tensions, explicit action items, owners, and unresolved questions
- **Older (3+ weeks):** Broader context — relationship history, past decisions, recurring themes
- **No attendee match but company match exists:** Use company-level context and label it clearly as company-level
- **No results / first meeting:** Note that, provide email context instead
- Preserve any source links Circleback returns (`[[N]](url)`)
- Include a short explicit line: **"Why this meeting now"** based on prior action items or current email trigger
- **Exact name matching**: When attributing Circleback results to an attendee, verify BOTH first AND last name match exactly. Different people can share a first name — never assume a match based on first name alone.
- **Auth failure:** If Circleback returns an auth error, re-run the OAuth flow (`hermes tools call circleback.list_meetings --args '{"limit": 1}'`) and retry once. If still failing, note "⚠️ Circleback unavailable" and continue without it.
- **Empty summary:** If Circleback returns a meeting record but with no/empty summary, note "Previous meeting found but no summary available" — don't silently skip it.

### 4. LinkedIn research
- Search: `"[attendee name] [company] LinkedIn"`
- Extract: current role, background, recent posts/activity

### 5. Company research
- Search: `"[company] recent news"`
- Search: `"[company] funding crunchbase"` (if startup/VC relevant)
- Extract: company stage, announcements, what they do

## Research rules
Read `{user.workspace}/style/MEETING_PREP_RULES.md` for additional research steps.

## Output format
Send the brief as a single email via `ms365.send-mail`:

- `from`: `{user.notification_email}`
- `to`: `{user.notification_email}`
- `subject`: `[EA] Meeting Prep — <YYYY-MM-DD> (<N> meetings)`
- `bodyType`: `HTML` or `Text` (Text is fine — Outlook renders the markdown-ish layout cleanly)
- `body`: intro line + one block per meeting, in chronological order, each block separated by a horizontal rule (`---`)

**Also archive to Obsidian:** Immediately after sending the email, write a note into your vault on disk at `{user.obsidian_vault_path}/{user.obsidian_meeting_notes_dir}/YYYY-MM-DD — Meeting Prep.md` containing the full markdown briefs. Use the `Write` tool (or `mkdir -p` + `Write`) to create the file directly — no MCP call required.

**Never collapse the per-meeting sections into one summary block.** Each meeting gets its own clearly-labeled block in the email body; the reader skims headers in chronological order.

Start the email body with a short intro: "📋 **MEETING PREP — <day>** — <N> meetings (<X> external, <Y> internal)"

Then one block per meeting in this format — use bold subsection labels and blank lines between each section for readability:
```
*<number>. <Name/Company> — <local time>*

*Who:* <Role>, <Company> (<location>). <What the company does, 1 sentence>. <Funding/stage if relevant>.

*Context:* <First call vs follow-up>. <If first call: who intro'd + when (date); if unavailable: "No intro trail found in email">.

*Email history:* <Key email context — include important commercial/decision triggers when present (pricing, scope, deliverables, urgency, budget, decision-maker request)>.

*Circleback:* <Richer recap: key decisions, action items, owners, unresolved questions, and why a follow-up was needed. If no attendee notes, use company-level notes and label it. Or "No previous meetings found in Circleback">.

*Why this meeting now:* <One sentence grounded in prior action items and/or current email trigger>.

*Focus areas:* <ONLY items derived from prior action items and current email trigger — not generic strategy prompts>.

*Links:* <LinkedIn, company site, Crunchbase>
```

Each section on its own paragraph (blank line before each bold label). Keep it concise but well-structured — readability over density.

If a meeting already happened, prefix with ✅ and keep brief.
If there's a schedule conflict, flag with ⚠️.

## Save full brief
Save the full detailed brief to `{user.workspace}/state/meeting-prep-YYYY-MM-DD.md` (for local dedup/state) and archive a copy in Obsidian per the step above. The same markdown is the email body — no attachment needed.

## ⚠️ CRON CREATION (CRITICAL — DO NOT SKIP)
This section is NON-OPTIONAL. Cron creation MUST happen for every run with meetings. If you run out of context or time before completing this section, the entire run is a FAILURE.

**Execution order:** Create ALL crons IMMEDIATELY after saving the brief file — BEFORE the assertions step. Do not defer cron creation to "after everything else."

Log: `python3 {user.workspace}/scripts/skill_log.py meeting-prep INFO "Starting cron creation for N meetings"`

## Pre-meeting reminders
After generating all briefs, create a one-shot cron job for EACH meeting that fires 5 minutes before start time. The cron job should:
1. Read `{user.workspace}/state/meeting-prep-YYYY-MM-DD.md`
2. Find the section for that specific meeting
3. Send the FULL formatted brief for that meeting via `ms365.send-mail`
   - `from` / `to`: `{user.notification_email}`
   - `subject`: `[EA] ⏰ 5-min reminder: <meeting title> (<local time>)`
   - `body`: the verbatim meeting block

**Hard formatting contract (no exceptions):**
- The reminder body must be copied verbatim from `{user.workspace}/state/meeting-prep-YYYY-MM-DD.md` for that meeting block.
- Do NOT rewrite, summarize, translate, normalize, or reformat any part of that block.
- Keep language exactly as generated in the source brief.
- Only allowed change is the subject-line prefix. The body is pasted unchanged.

Use `hermes cron add` with `--at` set to 5 min before meeting time, `--delete-after-run`, and `--no-deliver`. The `--no-deliver` flag prevents the announce mechanism from sending anything — the task sends the reminder email directly via `ms365.send-mail`.

**Hard requirement:** after creating jobs, run `hermes cron list` and verify the expected number of `pre-meeting-` jobs for today. If count is lower than expected, immediately retry creation and report failure explicitly.

Log each created cron: `python3 {user.workspace}/scripts/skill_log.py meeting-prep INFO "Created pre-meeting cron" '{"meeting": "<name>", "fires_at": "<time>"}'`

## Post-meeting action items + drafts
After generating all briefs, create a one-shot cron job for EACH meeting that fires 10 minutes after the meeting END time. The cron task should reference the action-items-obsidian skill:

Task: "Read and follow ~/executive-assistant-skills/action-items-obsidian/SKILL.md. Process ONLY the meeting titled '<meeting title>' that ended around <end time>. Email the summary to {user.notification_email}."

Use `hermes cron add` with `--at` set to 10 min after meeting end time, `--delete-after-run`, `--session isolated`, `--timeout-seconds 1200`, and `--no-deliver`. Name them `post-meeting-<short-name>`.

**Hard requirement:** after creating jobs, run `hermes cron list` and verify the expected number of `post-meeting-` jobs for today. If count is lower than expected, immediately retry creation and report failure explicitly.

Log each created cron: `python3 {user.workspace}/scripts/skill_log.py meeting-prep INFO "Created post-meeting cron" '{"meeting": "<name>", "fires_at": "<time>"}'`

Log final count: `python3 {user.workspace}/scripts/skill_log.py meeting-prep INFO "Cron creation complete" '{"pre_meeting": N, "post_meeting": M, "expected": E}'`

**If cron count doesn't match expected:** Log ERROR and send an email to `{user.notification_email}` with subject `[EA] ⚠️ Meeting prep incomplete` and body "Only created X/Y pre-meeting and A/B post-meeting crons. Some reminders/action-items may be missing."

### Deduplication (MANDATORY)
After processing, the cron MUST append the meeting title to `{user.workspace}/state/processed-meetings-YYYY-MM-DD.json` (array of meeting titles already processed). This lets the end-of-day catch-all skip them.

**Before creating ANY Obsidian task**, the cron MUST:
1. Read `{user.workspace}/state/processed-meetings-YYYY-MM-DD.json` — if this meeting is already listed, SKIP entirely (another cron already handled it)
2. Read the Obsidian tasks file on disk (`{user.obsidian_vault_path}/{user.obsidian_tasks_file}`) and check for duplicates by matching existing task lines against the new task intent (same person + same action = duplicate)
3. If a matching task already exists, do NOT create a duplicate — skip it silently

This prevents the scenario where a post-meeting cron and the daily end-of-day cron both process the same meeting and create duplicate tasks.

## Sanity checks
- **Calendar is source of truth for meeting count**: Cross-reference email threads with the actual calendar events. If an invite was moved/rescheduled, it's still ONE meeting — don't count it as multiple. Check the Outlook event ID, not email threads, to determine unique meetings.
- **First call vs follow-up**: Verify by checking if there is an ACTUAL past Circleback meeting with this specific person (exact name match). Rescheduled invites or multiple scheduling emails do NOT make it a follow-up. Only a previously held meeting does.
- **Email sent check (MANDATORY):** Exactly ONE meeting-prep email must be sent per run. The email body must contain a block for every meeting with attendees (external + internal). If any meeting block is missing from the sent body, send a follow-up email with the missing blocks immediately.
- **Cron count check (MANDATORY):** Number of `pre-meeting-` jobs and `post-meeting-` jobs created for today must each equal number of meetings with attendees (external + internal).

### Automated assertions (MANDATORY)
After sending all meeting messages and creating all one-shot jobs, run:

```bash
python3 {user.workspace}/scripts/meeting_prep_assertions.py \
  --date YYYY-MM-DD \
  --brief-file {user.workspace}/state/meeting-prep-YYYY-MM-DD.md
```

- If exit code is 0: proceed normally.
- If exit code is non-zero: create missing cron jobs and/or send missing meeting messages, then re-run up to 2 times. If still failing after 2 retries, report the assertion output in your completion note and proceed.
- Include the assertion result summary in your final internal completion note.

## Meeting type-specific enrichment

### Deal flow calls (new companies, potential clients/advisory)
- Extract MORE detail from email threads: what the company does, product description, funding status, round size, investors, ARR if mentioned
- Search Crunchbase/web for latest funding info if not in emails
- Include company stage, team size, and key metrics when available

### Investor/VC calls
- Include link to the fund's profile page (website, Crunchbase, or AngelList)
- Note their investment thesis, typical check size, and stage focus if findable
- Helps identify fit before the call

## Scheduling difficulty flag
- If a meeting took a long time to schedule (intro was weeks/months before the actual meeting), flag it: "⏳ *Scheduling note:* Intro came in <date>, took <N weeks/months> to get on the books."
- If the meeting was rescheduled multiple times, note how many times
- This provides useful context on the relationship dynamic and signals the meeting may be higher-stakes

## Rules
- Executive style, concise
- No meetings with attendees today → NO_REPLY
- Missing data → state briefly ("No email history found"), don't invent
- Never silently omit a data source — if something returned nothing, say so
- **No cross-contamination**: Each meeting brief must only include information verified for THAT specific attendee. Do not mix up intro sources, email threads, or Circleback notes between different meetings. Double-check that every fact in a brief belongs to the correct person.
- **No generic focus areas**: Focus must be anchored in (a) explicit prior action items from Circleback and/or (b) explicit email trigger for this meeting. If neither exists, say so and use a discovery focus.
