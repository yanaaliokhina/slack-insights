# slack-insights

Fetches Slack messages from specified channels, uses Claude AI to categorize and summarize them, and creates a new Google Doc digest in a configured Drive folder.

No Slack bot token, Anthropic API key, or Google OAuth credentials needed — Claude Code handles all service authentication via MCP connections.

## How it works

1. Edit `config.yaml` with your channels, mentions, and start date.
2. Open this project in Claude Code and run `/digest`.
3. Claude orchestrates three subagents: one **collector** per channel (2 in parallel) → **analyzer** → **writer**.

```
📅 May 20 – May 27, 2026
────────────────────────────────────────────────

COMPANY UPDATES
Leadership shared Q3 strategy and an engineering reorg announcement.
• @ceo in #general: Q3 focus is enterprise expansion — full memo in Notion.
• @hr in #announcements: Engineering reorg effective June 1.

INCIDENTS & ISSUES
One P1 incident this week, resolved in 4 hours.
• P1 on May 22 in #incidents — DB connection pool exhaustion. Post-mortem May 29.

ACTION ITEMS
• @john in #engineering: All backend engineers review API rate limits doc by EOW.

⭐ HIGHLIGHTS — YOU WERE MENTIONED
• @sarah in #engineering: "@yana can you review the auth PR before Friday?"
• @mike in #general: "Great work on the dashboard launch, @yana!"
```

---

## Setup

### 1. Connect MCP servers in Claude Code

Two MCP connections must be active:

- **Slack** — for fetching messages (provides `mcp__claude_ai_Slack__*` tools)
- **Google Drive** — for writing to the doc (restart Claude Code after connecting)

### 2. Configure `config.yaml`

```yaml
slack:
  channels:
    - general
    - engineering
    - incidents
  mentions:
    - U012AB3CD   # your Slack user ID (Profile → ⋮ More → Copy member ID)
    - "@yana"     # your display name as it appears in Slack
  start_date: "2026-05-20"   # YYYY-MM-DD, or leave blank for last 7 days

output:
  max_categories: 10
  google_drive_folder_id: "your_folder_id_here"   # from drive.google.com/drive/folders/<id>
```

### 3. Set the Google Drive folder

Create (or pick) a Google Drive folder where digest docs will be created. Copy its ID from the folder URL (`drive.google.com/drive/folders/<id>`) and set it as `output.google_drive_folder_id` in `config.yaml`.

---

## Usage

Open the project in Claude Code and run:

```
/digest
```

### What happens

```
[digest] config loaded — channels: general, engineering, incidents — date: 2026-05-20 to 2026-05-31
[digest] collection queue: 3 channel(s) to fetch, 0 already cached
[digest] launching collectors for batch: #general, #engineering (batch 1 of 2)
[collector] #general: starting — fetching 2026-05-20 to 2026-05-31
[collector] #engineering: starting — fetching 2026-05-20 to 2026-05-31
[collector] #general: 24 messages saved so far...
[collector] #engineering: 41 messages saved so far...
[collector] #general: done — 24 messages → data/general.json
[collector] #engineering: done — 57 messages → data/engineering.json
[digest] launching collectors for batch: #incidents (batch 2 of 2)
[collector] #incidents: done — 12 messages → data/incidents.json
[digest] all channels collected
[digest] starting analyzer...
[analyzer] analyzing 93 messages across 3 channels...
[analyzer] done — summary written
[digest] starting writer...
[writer] created "Slack Digest – May 20 – May 31, 2026"
[digest] cleaned up local data files
```

### Resuming a partial run

Each stage is skipped automatically if its output already exists in `data/`:
- Per-channel files (`data/<channel>.json`) skip re-fetching that channel.
- `data/summary.json` skips re-analyzing.

Delete a file to re-run that stage.

To change the digest format or categorization rules, edit `format_template.md`.

---

## Troubleshooting

| Problem | Fix |
|---|---|
| Slack MCP tools not available | Check MCP connection is active in Claude Code settings |
| Google Drive MCP tools not available | Connect Google Drive MCP and restart Claude Code |
| No messages / empty categories | Check date range and that your Slack account can read the channels |
| Agent exceeds max iterations | Reduce the number of channels or narrow the date range |
| Wrong digest format | Edit `format_template.md` |
| Writer fails to create file | Verify `mcp__claude_ai_Google_Drive__create_file` is listed in `.claude/agents/writer.md` tools |
# slack-insights
