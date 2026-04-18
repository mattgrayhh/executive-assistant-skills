# Debug Logging Convention

All executive assistant skills MUST log key steps for post-mortem debugging.

## Tool
```bash
python3 {user.workspace}/scripts/skill_log.py <skill_name> <level> "<message>" ['<details_json>']
```

## Levels
- **DEBUG**: Intermediate data (meeting lists, search results, task counts)
- **INFO**: Key milestones (step started, step completed, results summary)
- **WARN**: Recoverable issues (one source failed, falling back)
- **ERROR**: Failures that affect output (auth expired, command syntax error, no results)

## Mandatory Log Points
Every skill MUST log at minimum:
1. **Start**: `INFO "Starting <skill_name> run"`
2. **Config loaded**: `DEBUG "Config loaded" '{"user": "<name>", "accounts": [...]}'`
3. **Before each MCP call** (ms-365, obsidian, circleback, asana): `DEBUG "Calling <server>.<tool>" '{"args": {...}}'`
4. **After each MCP call**: `DEBUG "Result from <server>.<tool>" '{"status": "ok|error", "count": N}'` or `ERROR` if it failed
5. **Key decisions**: `INFO "Skipping meeting X (already processed)"` or `INFO "Drafting email for Y"`
6. **End**: `INFO "Completed <skill_name> run" '{"tasks_created": N, "drafts": N, "errors": N}'`

## Log Location
`{user.workspace}/logs/skills/<skill_name>-YYYY-MM-DD.log` (JSONL) — maps to `~/.hermes/logs/skills/` by default.

## Reviewing Logs
```bash
# Today's logs for a skill
cat ~/.hermes/logs/skills/action-items-obsidian-$(date +%Y-%m-%d).log | python3 -m json.tool

# Errors only
grep '"level": "ERROR"' ~/.hermes/logs/skills/action-items-obsidian-*.log

# Last run
tail -20 ~/.hermes/logs/skills/action-items-obsidian-$(date +%Y-%m-%d).log
```
