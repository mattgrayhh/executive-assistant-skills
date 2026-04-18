# Cron Jobs

Configure these cron jobs in Hermes after completing `docs/setup.md`.

Hermes reads cron entries from `~/.hermes/cron/` (one YAML file per job). You can also use `hermes cron add` if your build exposes that subcommand.

> **Tip:** Set all times in your local timezone using the `tz` field. Adjust to match your work schedule.

---

## 1. Daily Meeting Prep

**When:** 8:30 AM on work days (before meetings start)
**Skill:** `meeting-prep`

`~/.hermes/cron/meeting-prep.yaml`:

```yaml
name: Daily Meeting Prep
schedule:
  kind: cron
  expr: "30 8 * * 1-5"
  tz: America/New_York
sessionTarget: isolated
payload:
  kind: agentTurn
  thinking: high
  timeoutSeconds: 1200
  model: opus
  message: |
    Read ~/executive-assistant-skills/meeting-prep/SKILL.md and run daily meeting prep for today.
delivery:
  mode: none
```

> Adjust `expr` and `tz` for your work days and timezone.
> 1200s timeout (20 min) — needed for multi-meeting days with full research.

---

## 2. Daily Action Items → Obsidian

**When:** 6:00 PM on work days (after all meetings end)
**Skill:** `action-items-obsidian`

`~/.hermes/cron/action-items.yaml`:

```yaml
name: Daily Action Items → Obsidian
schedule:
  kind: cron
  expr: "0 18 * * 1-5"
  tz: America/New_York
sessionTarget: isolated
payload:
  kind: agentTurn
  thinking: high
  timeoutSeconds: 1200
  model: opus
  message: |
    Read ~/executive-assistant-skills/action-items-obsidian/SKILL.md and extract action items
    from today's Circleback meetings into Obsidian. Draft follow-up emails as needed.
delivery:
  mode: none   # the skill emails you the result via ms365.send-mail
```

---

## 3. Daily Executive Digest

**When:** 9:00 AM on weekdays
**Skill:** `executive-digest`

`~/.hermes/cron/executive-digest.yaml`:

```yaml
name: Daily Executive Digest
schedule:
  kind: cron
  expr: "0 9 * * 1-5"
  tz: America/New_York
sessionTarget: isolated
payload:
  kind: agentTurn
  thinking: high
  timeoutSeconds: 600
  model: opus
  message: |
    Read ~/executive-assistant-skills/executive-digest/SKILL.md and generate the daily
    executive digest from Microsoft 365 (mail + calendar) and Asana tasks.
delivery:
  mode: none   # the skill emails you the result via ms365.send-mail
```

---

## 4. Obsidian Due-Today Email Drafts

**When:** 7:00 AM on work days (before the day starts)
**Skill:** `obsidian-due-drafts`

Checks your Obsidian tasks file for tasks due today (and overdue) that involve pinging, emailing, or following up with someone. Auto-drafts the emails in the correct Outlook account/thread and emails you a summary via M365 listing what was drafted. Doesn't send anything — just creates drafts for review.

`~/.hermes/cron/obsidian-due-drafts.yaml`:

```yaml
name: Obsidian Due-Today Email Drafts
schedule:
  kind: cron
  expr: "0 7 * * 1-5"
  tz: America/Argentina/Buenos_Aires
sessionTarget: isolated
payload:
  kind: agentTurn
  thinking: high
  timeoutSeconds: 600
  model: sonnet
  message: |
    Read ~/executive-assistant-skills/obsidian-due-drafts/SKILL.md and process today's
    due tasks. Draft emails for any outreach/ping/follow-up tasks and email the
    summary to {user.notification_email}.

    After done: python3 ~/.hermes/scripts/cron_canary.py ping obsidian-due-drafts
delivery:
  mode: none   # the skill emails you the result via ms365.send-mail
```

---

## Notes

- **`sessionTarget: isolated`** is mandatory for all crons — shared sessions remember previous runs and may skip work thinking it's already done.
- **`model: opus`** for meeting-prep, action-items, and digest — these require deep reasoning and extraction quality. Use `sonnet` only for lightweight tasks (token refresh, health checks).
- **`thinking: high`** improves extraction and classification quality.
- All skill paths use `~/executive-assistant-skills/` absolute references so they work correctly from isolated cron sessions regardless of working directory.
- **Delivery is always email** (`ms365.send-mail` from `{user.notification_email}` to `{user.notification_email}`, subject prefixed `[EA] …`). `delivery.mode: none` at the cron level is intentional — the skill handles its own delivery so it can produce rich markdown bodies and add subject-line context.
- MCP OAuth tokens (Circleback, Asana, M365) refresh automatically through Hermes' native MCP client — no manual refresh crons needed, unlike the old mcporter flow.
