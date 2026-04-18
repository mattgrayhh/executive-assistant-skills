---
name: executive-digest
description: "Generate the daily executive digest — a single email summary of everything needing attention: stalled scheduling, pending intros, unanswered emails (Microsoft 365), promised follow-ups, open Asana tasks, and upcoming calendar events. Use when running the daily digest cron, or when user asks for a status digest, daily summary, \"what's pending\", or \"catch me up\"."
---
# Daily Executive Digest

## Config — read before starting
Read `../config/user.json` (resolves to `~/executive-assistant-skills/config/user.json`).
Extract and use throughout:
- `primary_email`, `work_email` — both Microsoft 365 accounts to check
- `notification_email` — inbox that receives the digest email
- `workspace` — absolute path to Hermes workspace
- `asana_workspace_gid` — Asana workspace to query
- `asana_default_project_gid` — (optional) default project to scope Asana queries; if empty, query across all projects in the workspace

Do not proceed until you have these values.

## Debug Logging (MANDATORY)
Read `../config/DEBUG_LOGGING.md` for the full convention. Use `python3 {user.workspace}/scripts/skill_log.py exec-digest <level> "<message>" ['<details>']` at every key step. Log BEFORE and AFTER every MCP call (ms365, asana, circleback). On any error, log the full tool args and error before continuing.

## Task source for this skill
The executive digest pulls pending work from **Microsoft 365 mail + Asana only**. It does NOT read the Obsidian tasks file — that's owned by the `action-items-obsidian` and `obsidian-due-drafts` skills. Asana is the authoritative task surface for the digest.

## Steps

### 1. Read rules and state
- Read `{user.workspace}/style/DIGEST_RULES.md` for format and rules
- Read state files:
  - `{user.workspace}/state/scheduling-threads.json` — stalled threads (>3 days since proposed)
  - `{user.workspace}/state/decisions-memory.json` — context on people/companies
  - `{user.workspace}/state/digest-state.json` — avoid repeating items

Schema: `{"lastRun": "ISO date", "surfacedItems": [{"id": "<conversation_id|task_gid>", "type": "intro|followup|draft|task|calendar", "surfacedAt": "ISO date"}]}`. Use consistent IDs: Outlook `conversationId` for email items, Asana task GID for tasks, Outlook event ID for calendar items. Do NOT use semantic strings like "calendar:golf-mar5" — always use the actual service ID. An item is "repeated" if its ID appeared in the last digest run. Re-surface only if its status changed since then.

### Error handling
If any data source (Outlook, Calendar, Asana, Circleback) fails or times out:
- Log the error, note it in the digest as "⚠️ [Source] unavailable — skipped"
- Continue with remaining sources — never halt the entire digest for one failure
- If BOTH Microsoft 365 accounts fail, abort and notify: "⚠️ Digest failed — Outlook unreachable"

### 2. Check Asana for pending tasks
```
asana.search_tasks --args '{"workspace": "{user.asana_workspace_gid}", "assignee": "me", "completed": false}'
asana.search_tasks --args '{"workspace": "{user.asana_workspace_gid}", "assignee": "me", "completed": false, "due_on.before": "<today>"}'
```

Include: overdue tasks, today's tasks, and any task with no due date that's been open >7 days (triage candidates).
Format as "📋 Asana — Open Tasks" section: task name, due date, project, priority (from task's `custom_fields` if present).
Highlight overdue first. Skip no-due-date tasks older than 30 days (likely stale — flag separately in a "Stale" subsection).

For each overdue or due-today task, include the Asana deep link using `permalink_url` from the task payload.

### 3. Check calendar (next 7 days) — BOTH accounts MANDATORY

```
ms365.list-calendar-events --args '{"user": "{user.primary_email}", "startDateTime": "<today>T00:00:00<tz>", "endDateTime": "<today+7>T00:00:00<tz>"}'
ms365.list-calendar-events --args '{"user": "{user.work_email}", "startDateTime": "<today>T00:00:00<tz>", "endDateTime": "<today+7>T00:00:00<tz>"}'
```

Use actual ISO8601 dates with the user's timezone offset (from `{user.timezone}`). Relative dates like 'today' and '+7 days' are not supported by the ms365 MCP.

**CRITICAL: You MUST run BOTH commands and merge ALL events from both calendars into a single timeline.** Events live on different calendars — showing only one gives an incomplete picture. Deduplicate by time+title if the same event appears on both. Look for OOO, travel, vacation blocks, back-to-back conflicts, and double-bookings across calendars.

**RSVP check:** For each upcoming meeting with external attendees, check the `attendees[].status.response` field. If NO attendee (other than the user) has `accepted`, flag it in the digest and suggest pinging them to confirm attendance. Frame as: "⚠️ [Meeting] — nobody accepted yet. Ping [names] to confirm?"

### 4. Check Outlook for pending items — BOTH accounts

**Universal rule (applies to ALL sub-steps below):** Before surfacing ANY email item, check the FULL conversation for replies from the user's team. If ANY team member already replied → skip the item. If a draft exists for an already-replied conversation → delete the draft silently (`ms365.delete-draft --args '{"user": "<acct>", "id": "<draftId>"}'`). This prevents surfacing stale items.

Run each query against BOTH accounts using `ms365.search-mail-messages`.

#### 4a. Pending intros and follow-ups
- Recent intros not actioned: `query: "subject:(intro OR introduction OR connecting) received>=<today-7d>"`
- Follow-ups from others: `query: "(following up OR checking in OR circling back) received>=<today-7d>"`
- Drafts awaiting send: `ms365.list-drafts --args '{"user": "<acct>"}'`

**Team-handled threads (MANDATORY check):**
Before flagging ANY intro, follow-up, or email item as "pending" or "no reply", check the FULL conversation for replies from the user's team. ANY reply from the following people counts as the item being handled:
- The user ({user.primary_email}, {user.work_email})
- Scheduling assistant ({user.scheduling_cc})
- Silent scheduling CC ({user.scheduling_silent_cc})
- Any configured additional team handlers in `{user.workspace}/style/DIGEST_RULES.md`

Fetch the full conversation:

```
ms365.get-conversation --args '{"user": "{user.work_email}", "conversationId": "<id>"}'
```

Scan ALL messages. If ANY message is from one of the above addresses, the item is handled — do NOT surface it.

**Previously resolved items (MANDATORY check):**
Before surfacing ANY item, check `{user.workspace}/state/digest-state.json` for the `resolvedItems` array. If the item's ID (conversation ID, Asana task GID, or item key) appears in `resolvedItems`, do NOT re-surface it. When the user tells you an item is "done" or "already handled", immediately add its ID to the `resolvedItems` array in the state file.

Schema for resolvedItems: `"resolvedItems": [{"id": "<conversationId|taskGid|itemKey>", "resolvedAt": "ISO date", "note": "brief reason"}]`

**Draft hygiene (MANDATORY):**
- For each draft candidate, check if a matching message (same conversation/subject intent) was already sent from that account.
- If already sent, delete the stale draft and exclude it from digest output.
- Only report drafts that still require a send/edit decision.

**Stale draft auto-cleanup (MANDATORY):**
- For EVERY draft found, fetch the full conversation (`ms365.get-conversation`) and check if the user already replied manually (sent message in the same conversation AFTER the draft was created).
- If the user already replied in the conversation → **delete the draft automatically** (`ms365.delete-draft --args '{"user": "<acct>", "id": "<draftId>"}'`) and do NOT surface it in the digest.
- This catches cases where auto-drafted replies become stale because the user replied on his own.

#### 4b. Unanswered emails from known contacts
Search BOTH Outlook accounts for recent inbound emails (last 7 days) from real people (not newsletters, automated, or system notifications) that have NO reply.
- If no reply exists after 24-48h, surface it as needing a decision (reply, ignore, or delegate)
- Prioritize: known contacts > first-time senders, VIPs always surface
- Frame as: "[Name] emailed about [topic] — reply, ignore, or delegate?"

#### 4c. Promised follow-ups not yet executed
Scan for commitments made but not yet completed:
- **From sent mail (last 14 days):** Look for promises like "I'll intro you", "I'll send the deck", "let me connect you with", "I'll follow up with" — then check if the intro/email was actually sent
- **From Asana:** Check open tasks tagged with follow-up intent (intros, send deck, ping someone) that are overdue or due today
- **From Circleback (last 7 days):** Query recent meetings for action items assigned to the user that haven't been completed:
  ```
  circleback.search_meetings --args '{"query": "action items {user.name}", "since": "<today-7d>T00:00:00Z"}'
  ```
  For each returned meeting:
  ```
  circleback.get_meeting --args '{"meeting_id": "<id>", "include": ["action_items", "transcript"]}'
  ```
  Cross-check each item against: (a) sent emails — was the intro/follow-up actually sent? (b) Asana — is there already an open task for it? (c) calendar — was the meeting/call already scheduled?
  Surface anything that fell through the cracks — promised but not yet actioned.
- Frame as: "Promised [action] to [person] on [date] — still pending"

**⚠️ MANDATORY sent-mail verification for EVERY promised follow-up (no exceptions):**
Before surfacing ANY item from this section, you MUST search sent mail on BOTH accounts for evidence it was already fulfilled:

```
ms365.search-mail-messages --args '{"user": "{user.primary_email}", "query": "to:<person_or_company>", "folder": "sentitems", "received": ">=<today-14d>"}'
ms365.search-mail-messages --args '{"user": "{user.work_email}", "query": "to:<person_or_company>", "folder": "sentitems", "received": ">=<today-14d>"}'
```

Also search by name/company if email address is unknown. Check the conversation content — a sent reply in the same conversation as the commitment means it was fulfilled. If you find a sent email that addresses the commitment → do NOT surface it. Log: `python3 {user.workspace}/scripts/skill_log.py exec-digest DEBUG "Follow-up already sent" '{"person": "<name>", "conversation": "<id>"}'`

This is the #1 source of false positives in the digest. Never skip this check.

### Additional checks (per DIGEST_RULES.md)
For sections not explicitly covered above, follow `{user.workspace}/style/DIGEST_RULES.md`:
- OOO conflict detection (§2)
- Action-required non-urgent emails — billing, contracts, renewals (§7)
- Decision memory integration — read and apply `{user.workspace}/state/decisions-memory.json` for context on people/companies

### 5. Compile and send
- Format per `{user.workspace}/style/DIGEST_RULES.md`
- If items exist → send ONE email via `ms365.send-mail`:
  - `from` / `to`: `{user.notification_email}`
  - `subject`: `[EA] Executive Digest — <YYYY-MM-DD>`
  - `body`: the full digest markdown, organized by section (Asana open tasks, upcoming calendar, pending intros/follow-ups, unanswered inbound, promised follow-ups still open). Use clear `## section` headings so you can skim on mobile Outlook.
- Update `{user.workspace}/state/digest-state.json` with items surfaced
- Nothing needs attention:
  - **Cron context**: NO_REPLY (don't send anything)
  - **User asked for digest** (interactive): respond in the terminal with "Nothing needs attention today ✅" (no email needed in interactive mode)
