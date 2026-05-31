# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## How to run a digest

Use the `/digest` slash command — it is the orchestrator for this project. It reads `config.yaml` and runs the full pipeline, skipping any stage whose output already exists in `data/`.

Typical flow:
1. Read `config.yaml` to get channels, date range, mentions, and max_categories.
2. Spawn one **collector** agent per channel, running at most 2 collectors in parallel (to avoid Slack API rate limits). Each collector writes its own `data/<channel_name>.json` and saves messages in chunks as they arrive.
3. Spawn the **analyzer** agent → reads all per-channel files, merges them, produces `data/summary.json`.
4. Spawn the **writer** agent → creates a new Google Doc in the configured Drive folder.
5. Orchestrator deletes local intermediate files (`data/*.json`) after the doc is confirmed created.

The command definition lives in `.claude/commands/digest.md`. Edit it to change pipeline behavior.

## Architecture

Three Claude subagents in `.claude/agents/`, invoked via the Agent tool. No Slack bot token, Anthropic API key, or Google OAuth credentials are needed — Claude Code manages all service authentication via MCP connections.

```
.claude/agents/collector.md   Slack MCP → data/<channel_name>.json  (one agent per channel)
.claude/agents/analyzer.md   Read/Write → data/summary.json
.claude/agents/writer.md     Google Drive MCP → new Google Doc in configured folder

format_template.md            Categorization rules and output JSON schema for the analyzer (user-editable)
config.yaml                   Channels, date range, mentions, max_categories
data/                         Intermediate files (gitignored)
```

## Agents

### collector
Fetches one channel per invocation. Saves messages to `data/<channel_name>.json`, writing the accumulated list to disk after each page so partial results are preserved if a run is interrupted. The orchestrator spawns collectors in parallel batches of 2.

### analyzer
Reads all per-channel `data/<channel>.json` files listed in its task prompt, merges them into a combined dataset, then produces `data/summary.json` containing `{date_range, categories[], highlights[]}`. To change categorization rules or the output JSON schema, edit `format_template.md`.

### writer
Reads `data/summary.json` and `google_drive_folder_id` from its task prompt, then creates a new Google Doc named `Slack Digest – <date_range>` in the configured folder using HTML (`text/html`) so Google Drive renders headings and lists correctly. To change the Google Doc layout or formatting, edit `.claude/agents/writer.md`. Cleanup of local files is handled by the orchestrator after the writer confirms success.

## Configuration (`config.yaml`)

| Key | Default | Purpose |
|---|---|---|
| `slack.channels` | required | Channel names to fetch (without `#`) |
| `slack.mentions` | `[]` | User IDs/names for the Highlights section |
| `slack.start_date` | last 7 days | Start of fetch window (YYYY-MM-DD); end is always today |
| `output.max_categories` | `10` | Max categories the Analyzer may create |
| `output.google_drive_folder_id` | required | Drive folder where digest files are created |

## Prerequisites

- **Slack MCP** (`mcp__claude_ai_Slack__*`) must be connected — for the collector agents.
- **Google Drive MCP** must be connected — for the writer agent (restart Claude Code after connecting).
- Set `output.google_drive_folder_id` in `config.yaml` to the target folder (get it from the folder URL: `drive.google.com/drive/folders/<id>`).

## Intermediate file schema

Each per-channel file `data/<channel_name>.json`:
```json
{
  "channel": "channel_name",
  "messages": [
    {
      "user": "display_name",
      "ts": "unix_timestamp",
      "text": "message text",
      "replies": [{"user": "display_name", "text": "reply text"}]
    }
  ]
}
```
