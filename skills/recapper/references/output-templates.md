# Output Templates

Full schema and examples for the Impact Archive JSON output.

## Top-Level Shape

```json
{
  "date": "2026-05-14",
  "contributions": [...],
  "journalEntries": [...],
  "quotes": [...]
}
```

---

## `contributions[]`

One entry per discrete unit of work. Multiple sources can confirm the same contribution — use `sources[]` for that.

### Schema

```json
{
  "id": "string",
  "date": "YYYY-MM-DD",
  "source": "github | linear | slack | notion | datadog | google_calendar",
  "sources": ["github", "slack"],
  "type": "string",
  "category": "shipped | in_progress | collaborated | incident | planned",
  "title": "string",
  "description": "string",
  "url": "string",
  "metadata": {}
}
```

### `id` convention

Compose from source + type + unique identifier to guarantee uniqueness:
- GitHub PR: `github-pr-{repo}-{number}` → `github-pr-fantastic-succotash-42`
- GitHub commit: `github-commit-{sha7}` → `github-commit-a1b2c3d`
- Linear issue: `linear-issue-{identifier}` → `linear-issue-ENG-123`
- Slack message: `slack-msg-{ts}` → `slack-msg-1715693520.123456`
- Notion page: `notion-page-{id8}` → `notion-page-a1b2c3d4`
- Datadog resource: `datadog-{type}-{id}` → `datadog-dashboard-abc123`
- Calendar event: `gcal-event-{id8}` → `gcal-event-d4e5f6a7`

### `type` values by source

**GitHub:**
- `commit` — push to any branch
- `pr_opened` — pull request created
- `pr_merged` — pull request merged
- `pr_closed` — pull request closed without merge
- `code_review` — submitted a review (approved, requested changes, commented)
- `review_comment` — inline comment on a PR
- `issue_opened` — issue created
- `issue_closed` — issue closed
- `issue_comment` — comment left on an issue
- `branch_created` — new branch pushed

**Linear:**
- `issue_created` — issue created by user
- `issue_completed` — issue moved to Done/Completed state
- `issue_in_progress` — issue updated/moved to In Progress
- `issue_comment` — comment left on an issue

**Slack:**
- `message` — message sent in a channel or DM
- `thread_reply` — reply within a thread
- `dm` — direct message sent

**Notion:**
- `page_created` — new page created
- `page_edited` — existing page edited
- `comment` — comment left on a page or block

**Datadog:**
- `incident_triggered` — new incident created or triggered
- `incident_resolved` — incident marked resolved
- `incident_response` — postmortem, timeline entry, or note added
- `dashboard_created`
- `dashboard_edited`
- `monitor_created`
- `monitor_edited`
- `notebook_created`
- `notebook_edited`

**Google Calendar:**
- `1on1`
- `interview`
- `performance_review`
- `standup`
- `all_hands`
- `team_meeting`
- `focus_time`

### `metadata` by source

**GitHub commit:**
```json
{
  "repo": "fantastic-succotash",
  "branch": "main",
  "sha": "a1b2c3d4e5f6",
  "message": "Fix auth token refresh",
  "files_changed": 3
}
```

**GitHub PR:**
```json
{
  "repo": "fantastic-succotash",
  "number": 42,
  "state": "merged",
  "head_branch": "feature/auth-fix",
  "base_branch": "main",
  "additions": 120,
  "deletions": 45
}
```

**Linear issue:**
```json
{
  "identifier": "ENG-123",
  "state": "Done",
  "priority": 2,
  "team": "Engineering",
  "labels": ["bug", "auth"]
}
```

**Slack message:**
```json
{
  "channel": "#engineering",
  "channel_type": "public | private | dm",
  "thread": true,
  "word_count": 42
}
```

**Notion page:**
```json
{
  "page_id": "a1b2c3d4-...",
  "parent_type": "workspace | database | page",
  "last_edited_by": "Jaimie Diemer"
}
```

**Datadog resource:**
```json
{
  "resource_type": "dashboard | monitor | notebook | incident",
  "resource_id": "abc-123",
  "resource_name": "API Latency Overview",
  "action": "created | modified | resolved"
}
```

**Google Calendar event:**
```json
{
  "start": "2026-05-14T10:00:00",
  "end": "2026-05-14T10:30:00",
  "duration_minutes": 30,
  "attendees": 2,
  "attended": true,
  "organizer": "someone@company.com"
}
```

---

## `journalEntries[]`

Synthesized narrative paragraphs. Typically 1 entry covering the full day. May use 2–3 entries for very long or varied days.

### Schema

```json
{
  "date": "YYYY-MM-DD",
  "content": "string",
  "tags": ["string"]
}
```

### Tag vocabulary

Draw tags from the actual work — don't invent tags not supported by the contributions:

```
engineering, product, design, data, infra, security, reliability
feature, bugfix, refactor, testing, documentation
shipped, in-progress, blocked, unblocked
meeting, interview, planning, review
incident, oncall, postmortem
slack, linear, github, notion, datadog
```

### Example

```json
{
  "date": "2026-05-14",
  "content": "Today was mostly heads-down on the promo engine. I merged the Slack integration PR (#47) after addressing review feedback from Sarah — the main change was switching from polling to webhooks, which cuts latency significantly. In the afternoon I shifted to the recapper skill, which took longer than expected because the Datadog audit API requires both API and app keys but the error messages aren't obvious. Wrapped up with a 1:1 with Alex where we aligned on the Q2 roadmap priorities.",
  "tags": ["engineering", "feature", "shipped", "meeting", "linear", "github"]
}
```

---

## `quotes[]`

Notable, substantive, or quotable things written during the day. Good candidates: analytical Slack messages, PR descriptions that explain a non-obvious decision, Linear comments that articulate a constraint.

### Schema

```json
{
  "date": "YYYY-MM-DD",
  "source": "slack | notion | github | linear",
  "context": "string",
  "content": "string",
  "url": "string"
}
```

### Selection criteria

Include a quote if it is:
- **Substantive**: > 30 words
- **Analytical**: explains *why*, not just *what*
- **Opinionated**: takes a position or recommendation
- **Memorable**: something you'd want to re-read in 6 months

Exclude:
- Status updates ("LGTM", "on it", "done")
- Emoji-only or short reactions
- Auto-generated content (CI bot messages, Linear auto-comments)

### Example

```json
{
  "date": "2026-05-14",
  "source": "slack",
  "context": "#engineering — thread on promo engine architecture",
  "content": "The real issue with polling here isn't the latency — it's that we're burning Slack API quota on read calls that return nothing 90% of the time. Switching to webhooks drops our API calls by ~10x and gives us sub-second delivery. The tradeoff is we need to handle webhook retries, but that's a solved problem with our existing retry queue.",
  "url": "https://volleygames.slack.com/archives/C.../p..."
}
```

---

## Full Example Output

```json
{
  "date": "2026-05-14",
  "contributions": [
    {
      "id": "github-pr-fantastic-succotash-47",
      "date": "2026-05-14",
      "source": "github",
      "sources": ["github", "slack"],
      "type": "pr_merged",
      "category": "shipped",
      "title": "Add Slack webhook integration",
      "description": "Replaced polling with webhooks for Slack message delivery. Cuts API quota usage by ~10x.",
      "url": "https://github.com/Volley-Inc/fantastic-succotash/pull/47",
      "metadata": {
        "repo": "fantastic-succotash",
        "number": 47,
        "state": "merged",
        "head_branch": "feature/slack-webhooks",
        "base_branch": "main"
      }
    },
    {
      "id": "gcal-event-a1b2c3d4",
      "date": "2026-05-14",
      "source": "google_calendar",
      "sources": ["google_calendar"],
      "type": "1on1",
      "category": "collaborated",
      "title": "1:1 with Alex",
      "description": "Weekly 1:1. Discussed Q2 roadmap priorities.",
      "url": "https://calendar.google.com/calendar/event?eid=...",
      "metadata": {
        "start": "2026-05-14T16:00:00",
        "end": "2026-05-14T16:30:00",
        "duration_minutes": 30,
        "attendees": 2,
        "attended": true
      }
    }
  ],
  "journalEntries": [
    {
      "date": "2026-05-14",
      "content": "Today was mostly heads-down on the promo engine. I merged the Slack integration PR (#47) after addressing review feedback from Sarah — the main change was switching from polling to webhooks, which cuts latency significantly. In the afternoon I shifted to the recapper skill, which took longer than expected because the Datadog audit API requires both API and app keys but the error messages aren't obvious. Wrapped up with a 1:1 with Alex where we aligned on the Q2 roadmap priorities.",
      "tags": ["engineering", "feature", "shipped", "meeting", "github", "slack"]
    }
  ],
  "quotes": [
    {
      "date": "2026-05-14",
      "source": "slack",
      "context": "#engineering — thread on promo engine architecture",
      "content": "The real issue with polling here isn't the latency — it's that we're burning Slack API quota on read calls that return nothing 90% of the time. Switching to webhooks drops our API calls by ~10x and gives us sub-second delivery.",
      "url": "https://volleygames.slack.com/archives/C.../p..."
    }
  ]
}
```
