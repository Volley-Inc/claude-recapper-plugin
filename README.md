# Recapper

Collects your daily work activity across all your tools and produces:

1. A **conversational journal summary** — first-person, past-tense prose suitable for journaling or end-of-day notes
2. A **structured JSON object** ready to import into your Impact Archive app

## Sources

| Source | Method |
|--------|--------|
| Slack | MCP (`claude.ai Slack`) → falls back to REST API |
| Linear | MCP (`claude.ai Linear`) → falls back to GraphQL API |
| GitHub | `gh` CLI (reads `GITHUB_TOKEN` automatically) |
| Notion | MCP (`claude.ai Notion`) → falls back to REST API |
| Datadog | REST API (`DATADOG_API_KEY` + `DATADOG_APP_KEY`) |
| Google Calendar | MCP (`claude.ai Google Calendar`) |

## Usage

```
/recapper              # Recap today
/recapper 2026-05-13   # Recap a specific date
```

## Environment Variables

MCP-authenticated sources (Slack, Linear, Notion, Google Calendar) don't need env vars — authenticate once via Claude Code settings and they just work. Env vars are fallbacks for when MCP isn't available, plus credentials for Datadog (no MCP).

```bash
# Slack fallback (only needed if Slack MCP isn't authenticated)
SLACK_USER_ID=U012AB3CD        # Your Slack member ID (profile → ••• → Copy member ID)
SLACK_BOT_TOKEN=xoxp-...       # User OAuth token with search:read scope (starts with xoxp-)

# GitHub (managed by gh CLI — run `gh auth login` instead of setting these manually)
GITHUB_TOKEN=ghp_...           # Read automatically by gh CLI
GITHUB_USERNAME=yourhandle     # Optional — auto-detected from gh CLI if unset

# Linear fallback (only needed if Linear MCP isn't authenticated)
LINEAR_API_KEY=lin_api_...     # Settings → API → Personal API keys

# Notion fallback (only needed if Notion MCP isn't authenticated)
NOTION_TOKEN=secret_...        # notion.so/my-integrations → Internal Integration Token

# Datadog (always required — no MCP available)
DATADOG_API_KEY=...            # Organization Settings → API Keys
DATADOG_APP_KEY=...            # Organization Settings → Application Keys (personal, not shared)
DATADOG_USER_EMAIL=...         # Your Datadog login email (used to filter audit logs to your actions)
```

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
