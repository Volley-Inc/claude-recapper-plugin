# Recapper

Collects your daily work activity across all your tools and produces:

1. A **conversational journal summary** — first-person, past-tense prose suitable for journaling or end-of-day notes
2. A **structured JSON object** ready to import into your Impact Archive app

## Sources

| Source | Method |
|--------|--------|
| Slack | MCP (`claude.ai Slack`) → falls back to REST API |
| Linear | MCP (`claude.ai Linear`) → falls back to GraphQL API |
| GitHub | `gh` CLI (reads `GITHUB_TOKEN`) |
| Notion | MCP (`claude.ai Notion`) → falls back to REST API |
| Datadog | REST API (`DD-API-KEY` + `DD-APPLICATION-KEY`) |
| Google Calendar | MCP (`claude.ai Google Calendar`) |

## Usage

```
/recapper              # Recap today
/recapper 2026-05-13   # Recap a specific date
```

## Environment Variables

Set these in your shell profile or `.env`:

```bash
SLACK_BOT_TOKEN=xoxb-...       # Slack bot token (fallback only)
SLACK_USER_ID=U012AB3CD        # Your Slack user ID
GITHUB_TOKEN=ghp_...           # GitHub personal access token
GITHUB_USERNAME=yourhandle     # Your GitHub username
LINEAR_API_KEY=lin_api_...     # Linear API key (fallback only)
NOTION_TOKEN=secret_...        # Notion integration token (fallback only)
DATADOG_API_KEY=...            # Datadog API key
DATADOG_APP_KEY=...            # Datadog Application key
```

MCPs (Slack, Linear, Notion, Google Calendar) are preferred and use their own authentication — env vars are only needed as fallbacks.

## Output Format

```json
{
  "date": "YYYY-MM-DD",
  "contributions": [...],
  "journalEntries": [...],
  "quotes": [...]
}
```

See `skills/recapper/references/output-templates.md` for full schema documentation.

## Installation

Install via the Volley Claude Marketplace:

```
/plugin install recapper
```

Or install directly from the plugin repo:

```
/plugin install github:Volley-Inc/claude-recapper-plugin
```
