---
name: action-items-obsidian
description: "Extract action items from today's Circleback meetings, capture them as tasks in Obsidian, check off fulfilled tasks, and draft meeting-triggered follow-up emails in Microsoft 365. Use when running the daily action items cron, post-meeting cron, or when user asks to process meeting action items. NOT for general email drafting without meeting context."
---
# Action Items → Obsidian + Email Drafts

## Config — read before starting
Read `../config/user.json` (resolves to `~/executive-assistant-skills/config/user.json`).
Extract and use throughout:
- `name`, `full_name` — to identify your action items in meeting notes (e.g. "Martin Gonto")
- `notification_email` — inbox that receives the action-items summary email
- `workspace` — absolute path to Hermes workspace
- `obsidian_vault_path` — absolute path to your local Obsidian vault (just a folder of markdown files)
- `obsidian_tasks_file` — vault-relative path to tasks file (e.g. `Tasks.md`)

The full path to the tasks file is `{user.obsidian_vault_path}/{user.obsidian_tasks_file}`. All vault access is direct filesystem IO — no MCP, no Obsidian-REST plugin. Obsidian picks up the changes the next time it's focused.

Do not proceed until you have these values.

## Debug Logging (MANDATORY)
Read `../config/DEBUG_LOGGING.md` for the full convention. Use `python3 {user.workspace}/scripts/skill_log.py action-items <level> "<message>" ['<details>']` at every key step. Log BEFORE and AFTER every MCP call (ms365, circleback) and every vault filesystem write. On any error, log the full tool args and error before continuing.

## Task format in Obsidian
All tasks go into `{user.obsidian_tasks_file}` as Obsidian Tasks plugin checkboxes:

```
- [ ] <actionable title> 📅 YYYY-MM-DD 🔺 #meeting/<short> #person/<name> [link](<circleback-url>)
```

Priority glyphs: 🔺 = urgent, ⏫ = high, 🔼 = medium, 🔽 = low.
The description lives on the same line as trailing metadata tags + the Circleback link.
Completed tasks become `- [x]` and get a ✅ date suffix: `✅ 2026-04-18`.

## Steps

### 0. Check today's calendar for meetings (BOTH accounts)
Before querying Circleback, get today's actual meetings from BOTH Outlook calendars to know what to expect:

```
python3 {user.workspace}/scripts/skill_log.py action-items INFO "Starting action-items run"

ms365.list-calendar-events --args '{"user": "{user.primary_email}", "startDateTime": "<today>T00:00:00<tz>", "endDateTime": "<tomorrow>T00:00:00<tz>"}'
ms365.list-calendar-events --args '{"user": "{user.work_email}", "startDateTime": "<today>T00:00:00<tz>", "endDateTime": "<tomorrow>T00:00:00<tz>"}'
```

Log the results: `python3 {user.workspace}/scripts/skill_log.py action-items DEBUG "Calendar events found" '{"primary": N, "work": M, "total": N+M}'`

Merge events from both calendars. Filter for meetings with attendees (skip solo/personal events). This gives you the ground truth of what meetings happened today — use it to cross-check Circleback results and catch any meetings Circleback missed.

**CRITICAL date syntax:** Use explicit ISO8601 dates with the user's timezone offset. Do NOT use relative expressions like `today` or `tomorrow` — always compute the actual date strings.

### 1. Get today's meetings from Circleback

```
circleback.list_meetings --args '{"since": "<today>T00:00:00Z", "until": "<tomorrow>T00:00:00Z"}'
```

Collect meeting IDs and titles. Log: `python3 {user.workspace}/scripts/skill_log.py action-items INFO "Circleback meetings found" '{"count": N, "titles": [...]}'`

**Cross-check with calendar:** Compare Circleback meetings against the calendar events from Step 0. If a calendar meeting with attendees has no Circleback match (by time overlap within 15 min), log a warning — it may not have been recorded. Proceed with what Circleback has, but note unmatched meetings in the output.

Skip if no meetings (from either Circleback or calendar).

### 2. Query Circleback for MY action items + email triggers
For each meeting ID, fetch the full AI notes / transcript:

```
circleback.get_meeting --args '{"meeting_id": "<id>", "include": ["transcript", "summary", "action_items"]}'
```

Then mine it for:
- Personal action items, follow-ups, and commitments for {user.name} ({user.full_name})
- Only things HE needs to do, not what others committed to
- The specific person, company, project, or candidate name involved — never use generic references
- Any promises made to do ANYTHING via email (intros, follow-ups, sending docs, sharing info, connecting people, etc.)
- Whether each meeting was a FIRST meeting with that person/company or a follow-up

Preserve Circleback source links for citations.

### 3. Capture tasks in Obsidian

Read the current tasks file (direct filesystem read) to know existing entries and avoid duplicates:

```bash
cat "{user.obsidian_vault_path}/{user.obsidian_tasks_file}"
```

(Use the `Read` tool on the absolute path in-agent.)

For each action item, build the task line per the "Task format" section above. Then append it to the file via plain filesystem append:

```bash
printf '%s\n' '- [ ] <title> 📅 <YYYY-MM-DD> <priority_glyph> #meeting/<short> #person/<name> [link](<circleback_url>)' \
  >> "{user.obsidian_vault_path}/{user.obsidian_tasks_file}"
```

(Prefer the `Edit` tool to insert lines in a specific section when the tasks file is organized by heading — otherwise append at EOF.)

**Task line MUST include:**
- Actionable title (starts with a verb)
- Due-date emoji + YYYY-MM-DD
- Priority glyph (🔺 urgent / ⏫ high / 🔼 medium / 🔽 low)
- Tags: `#meeting/<short-title>`, `#person/<name>`, plus intent tags like `#intro`, `#followup`, `#email`, `#urgent`
- Circleback permalink to the source meeting

**Rules:**
- **Only your actions**: Circleback summaries often list "next steps" without clear ownership. Be skeptical — if the action could belong to the other person (e.g. "digest their own content", "check availability", "get back to us"), do NOT create a task. When in doubt, create a FOLLOW-UP task ("Ping X about Y") rather than an ownership task.
- **Capture commitments from others**: If the other person said they'd do something (e.g. "I'll get back in 2 days"), create a follow-up/ping task with the appropriate due date, not a task to do the thing yourself.
- **Specificity**: Every task MUST include specific name of person/company/project
- **Due dates**: If implied deadline, use the concrete date; otherwise default to today + 2 business days
- **Priority**: 🔺 urgent/time-sensitive, ⏫ promised deliverables, 🔼 general follow-ups, 🔽 normal

#### No split tasks for sequential steps (MANDATORY)
Never create separate tasks for steps that are part of the same workflow. If the action is "prepare X then send X" — that's ONE task, not two. Examples:
- ❌ "Build proposal" + "Send proposal" → ✅ "Build and send proposal"
- ❌ "Write draft" + "Send email" → ✅ "Draft and send email to X"
- ❌ "Review deck" + "Share deck" → ✅ "Review and share deck with X"

One task per intent. The user will naturally do the steps in order.

#### Todo dedup (MANDATORY)
Before appending a task, run a duplicate check against the current tasks file:
1. Normalize proposed title (lowercase, trim punctuation, collapse whitespace)
2. Parse every `- [ ]` line from `{user.obsidian_vault_path}/{user.obsidian_tasks_file}` and compare normalized task titles
3. Treat near-identical intro tasks as duplicates (e.g., "Intro David to Marcos" vs "Intro David (n8n) to Marcos Nils")
4. If duplicate exists: do NOT append a new task; if useful, update the existing line's trailing metadata (add the new meeting tag + Circleback link) via the `Edit` tool on that exact line
5. In output, report dedup decisions under `Skipped as duplicates:`

### 4. Draft follow-up emails
For ANY email that needs drafting (intros, follow-ups, VC replies, sending docs, etc.):
- **Read and follow `~/executive-assistant-skills/email-drafting/SKILL.md`** — it is the single source of truth for all drafting rules, style, templates, humanization, and delivery
- Identify draft triggers from meeting notes:
  - Promised intros or follow-ups
  - Promised docs/PDFs/decks
  - First call with a VC or new lead
  - Any promise to email someone
- When in doubt about whether to draft → DRAFT IT
- **Exception**: The proposal-only commitment rule below overrides this. If the commitment is to build a proposal first, do NOT draft — create an Obsidian task only.
- **When drafting intros to known contacts**: search sent Outlook mail for previous intros to them, use the same format, tone, and description
- Drafts are created in Outlook via `ms365.create-draft` (attachments via `ms365.add-attachment`)

#### Intro-specific hard requirements (MANDATORY)
If action items include intros, follow this exactly:
1. Create **one separate intro draft per intro pair** (never merge multiple intros into one generic follow-up).
2. Subject must be explicit: `Intro: <Person A> <> <Person B>`.
3. Body must include:
   - one-line who each person is,
   - one-line context for why this intro is happening,
   - close with `I'll let you two take it from here.` and `{user.signature}`.
4. **Never replace intro drafts with a generic recap email** like "Great meeting today" when intros were promised.
5. If recipient emails are known, create Outlook drafts immediately; if any email is missing, still generate the full draft text and report `MISSING_EMAIL: <name>`.
6. In the summary email, include an `Intro drafts created:` section listing each intro pair and whether it was drafted in Outlook or blocked by missing email.

#### First VC / first dealflow call follow-up (MANDATORY)
When the meeting is a FIRST call with a VC or dealflow company:
1. Create a first-meeting follow-up draft (unless blocked by proposal-only rule below).
2. Use meeting-specific context in the body (what was discussed, explicit next steps, concrete offers), not generic pleasantries.
3. Include your positioning (how you work) and attach `{user.workspace}/assets/Deck.pdf` (if present) via `ms365.add-attachment` when relevant.
4. Allowed to use "Great meeting today" only for this first-meeting follow-up class.

#### Proposal-only commitment rule (MANDATORY)
If you committed to "build a proposal" (or equivalent: proposal/deck/scope draft to prepare first):
- **Do NOT draft an outbound email yet**.
- Create a **single** Obsidian task to build AND send the proposal — e.g. "Build and send advisory proposal to Gabriel (BairesDev)".
- Do NOT create separate tasks for "build" and "send" — that's redundant. One task covers the full lifecycle.

### 5. Check if existing Obsidian tasks were fulfilled in today's meetings
After processing new action items, also check if any **existing open tasks** were addressed/completed during today's meetings:

1. Re-read `{user.obsidian_vault_path}/{user.obsidian_tasks_file}` and list every open `- [ ]` line.
2. For each meeting, check if the discussion covered or fulfilled any open task (e.g., "Share AI strategy with Colin" → discussed AI strategy directly with Colin in the call).
3. If a task was clearly fulfilled in the meeting, **complete it**: use the `Edit` tool to flip `- [ ]` → `- [x]` on that exact line and append `✅ <today YYYY-MM-DD>` at the end of the line.
4. Report completed tasks in the output: "✅ Completed: [task] — fulfilled during [meeting name]"

This ensures the Obsidian tasks list stays clean and reflects what actually happened.

## Error handling
- If `ms365.list-calendar-events` fails: log the error with full args + error (`python3 {user.workspace}/scripts/skill_log.py action-items ERROR "ms365 calendar failed" '{"args": "...", "error": "..."}'`), continue with the other account and/or Circleback-only.
- If Circleback returns auth errors: email `{user.notification_email}` with subject `[EA] ⚠️ Circleback auth expired` and body "Re-run the OAuth flow on the Hermes host." Then stop.
- If the vault path is missing or unwritable (wrong `obsidian_vault_path`, permission error): log the error, fall back to writing tasks to `{user.workspace}/state/pending-obsidian-tasks.md`, and email `{user.notification_email}` with subject `[EA] ⚠️ Obsidian vault unreachable` so you know to fix the path.
- If any step fails, continue with remaining steps when possible — don't abort the entire run for one failure.

## Deduplication (CRITICAL — prevents duplicate tasks)

### Meeting-level dedup
- **FIRST STEP before any processing**: Read `{user.workspace}/state/processed-meetings-YYYY-MM-DD.json` (if it exists — array of Circleback meeting IDs)
- Skip any meetings whose ID is already in that list — do NOT re-process them
- **Immediately after processing each meeting** (before moving to the next), append its Circleback meeting ID to the file. Do NOT wait until the end — write after each meeting to prevent races with other crons.
- This file is the single source of truth for "was this meeting already processed today?"

> **Note:** Titles are fragile for recurring meetings that share the same name. Always dedup by Circleback meeting ID.

### Task-level dedup
- Before appending ANY task, check BOTH open AND recently completed tasks for near-duplicates:
  1. Read `{user.obsidian_vault_path}/{user.obsidian_tasks_file}` once
  2. Parse open `- [ ]` lines AND completed `- [x]` lines whose `✅ YYYY-MM-DD` is today
  - Same person + same action intent = duplicate (e.g. "Text Morgane about Hank" and "Text Morgane (Braintrust) after Hank call")
  - Normalize: lowercase, strip parentheticals, collapse whitespace
  - If duplicate exists in EITHER open or today's-completed → SKIP, do not append
- Report skipped duplicates in output: "⏭️ Skipped (already exists): [task]"
- **Why check completed tasks**: The user may have already completed a task created by a post-meeting cron earlier today. Recreating it is wrong — the work is done.

### Why both layers matter
Post-meeting crons fire per-meeting. The daily end-of-day cron processes ALL meetings. Without meeting-level dedup, the same meeting gets processed twice. Without task-level dedup, even if the meeting is re-processed (e.g. file write failed), individual tasks won't be duplicated.

### Dedup is NON-NEGOTIABLE
If a meeting appears in `processed-meetings-YYYY-MM-DD.json`, do NOT process it again under any circumstances — even if you think the post-meeting cron "might have missed something." The post-meeting cron already handled it. If it had issues, the user will ask for a re-run manually.

## Output

Send ONE summary email via `ms365.send-mail`:

- `from` / `to`: `{user.notification_email}`
- `subject`: `[EA] Action Items — <YYYY-MM-DD> (<N> tasks, <M> drafts)`
- `body` sections:
  - **Tasks appended** → list each with its `#meeting/<short>` tag and due date
  - **Tasks completed (fulfilled in meetings)** → list with ✅
  - **Drafts composed** → recipient + intent + Outlook `webLink` for each draft
  - **Skipped as duplicates** → short list
- **Always name the specific meetings** processed (e.g. "from Fyxer call, Braintrust weekly, HGP Staff") — never say "from today's meetings"
- No meetings or no action items → do not send an email (NO_REPLY to the cron).

## Transcript cross-check (MANDATORY)
Circleback returns AI notes + a full transcript. Always cross-check action items against the transcript — not just the summary:

1. For each meeting, include `"transcript"` in the `include` list of `circleback.get_meeting`.
2. Scan the transcript for each candidate action item:
   - Is the item actually assigned to you, or to the other person?
   - Are there commitments with timelines that the summary missed (e.g. "I'll get back in 2 days")?
   - Are there action items the summary dropped entirely?
3. **Prefer the transcript** over the summary when they conflict.
4. If the transcript is absent (e.g. meeting wasn't recorded), proceed with the summary + skepticism rules.
