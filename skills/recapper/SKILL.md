---
name: recapper
description: Use when the user wants to recap their day, collect daily work activity, generate a daily summary, or populate their Impact Archive. Collects activity from Slack, Linear, GitHub, Notion, Datadog, and Google Calendar and outputs a conversational journal summary plus structured JSON. Triggers on "recap my day", "daily recap", "what did I do today", "impact archive", "recapper", or "/recapper".
version: 1.0.0
---

# Daily Activity Recapper

Collects your work activity across all tools for a given day and produces a conversational journal entry plus structured JSON for your Impact Archive.

## Usage

```
/recapper [date]
```

**Arguments:**
- `date` (optional): Target date in `YYYY-MM-DD` format. Defaults to today.

**Examples:**
```
/recapper              # Recap today
/recapper 2026-05-13   # Recap a specific date
```

## Environment Variables

Read from the environment — never hardcode values:

| Variable | Purpose | Required |
|----------|---------|----------|
| `SLACK_USER_ID` | Your Slack user ID (e.g. `U012AB3CD`) | Yes, for Slack fallback |
| `SLACK_BOT_TOKEN` | Slack bot token | Only if MCP unavailable |
| `GITHUB_USERNAME` | Your GitHub handle | Yes |
| `GITHUB_TOKEN` | GitHub PAT | Yes (read by `gh` CLI) |
| `LINEAR_API_KEY` | Linear API key | Only if MCP unavailable |
| `NOTION_TOKEN` | Notion integration token | Only if MCP unavailable |
| `DATADOG_API_KEY` | Datadog API key | Yes, for Datadog |
| `DATADOG_APP_KEY` | Datadog Application key | Yes, for Datadog |

## Workflow

```
Phase 1: Setup          → Resolve date, verify env vars
Phase 2: Parallel Fetch → Collect from all 6 sources concurrently
Phase 3: Synthesize     → Deduplicate, categorize, rank by significance
Phase 4: Output         → Write conversational summary + structured JSON
```

---

## Phase 1: Setup

Resolve the target date from the first argument, defaulting to today:

```bash
TARGET_DATE="${1:-$(date +%Y-%m-%d)}"
echo "Recapping activity for: $TARGET_DATE"
```

Then announce:
> "Collecting activity for **{TARGET_DATE}**. Fetching from Slack, Linear, GitHub, Notion, Datadog, and Google Calendar..."

---

## Phase 2: Parallel Fetch

Fetch all sources. Run independent fetches concurrently where possible. If an MCP tool is unavailable or returns an error, fall back to the REST API. If both fail, mark the source as `unavailable` and continue — never halt on a single source failure.

### 2a. Slack

**Preferred: MCP** (`mcp__claude_ai_Slack__slack_search_public_and_private`)

Search for messages sent by the user on the target date. Use these queries:
- `from:@me after:{TARGET_DATE} before:{NEXT_DAY}` — messages sent
- Look for threads where the user replied

From results, collect:
- Messages sent: text, channel name, timestamp, thread context (if reply)
- Channels where at least one message was sent
- Notable message bodies (substantive > emoji reactions)

**Fallback: REST API** (if MCP unavailable or unauthenticated):
```bash
curl -s "https://slack.com/api/search.messages" \
  -H "Authorization: Bearer $SLACK_BOT_TOKEN" \
  --data-urlencode "query=from:<@${SLACK_USER_ID}> after:${TARGET_DATE}" \
  --data-urlencode "count=100" \
  | jq '.messages.matches[] | {text, channel: .channel.name, ts}'
```

For each message, capture the `permalink` field from the MCP result or REST response — this is the direct link to the message in Slack. Always populate `url` with the permalink; never leave it null.

**Structured output per message:**
```json
{
  "source": "slack",
  "type": "message",
  "channel": "#channel-name",
  "text": "...",
  "timestamp": "2026-05-14T10:32:00Z",
  "thread_id": "optional",
  "url": "https://volleygames.slack.com/archives/C.../p..."
}
```

---

### 2b. Linear

**Preferred: MCP** (`mcp__claude_ai_Linear__list_issues`, `mcp__claude_ai_Linear__list_comments`)

Fetch in parallel:
1. Issues assigned to you with `updatedAt >= TARGET_DATE`
2. Comments you created on `TARGET_DATE`

For each issue returned, note the state transition if available (e.g., moved to "In Progress" → "Done").

**Fallback: GraphQL API** (if MCP unavailable):
```bash
curl -s -X POST https://api.linear.app/graphql \
  -H "Authorization: $LINEAR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "{ viewer { assignedIssues(filter: { updatedAt: { gt: \"'"$TARGET_DATE"'T00:00:00.000Z\" } }) { nodes { id identifier title state { name } updatedAt url priority } } } }"
  }' | jq '.data.viewer.assignedIssues.nodes[]'
```

**Classify each issue as:**
- `completed` — state name contains "Done", "Completed", "Merged", "Deployed"
- `in_progress` — state name contains "In Progress", "In Review"
- `created` — `createdAt` starts with TARGET_DATE
- `commented` — user left a comment on this date (from comments fetch)

---

### 2c. GitHub

Use the `gh` CLI (reads `GITHUB_TOKEN` automatically). Run these sequentially:

```bash
GITHUB_USERNAME="${GITHUB_USERNAME:-$(gh api user --jq '.login' 2>/dev/null)}"

# User events for the target date
gh api "users/$GITHUB_USERNAME/events" --paginate \
  --jq "[.[] | select(.created_at | startswith(\"$TARGET_DATE\"))]" \
  2>/dev/null > /tmp/recap-gh-events.json

# PRs reviewed on this date
gh search prs \
  --reviewed-by="$GITHUB_USERNAME" \
  --updated="${TARGET_DATE}..${TARGET_DATE}" \
  --json number,title,url,repository,state \
  --limit 50 \
  2>/dev/null > /tmp/recap-gh-reviews.json
```

**Map event types to contribution types:**

| GitHub Event | Contribution Type |
|---|---|
| `PushEvent` | `commit` — extract `.payload.commits[]` |
| `PullRequestEvent` action=`opened` | `pr_opened` |
| `PullRequestEvent` action=`closed` + merged=true | `pr_merged` |
| `PullRequestReviewEvent` | `code_review` |
| `PullRequestReviewCommentEvent` | `review_comment` |
| `IssueCommentEvent` | `issue_comment` |
| `IssuesEvent` action=`opened` | `issue_opened` |
| `IssuesEvent` action=`closed` | `issue_closed` |
| `CreateEvent` | `branch_created` (skip tag events) |

Parse `/tmp/recap-gh-events.json` with `jq` to extract structured records. Pull commit messages, PR titles, repo names, and URLs.

---

### 2d. Notion

**Preferred: MCP** (`mcp__claude_ai_Notion__notion-search`)

Search for pages edited on or after the target date:
```
filter: { property: "last_edited_time", date: { on_or_after: TARGET_DATE } }
sort: { direction: "descending", timestamp: "last_edited_time" }
```

Then filter results client-side to pages where `last_edited_time` starts with `TARGET_DATE`.

For each page found, optionally fetch comments via `mcp__claude_ai_Notion__notion-get-comments` to identify pages where the user left comments.

**Fallback: REST API** (if MCP unavailable):
```bash
curl -s -X POST https://api.notion.com/v1/search \
  -H "Authorization: Bearer $NOTION_TOKEN" \
  -H "Notion-Version: 2022-06-28" \
  -H "Content-Type: application/json" \
  -d '{
    "filter": { "property": "last_edited_time", "date": { "on_or_after": "'"$TARGET_DATE"'" } },
    "sort": { "direction": "descending", "timestamp": "last_edited_time" }
  }' | jq '.results[] | {id, title: .properties.title.title[0].plain_text, last_edited_time, url}'
```

**Classify each page as:**
- `created` — `created_time` starts with TARGET_DATE
- `edited` — `last_edited_time` starts with TARGET_DATE but created earlier
- `commented` — user comment found from get-comments call

---

### 2e. Datadog

No Datadog MCP is available — always use the REST API. All four calls use the same auth headers:

```bash
DD_HEADERS=(-H "DD-API-KEY: $DATADOG_API_KEY" -H "DD-APPLICATION-KEY: $DATADOG_APP_KEY")

# Audit trail — captures dashboard/monitor/notebook creates and edits attributed to your user
curl -s "https://api.datadoghq.com/api/v2/audit/events?filter[from]=${TARGET_DATE}T00:00:00Z&filter[to]=${TARGET_DATE}T23:59:59Z&page[limit]=100" \
  "${DD_HEADERS[@]}" 2>/dev/null > /tmp/recap-dd-audit.json

# Active incidents (created or updated today)
curl -s "https://api.datadoghq.com/api/v2/incidents?filter[created][start]=${TARGET_DATE}T00:00:00Z&filter[created][end]=${TARGET_DATE}T23:59:59Z&page[size]=50" \
  "${DD_HEADERS[@]}" 2>/dev/null > /tmp/recap-dd-incidents.json
```

**Parse audit events by `type`:**

| Audit type | Contribution type | URL pattern |
|---|---|---|
| `dashboard_created` | `dashboard_created` | `https://app.datadoghq.com/dashboard/{id}` |
| `dashboard_modified` | `dashboard_edited` | `https://app.datadoghq.com/dashboard/{id}` |
| `monitor_created` | `monitor_created` | `https://app.datadoghq.com/monitors/{id}` |
| `monitor_modified` | `monitor_edited` | `https://app.datadoghq.com/monitors/{id}` |
| `notebook_created` | `notebook_created` | `https://app.datadoghq.com/notebook/{id}` |
| `notebook_modified` | `notebook_edited` | `https://app.datadoghq.com/notebook/{id}` |

Extract the resource `id` from `attributes.resource.id` in each audit event and construct the URL using the pattern above. Always populate `url`; never leave it null.

Filter audit events to only those where the `userId` or `userEmail` matches the current user.

**For incidents**, check `attributes.created` and `attributes.resolved` timestamps against TARGET_DATE. Incidents link to `https://app.datadoghq.com/incidents/{id}`.

---

### 2f. Google Calendar

**Preferred: MCP** (`mcp__claude_ai_Google_Calendar__list_events`)

Fetch all events for the target date:
- `timeMin: TARGET_DATE + "T00:00:00Z"`
- `timeMax: TARGET_DATE + "T23:59:59Z"`

**Classify each event:**

| Classification | Criteria |
|---|---|
| `1on1` | Exactly 2 attendees, duration 20–35 min |
| `interview` | Title matches: `interview`, `loop`, `debrief`, `onsite` |
| `performance_review` | Title matches: `perf review`, `performance`, `calibration`, `feedback session` |
| `standup` | Title matches: `standup`, `stand-up`, `sync`; ≤ 15 min; recurring |
| `all_hands` | Title matches: `all-hands`, `all hands`, `company meeting`, `town hall` |
| `team_meeting` | 3+ attendees, doesn't match above |
| `focus_time` | Blocked time, 0 attendees, title: `focus`, `deep work`, `blocked` |

For each event, record: title, start time, duration (minutes), attendee count, classification, whether the user declined (`responseStatus === "declined"` → `attended: false`), and the `htmlLink` field from the event as `url`. Always populate `url`; never leave it null.

---

## Phase 3: Synthesize

After all fetches complete:

### 3a. Deduplicate

The same work may surface in multiple sources (e.g., a merged PR appears in GitHub events AND may be mentioned in a Slack message). Identify duplicates by:
- Matching URLs across sources
- Matching PR/issue titles with high similarity
- Consolidate into a single contribution entry with `sources: ["github", "slack"]`

### 3b. Classify each item

Assign a `category` to every contribution:

| Category | Criteria |
|---|---|
| `shipped` | Code merged to main, issue moved to Done, incident resolved |
| `in_progress` | PR opened/updated, issue in progress, document drafted |
| `collaborated` | Code review, meeting, substantive Slack thread, Notion comment |
| `incident` | Datadog incident triggered, responded to, or investigated |
| `planned` | Calendar blocked, issue created but not started |

### 3c. Select notable quotes

From Slack messages, PR descriptions, Linear comments, and Notion edits — identify 1–3 items that are:
- Substantive (>30 words)
- Opinionated or analytical (not just status updates)
- Memorable or shareable

These become `quotes[]`.

### 3d. Rank by significance

Sort contributions: `shipped` > `incident` > `collaborated` > `in_progress` > `planned`

Within each category, prefer items with longer descriptions or cross-source confirmation.

---

## Phase 4: Output

### Part 1: Conversational Summary

Write 2–4 paragraphs in first-person, past tense, suitable for a journal entry. Rules:
- Lead with the most significant contribution
- Name specific PRs, issues, meetings, and documents
- Group related items naturally ("I spent most of the morning on X, then switched to Y")
- Mention collaborators if their names appear in the data
- Note if something was blocked, resolved, or handed off
- Tone: professional but personal — like writing to your future self
- No bullet points — flowing prose only

### Part 2: JSON Output

Print the full JSON in a fenced code block. See [references/output-templates.md](references/output-templates.md) for complete field definitions.

```json
{
  "date": "YYYY-MM-DD",
  "contributions": [
    {
      "id": "github-pr-123",
      "date": "YYYY-MM-DD",
      "source": "github",
      "sources": ["github"],
      "type": "pr_merged",
      "category": "shipped",
      "title": "...",
      "description": "...",
      "url": "https://...",
      "metadata": {}
    }
  ],
  "journalEntries": [
    {
      "date": "YYYY-MM-DD",
      "content": "...",
      "tags": ["engineering", "feature"]
    }
  ],
  "quotes": [
    {
      "date": "YYYY-MM-DD",
      "source": "slack",
      "context": "#channel or PR title",
      "content": "..."
    }
  ]
}
```

---

## Troubleshooting

| Issue | Cause | Fix |
|-------|-------|-----|
| Slack returns 0 messages | Wrong `SLACK_USER_ID` format | Must be `U012ABCDE` not `@username` |
| Slack MCP not authenticated | MCP session expired | Re-authenticate via MCP settings, or set `SLACK_BOT_TOKEN` |
| Linear shows no issues | MCP not authenticated | Set `LINEAR_API_KEY` for fallback |
| GitHub shows 0 events | Wrong username or missing token | Run `gh api user` to confirm identity |
| Datadog 403 | Missing app key | Both `DATADOG_API_KEY` and `DATADOG_APP_KEY` required |
| Datadog audit returns no user events | Wrong user filter | Check `userEmail` field in audit response against your email |
| Google Calendar empty | MCP not authenticated | Authenticate Google Calendar MCP |
| Notion 401 | Token invalid | Run `NOTION_TOKEN=secret_... /recapper` or re-authenticate MCP |
| Date format error | Wrong date format | Use `YYYY-MM-DD` — e.g. `2026-05-14` not `May 14` |
