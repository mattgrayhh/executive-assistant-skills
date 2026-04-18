# executive-assistant-skills

[Hermes Agent](https://hermes-agent.nousresearch.com/docs/) skills that replace a human executive assistant.

After setting up these skills with Hermes, I let go of my human EA and replaced her entirely with my Hermes agent. These five skills handle the core of what an executive assistant does: prepping for meetings, following up on action items, drafting emails, and keeping you on top of everything.

## What's included

| Skill | What it replaces |
|-------|-----------------|
| `meeting-prep` | EA researching attendees, pulling email history, and briefing you before each call |
| `action-items-obsidian` | EA reviewing meeting notes, capturing follow-up tasks in Obsidian, and drafting emails you promised to send |
| `email-drafting` | EA drafting replies, intro emails, scheduling responses, and thank-you notes in your voice |
| `executive-digest` | EA giving you a morning status update: stalled threads, pending intros, overdue Asana tasks, calendar conflicts |
| `obsidian-due-drafts` | EA checking your Obsidian task list each morning, drafting follow-up/ping emails for anything due today, and notifying you to review |
| `humanizer` | Making sure nothing your agent writes sounds like AI wrote it |

## How it works

Each skill is a markdown file (`SKILL.md`) that tells Hermes exactly how to do the job. Hermes reads the skill, follows the instructions, and delivers results by emailing you directly via Microsoft 365 (so you get everything in your inbox).

Skills run on cron schedules — meeting prep fires before your first meeting, action items run after your last meeting, and the digest hits every morning. You can also trigger any skill manually by asking Hermes (e.g. `/meeting-prep`) or in natural language.

All personal config (email accounts, timezone, work schedule, etc.) lives in a single `config/user.json` that's gitignored and never committed.

## Prerequisites

- [Hermes Agent](https://hermes-agent.nousresearch.com/docs/) running (local or server)
- Two Microsoft 365 accounts connected via the [ms-365 MCP server](https://github.com/softeria/ms-365-mcp-server)
- [Circleback](https://circleback.ai/) as your AI notetaker (via the [Circleback MCP](https://support.circleback.ai/en/articles/13249081-circleback-mcp))
- [Obsidian](https://obsidian.md/) installed locally — your vault is just a folder of markdown files; Hermes reads and writes it directly via filesystem tools (no MCP, no plugin required)
- [Asana](https://asana.com/) connected via the [official Asana MCP v2](https://developers.asana.com/docs/integrating-with-asanas-mcp-server) (`https://mcp.asana.com/v2/mcp`)
- A `style/` directory in your Hermes workspace with email style guides (see `docs/setup.md`)

---

## Quick setup

### 1. Clone the repo

```bash
git clone https://github.com/mgonto/executive-assistant-skills.git ~/executive-assistant-skills
```

### 2. Create your config

```bash
cp ~/executive-assistant-skills/config/user.example.json ~/executive-assistant-skills/config/user.json
# Edit user.json with your values — it's gitignored
```

### 3. Tell Hermes to load these skills

Symlink each skill directory into `~/.hermes/skills/`:

```bash
mkdir -p ~/.hermes/skills
for d in meeting-prep action-items-obsidian email-drafting executive-digest obsidian-due-drafts humanizer; do
  ln -s ~/executive-assistant-skills/$d ~/.hermes/skills/$d
done
```

Hermes auto-discovers any `SKILL.md` under `~/.hermes/skills/` on the next start.

### 4. Start the gateway

```bash
hermes gateway start
```

### 5. Set up crons

See `docs/crons.md` for ready-to-paste cron configs for `~/.hermes/cron/`.

---

## Config fields

See `config/user.example.json` for the full template:

| Field | Example | Used for |
|-------|---------|----------|
| `name` | `"YourName"` | Meeting transcript queries, task attribution |
| `full_name` | `"Your Full Name"` | Exact-match attribution in meeting notes |
| `primary_email` | `"you@outlook.com"` | Microsoft 365 account 1 |
| `work_email` | `"you@company.com"` | Microsoft 365 account 2 |
| `notification_email` | `"you@outlook.com"` | Inbox that receives all EA output emails (usually same as `primary_email`) |
| `timezone` | `"America/New_York"` | Meeting times, cron scheduling |
| `scheduling_cc` | `"assistant@company.com"` | CC on scheduling emails |
| `scheduling_silent_cc` | `"colleague@company.com"` | Silent CC (not mentioned in body) |
| `signature` | `"--yourname"` | Email sign-off |
| `workspace` | `"/home/user/.hermes"` | Absolute path to your Hermes workspace |
| `obsidian_vault_path` | `"/home/user/Obsidian/MyVault"` | Local vault root (filesystem path) |
| `obsidian_tasks_file` | `"Tasks.md"` | Vault-relative path where EA tasks live |
| `asana_workspace_gid` | `"1199999999999999"` | Asana workspace ID surfaced in the digest |

---

## Full setup guide

- `docs/setup.md` — complete setup (Hermes, M365 MCP, Circleback MCP, Asana MCP, local Obsidian vault)
- `docs/crons.md` — cron job templates for all skills

---

## Repository conventions

- `config/user.json` is **gitignored** — each person creates their own
- `config/user.example.json` is the committed template
- `state/` and `logs/` are gitignored (machine-local)
- Skills reference workspace files (`style/`, `state/`, `scripts/`) via `{user.workspace}/` prefix for portability
