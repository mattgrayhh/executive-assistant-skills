---
name: obsidian-due-drafts
description: Check your Obsidian tasks file for tasks due today (and overdue) that involve pinging, emailing, or following up with someone. Auto-draft the emails in Outlook using meeting context and notify via WhatsApp. Use when running the daily due-drafts cron, or when user asks to process email tasks from Obsidian.
---
# Obsidian Due-Today Email Drafts

## Config — read before starting
Read `../config/user.json` (resolves to `~/executive-assistant-skills/config/user.json`).
Extract and use throughout:
- `primary_email`, `work_email` — Microsoft 365 accounts
- `whatsapp` — for notification delivery
- `workspace` — absolute path to Hermes workspace
- `signature` — email signature
- `obsidian_tasks_file` — vault-relative path to your tasks file

## Debug Logging (MANDATORY)
Read `../config/DEBUG_LOGGING.md` for the full convention. Use `python3 {user.workspace}/scripts/skill_log.py obsidian-due-drafts <level> "<message>" ['<details>']` at every key step. Log BEFORE and AFTER every MCP call (obsidian, ms365, circleback). On any error, log the full tool args and error before continuing.

## Steps

### 1. Read today's due tasks from Obsidian
```
obsidian.obsidian_read_file --args '{"filepath": "{user.obsidian_tasks_file}"}'
```

Parse every open task line (`- [ ]`). A task counts as "due today or overdue" if:
- It has a 📅 `YYYY-MM-DD` glyph with a date <= today in the user's timezone, OR
- It has a 🔺 (urgent) priority glyph AND no due date (treat urgent as "do today")

### 2. Identify email/ping tasks
Filter for tasks whose content matches outreach intent:
- Tags: `#email`, `#intro`, `#followup`, `#ping`, `#send`
- Keywords in the title: `ping`, `email`, `follow up`, `follow-up`, `send`, `reach out`, `text`, `message`, `intro`, `connect`, `check in`, `nudge`, `reply`, `respond`, `draft`
- Pattern: any task that implies sending a communication to a specific person

Skip tasks that are clearly NOT outreach (e.g. "review doc", "read report", "build proposal").

### 3. For each outreach task, draft the email
Read and follow `~/executive-assistant-skills/email-drafting/SKILL.md` for all drafting rules.

For each task:
1. **Identify the recipient**: Extract person/company name from task content + its tags/trailing metadata (e.g. `#person/david-aronchick`)
2. **Pull meeting context (if available)**: If the task line has a `[link](<circleback-url>)` or a `#meeting/<short>` tag, retrieve the meeting from Circleback. If no meeting reference exists in the task, skip directly to step 3 — draft from email history and task content alone.
   ```
   circleback.search_meetings --args '{"query": "<meeting short title or attendee name>", "limit": 3}'
   circleback.get_meeting --args '{"meeting_id": "<id>", "include": ["transcript", "summary"]}'
   ```
   When meeting context is available, the email draft must reflect what was actually said — not just the task title. Look for: specific commitments, timelines discussed, names/projects mentioned, tone of the conversation, and any docs/links promised.
3. **Search email history**: Find the latest thread with this person across both Microsoft 365 accounts to get context, their email address, and the right account to reply from.
   ```
   ms365.search-mail-messages --args '{"user": "{user.primary_email}", "query": "from:<addr> OR to:<addr>", "top": 5}'
   ms365.search-mail-messages --args '{"user": "{user.work_email}", "query": "from:<addr> OR to:<addr>", "top": 5}'
   ```
   If no prior thread exists with this recipient, default to `{user.work_email}` for professional/business contacts or `{user.primary_email}` for personal contacts. When ambiguous, use `{user.work_email}`.
4. **Determine email type**: follow-up, ping/check-in, intro, send-doc, etc.
5. **Draft the email**: Create an Outlook draft on the correct account via `ms365.create-draft`. If it's a reply, include the `conversationId` / `replyTo` reference so it threads correctly.
6. **Use all context together**: Combine the Circleback transcript (what was actually discussed), task description, and email history to write a specific, contextual draft. Never write generic follow-ups — reference concrete topics from the conversation.

#### Draft rules
- Mirror the language of the existing thread (English or Spanish)
- Keep it short — these are follow-ups and pings, not essays
- Sign with `{user.signature}`
- If the task says "ping" or "check in" — write a brief, friendly nudge
- If the task says "send [thing]" — draft the email with the attachment reference (use `ms365.add-attachment` if the file path is known)
- If the task says "intro" — follow intro format from email-drafting skill
- If recipient email can't be found — report it, don't skip silently

### 4. Notify via WhatsApp
Send a single WhatsApp message to {user.whatsapp} with:

```
📬 *Due-today drafts*

<N> email drafts created from today's Obsidian tasks:

1. **<recipient>** — <one-line intent> (draft in <account>)
2. ...

Review and send when ready.
```

If tasks exist but none are outreach-type, report:
```
📋 *Due-today tasks (no drafts needed)*

<list of today's tasks — quick reference>
```

If no tasks due today → NO_REPLY (skip notification entirely).

### 5. Infrastructure
```bash
python3 {user.workspace}/scripts/cron_canary.py ping obsidian-due-drafts
```

## Error handling
- If Obsidian MCP fails: notify via WhatsApp "⚠️ Obsidian due-drafts failed: <error>", ping canary, exit.
- If Circleback lookup fails for a task: continue without meeting context — draft from email history + task content only.
- If Outlook draft creation fails for one task: continue with remaining tasks, report failures in the notification.
- If the Obsidian REST plugin is unreachable (app closed): report "⚠️ Obsidian not running — start Obsidian to allow due-drafts" and exit.

## Rules
- Don't check off the tasks — just draft and notify. User decides when to send and ticks the checkbox manually (or via the email-drafting skill's "after send → complete task" rule).
- Before drafting, check if the user already replied: `ms365.search-mail-messages --args '{"user": "<account>", "query": "to:<recipient> in:sent", "top": 5}'` and filter by recent (last 14 days). If recent sent mail exists in the same conversation, skip drafting and note "already replied" in the notification.
- Check for existing drafts before creating: `ms365.list-drafts --args '{"user": "<account>"}'`. If a draft already exists for the same conversationId or recipient + subject, skip and note "draft already exists."
- Overdue outreach tasks get a ⚠️ prefix in the notification.
