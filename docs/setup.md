# Setup Guide

Complete setup for a new machine. Follow in order.

## 1. Clone the repo

```bash
git clone https://github.com/mgonto/executive-assistant-skills.git ~/executive-assistant-skills
```

> The repo must live at `~/executive-assistant-skills/`. Skills reference config at `../config/user.json` which resolves relative to each skill's location.

## 2. Create your user config

```bash
cp ~/executive-assistant-skills/config/user.example.json ~/executive-assistant-skills/config/user.json
```

Edit `user.json` with your personal values — it's gitignored and never committed.

Fields to fill in:
- `name` / `full_name` — your name (used in Circleback queries)
- `timezone` — IANA timezone string (e.g. `America/Argentina/Buenos_Aires`)
- `work_days` — your working days (e.g. `["Monday", "Wednesday"]`)
- `availability_window` — hours available for meetings (e.g. `"15:00-19:00"`)
- `primary_email` — personal Microsoft 365 account
- `work_email` — work Microsoft 365 account
- `notification_email` — inbox where EA output emails land (usually the same as `primary_email`)
- `scheduling_cc` — scheduling assistant email (CC'd on all scheduling emails)
- `scheduling_silent_cc` — silent CC for scheduling visibility (never mentioned in email body)
- `calendar_id` — usually `"primary"`
- `signature` — email sign-off (e.g. `"--yourname"`)
- `workspace` — absolute path to your Hermes workspace (e.g. `/home/youruser/.hermes`)
- `obsidian_vault_path` — absolute path to your local Obsidian vault root (a plain folder of markdown files)
- `obsidian_tasks_file` — vault-relative path where EA tasks live (e.g. `Tasks.md`)
- `obsidian_meeting_notes_dir` — vault-relative folder for meeting briefs (e.g. `Meetings`)
- `asana_workspace_gid` — Asana workspace ID the digest should pull from
- `asana_default_project_gid` — optional default project GID (leave empty to search across all projects)

## 3. Install Hermes Agent

Follow the official install guide: https://hermes-agent.nousresearch.com/docs/quickstart/installation

Then initialize the workspace:

```bash
mkdir -p ~/.hermes
cp cli-config.yaml.example ~/.hermes/config.yaml
touch ~/.hermes/.env
```

## 4. Load these skills into Hermes

Hermes auto-discovers any `SKILL.md` under `~/.hermes/skills/`. Symlink each skill directory:

```bash
mkdir -p ~/.hermes/skills
for d in meeting-prep action-items-obsidian email-drafting executive-digest obsidian-due-drafts humanizer; do
  ln -s ~/executive-assistant-skills/$d ~/.hermes/skills/$d
done
```

Verify the skills are discoverable:

```bash
hermes skills browse
```

Then start the gateway:

```bash
hermes gateway start
```

(You do NOT need `hermes gateway setup` for messaging platforms — these skills deliver all output via Microsoft 365 email, not WhatsApp/Slack/Telegram. Skip the gateway messaging setup unless you want it for unrelated skills.)

## 5. Configure MCP servers in `~/.hermes/config.yaml`

Hermes has a first-party native MCP client. Add these servers to the `mcp_servers` section of `config.yaml`:

```yaml
mcp_servers:
  ms365:
    transport: stdio
    command: npx
    args: ["-y", "@softeria/ms-365-mcp-server"]
    env:
      MS365_MCP_CLIENT_ID: "${MS365_MCP_CLIENT_ID}"

  circleback:
    transport: http
    url: "https://app.circleback.ai/api/mcp"

  asana:
    transport: http
    url: "https://mcp.asana.com/v2/mcp"
```

Put secrets in `~/.hermes/.env`:

```bash
MS365_MCP_CLIENT_ID=<your Azure AD app registration client ID>
```

Obsidian needs no MCP — the skills read and write the vault directly as markdown files. Just make sure the path in `obsidian_vault_path` points at the folder Obsidian treats as the vault root.

Hermes reloads MCP servers on gateway restart.

## 6. Build your email style guide

The `email-drafting` skill writes emails in your voice — but it needs to learn your voice first. You do this by analyzing your sent emails and creating a style guide.

### Generate the style files

Ask Hermes:

> "Read my last 200 sent emails from both Microsoft 365 accounts (use the ms365 MCP) and create an email writing style guide. Analyze: sentence length, greeting patterns, sign-off patterns, vocabulary, tone, punctuation habits, and common phrases. Write it to `~/.hermes/style/EMAIL_STYLE.md` using filesystem tools."

Then:

> "Now create `~/.hermes/style/EMAIL_TEMPLATES.md` with template patterns for the email types I send most: intros, follow-ups, scheduling, thank-you notes, VC/investor replies. Base each template on real examples from my sent mail."

Finally, create an empty feedback log:

```bash
mkdir -p ~/.hermes/style
touch ~/.hermes/style/FEEDBACK_LOG.md
```

This is where corrections accumulate over time. Every time you tell Hermes "that draft was too formal" or "I wouldn't say it that way," it logs the feedback. Latest corrections always override the original style guide.

### What you end up with

```
~/.hermes/style/
├── EMAIL_STYLE.md          # Your writing voice (auto-generated from sent mail)
├── EMAIL_TEMPLATES.md      # Template patterns for common email types
├── FEEDBACK_LOG.md         # Running log of your corrections (starts empty)
├── DIGEST_RULES.md         # Format rules for the executive digest
└── MEETING_PREP_RULES.md   # Additional research steps for meeting prep
```

The digest and meeting prep rules files are optional — create them if you want to customize the output format beyond the defaults in the skill files.

## 7. Keep USER.md in sync

Your Hermes workspace should have a `USER.md` with the same info as `user.json` in human-readable form. Skills read `user.json` programmatically; `USER.md` provides context to the agent in every session. Keep both in sync when you update one.

## 8. Authenticate Microsoft 365

On first invocation, the ms365 MCP starts an OAuth device-code flow. Trigger it once with any call:

```bash
hermes tools call ms365.list-mail-folders --args '{}'
```

Follow the device-code URL, sign in with your primary account, approve the scopes. Tokens land in the MCP's auth store. Repeat for the work account by switching the active profile (see ms365 MCP README).

## 9. Authenticate Circleback

```bash
hermes tools call circleback.list_meetings --args '{"limit": 1}'
```

Circleback uses OAuth with dynamic client registration. Follow the auth URL printed to the terminal. Tokens are stored by Hermes.

## 10. Verify the Obsidian vault

No auth, no plugin. Confirm that `obsidian_vault_path` in `user.json` points at the folder Obsidian opens as a vault (it should contain a `.obsidian/` subdirectory):

```bash
ls "$(python3 -c 'import json; print(json.load(open("'$HOME'/executive-assistant-skills/config/user.json"))["obsidian_vault_path"])')/.obsidian"
```

If that prints plugin/settings files, you're set. The skills read/write markdown files in the vault directly. You do NOT need Obsidian to be running for the skills to work — Obsidian will pick up the changes on next open.

## 11. Authenticate Asana

```bash
hermes tools call asana.search_tasks --args '{"workspace": "<your workspace gid>", "text": "test"}'
```

Asana MCP v2 uses OAuth; tokens expire every hour but refresh automatically. Follow the auth URL on first call.

## 12. Verify everything works

```bash
hermes tools call ms365.list-mail-messages --args '{"top": 1}'
hermes tools call circleback.list_meetings --args '{"limit": 1}'
cat "<obsidian_vault_path>/Tasks.md"   # local file read, no MCP
hermes tools call asana.search_tasks --args '{"workspace": "<gid>", "text": "test"}'
```

Then set up cron jobs per `docs/crons.md`.

### Skill-specific notes

- **obsidian-due-drafts** needs ms365 MCP (step 8) and read access to `obsidian_vault_path/obsidian_tasks_file` on disk. It reads tasks, searches Outlook for existing threads, and creates drafts. It also depends on the `email-drafting` skill's style guide (step 6).
- **executive-digest** reads from ms365 (mail + calendar) and Asana (tasks). Obsidian is not queried by the digest — Asana is the task source.
- **action-items-obsidian** writes tasks directly into your vault on disk (`obsidian_vault_path/obsidian_tasks_file`). Obsidian will pick up the changes the next time the app is focused. If you prefer daily-note capture, change that field to a date-stamped path and update the skill accordingly.
