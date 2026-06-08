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
| `SLACK_USER_ID` | Your Slack user ID (e.g. `U012AB3CD`) | Only if Slack MCP unavailable |
| `SLACK_BOT_TOKEN` | Slack bot token | Only if Slack MCP unavailable |
| `GITHUB_USERNAME` | Your GitHub handle (auto-detected if unset) | No |
| `GITHUB_TOKEN` | GitHub PAT (read by `gh` CLI) | No — `gh auth login` handles this |
| `LINEAR_API_KEY` | Linear API key | Only if Linear MCP unavailable |
| `NOTION_TOKEN` | Notion integration token | Only if Notion MCP unavailable |
| `DATADOG_API_KEY` | Datadog API key | Yes, for Datadog |
| `DATADOG_APP_KEY` | Datadog Application key | Yes, for Datadog |
| `DATADOG_USER_EMAIL` | Your Datadog account email (used to filter audit logs to your actions only) | Yes, for Datadog |

MCP-preferred sources (Slack, Linear, Notion, Google Calendar) use their own authentication — env vars are fallbacks only.

## Workflow

```
Phase 1: Setup          → Resolve date, check config, prompt for anything missing
Phase 2: Parallel Fetch → Collect from all 6 sources concurrently
Phase 3: Synthesize     → Deduplicate, categorize, rank by significance
Phase 4: Output         → Write conversational summary + structured JSON
```

---

## Phase 1: Setup

### 1a. Resolve target date

```bash
TARGET_DATE="${1:-$(date +%Y-%m-%d)}"
# NEXT_DAY is used in Slack search queries — the day after TARGET_DATE
NEXT_DAY=$(date -d "$TARGET_DATE + 1 day" +%Y-%m-%d 2>/dev/null || \
           date -j -v+1d -f "%Y-%m-%d" "$TARGET_DATE" +%Y-%m-%d 2>/dev/null)
```

### 1b. Load ignored-sources config

```bash
RECAPPER_CONFIG="${HOME}/.config/recapper/config.json"
mkdir -p "${HOME}/.config/recapper"
if [ ! -f "$RECAPPER_CONFIG" ]; then
  echo '{"ignoredSources":[],"calendarIds":[],"slackIncludeDMs":true,"onboardingComplete":false}' > "$RECAPPER_CONFIG"
fi
# FIRST_RUN=true if onboarding was never completed (new install or interrupted mid-onboarding)
# Missing key defaults to true (backward compat — existing configs pre-date this field)
FIRST_RUN=$(jq -r 'if .onboardingComplete == true then "false" else "true" end' "$RECAPPER_CONFIG" 2>/dev/null || echo "true")
```

To check whether a source is ignored:
```bash
jq -e --arg src "source-name" '.ignoredSources | index($src) != null' "$RECAPPER_CONFIG" 2>/dev/null
```

To add a source to the ignored list:
```bash
tmp="$(mktemp)" && jq --arg src "source-name" '.ignoredSources += [$src] | .ignoredSources |= unique' "$RECAPPER_CONFIG" > "$tmp" && mv "$tmp" "$RECAPPER_CONFIG"
```

After loading config, **immediately mark all sources in `ignoredSources` as `unavailable`** for this run — this applies on every run, not just first:
```bash
jq -r '.ignoredSources[]?' "$RECAPPER_CONFIG" 2>/dev/null
# For each source returned, mark it unavailable before proceeding to any credential checks or fetching
```
Sources marked unavailable here are silently skipped in all subsequent steps — no prompts, no fetching.

### 1c. First-run source configuration

If `FIRST_RUN` is true (set in step 1b), show the following before doing anything else:

> "Welcome to Recapper! Here's what I'll collect from each source. You can choose what to include now — you can always change this by running `/recapper` again.
>
> | Source | What's collected |
> |---|---|
> | **Slack** | Messages you sent and threads you replied in, with channel context and links |
> | **Linear** | Issues assigned to you that were updated, comments you left, and state transitions (e.g. Backlog → In Progress → Done) |
> | **GitHub** | Commits pushed, PRs opened or merged, branches created, code reviews given, review comments, and issue comments |
> | **Notion** | Pages you created or edited, and comments you left on any page |
> | **Datadog** | Dashboards, monitors, and notebooks you created or edited; incidents that were created or resolved |
> | **Google Calendar** | Meetings you attended, classified by type (1:1, standup, team meeting, all-hands, interview, focus time) |
>
> For each source, reply with:
> **yes** — include this source every run
> **skip** — skip today, ask again next run
> **never** — never include this source"

Then prompt for each source **individually**, waiting for a response before moving to the next:

> "**Slack** [yes/skip/never]:"

[Wait for input.]

If the user chose **yes** for Slack, immediately follow up with:

> "**Include Direct Messages?** Should DMs appear in your Slack recap?
> **yes** — include DMs alongside channel messages
> **no** — channel messages only (recommended for work recaps)"

[Wait for input. Save the DM preference to config:]

```bash
# If user said yes to DMs:
tmp="$(mktemp)" && jq '.slackIncludeDMs = true' "$RECAPPER_CONFIG" > "$tmp" && mv "$tmp" "$RECAPPER_CONFIG"
# If user said no to DMs:
tmp="$(mktemp)" && jq '.slackIncludeDMs = false' "$RECAPPER_CONFIG" > "$tmp" && mv "$tmp" "$RECAPPER_CONFIG"
```

> "**Linear** [yes/skip/never]:"

[Wait for input.]

> "**GitHub** [yes/skip/never]:"

[Wait for input.]

> "**Notion** [yes/skip/never]:"

[Wait for input.]

> "**Datadog** [yes/skip/never]:"

[Wait for input.]

> "**Google Calendar** [yes/skip/never]:"

[Wait for input.]

If the user chose **yes** for Google Calendar and the Calendar MCP is available, call `mcp__claude_ai_Google_Calendar__list_calendars` to fetch the user's calendars. Then show:

> "You have access to the following calendars:
> [list each calendar with a number, e.g. "1. Work (primary)", "2. Team Meetings", "3. Personal"]
>
> Which would you like to include in your recaps? Enter the numbers separated by commas (or press Enter to use your primary calendar only):"

[Wait for input. Parse the numbers and save the selected calendar IDs to config:]

```bash
# Save selected calendarIds to config
tmp="$(mktemp)" && jq --argjson ids '["cal-id-1","cal-id-2"]' '.calendarIds = $ids' "$RECAPPER_CONFIG" > "$tmp" && mv "$tmp" "$RECAPPER_CONFIG"
```

If the user presses Enter without selecting, save only the primary calendar ID. If the MCP is unavailable, skip calendar selection and default to primary.

For each source where the user chose **never**: add it to `ignoredSources` using the pattern in 1b (use the exact slugs: `"slack"`, `"linear"`, `"github"`, `"notion"`, `"datadog"`, `"calendar"`), and mark as `unavailable` for this run. For **skip**: mark as `unavailable` for this run only — do not write to config. For **yes**: no action needed (source remains available).

After all six sources are answered, mark onboarding as complete so a future interrupted run doesn't re-trigger it:

```bash
tmp="$(mktemp)" && jq '.onboardingComplete = true' "$RECAPPER_CONFIG" > "$tmp" && mv "$tmp" "$RECAPPER_CONFIG"
```

Then continue to step 1d. The credential check steps (1e, 1f, and the Calendar check in Phase 2) must skip any source already marked `unavailable` here — do not prompt again for the same source.

If `FIRST_RUN` is false, skip this step entirely.

### 1d. Shell profile setup

Define these helpers once — they are used by the Slack, Linear, Notion, and Datadog save flows later. **This step always runs unconditionally**, regardless of which sources are available or skipped:

```bash
if [ -n "$ZSH_VERSION" ] || case "$SHELL" in */zsh) true;; *) false;; esac; then
  SHELL_PROFILE="$HOME/.zshrc"
elif [ -f "$HOME/.bashrc" ]; then
  SHELL_PROFILE="$HOME/.bashrc"
else
  SHELL_PROFILE="$HOME/.bash_profile"
fi

# Escape any embedded single quotes in values before writing
escape_sq() { printf '%s' "$1" | sed "s/'/'\\\\''/g"; }
```

### 1e. Check GitHub CLI

If GitHub was already marked `unavailable` in step 1b or 1c, skip this step entirely.

```bash
gh auth status 2>/dev/null
```

If the command fails or reports "not logged in":
- Check if `"github"` is in `ignoredSources`. If yes, silently mark GitHub as `unavailable` and continue.
- If not ignored, show:

> "⚠️ GitHub CLI isn't authenticated — GitHub activity won't be included.
>
> What would you like to do?
> **a) Ignore forever** — don't remind me about GitHub again
> **b) Ignore this time** — skip GitHub now, remind me next run
> **c) Fix it** — I'll walk you through logging in"

If **a)**: add `"github"` to `ignoredSources` in config, mark as `unavailable`, continue.
If **b)**: mark as `unavailable`, continue.
If **c)**: tell the user:
> "Run this in your terminal and follow the prompts, then re-run `/recapper`:
> ```
> gh auth login
> ```"
Mark as `unavailable` and stop — the user needs to re-run after authenticating.

### 1f. Check Datadog keys

If Datadog was already marked `unavailable` in step 1b or 1c, skip this step entirely.

```bash
echo "${DATADOG_API_KEY:+set}" && echo "${DATADOG_APP_KEY:+set}" && echo "${DATADOG_USER_EMAIL:+set}"
```

If any of the three are missing:
- Check if `"datadog"` is in `ignoredSources`. If yes, silently mark Datadog as `unavailable` and continue.
- If not ignored, show:

> "⚠️ Datadog isn't configured — dashboards, monitors, and incidents won't be included.
>
> What would you like to do?
> **a) Ignore forever** — don't remind me about Datadog again
> **b) Ignore this time** — skip Datadog now, remind me next run
> **c) Fix it** — I'll walk you through getting your API keys"

If **a)**: add `"datadog"` to `ignoredSources` in config, mark as `unavailable`, continue.
If **b)**: mark as `unavailable`, continue.
If **c)**: only prompt for keys that are actually missing — skip any step whose key is already set in the environment:

If `DATADOG_API_KEY` is not set:

> "**Datadog API Key**
> 1. Go to **Datadog → Organization Settings → API Keys** (or ask your admin)
> 2. Click **New Key**, give it a name, and copy the value
>
> Paste your Datadog API Key here (or press Enter to skip Datadog):"

[Wait for user input. If empty, mark Datadog as `unavailable` and stop — do NOT prompt for remaining keys.]

If `DATADOG_APP_KEY` is not set:

> "**Datadog Application Key**
>
> ⚠️ **This must be your own personal Application Key** — not a shared or org-wide one. The recapper uses it to filter audit logs to your activity only; a shared key will return someone else's data or nothing useful.
>
> You'll also need the `audit_logs_read` scope. **This requires admin approval** — if you can't add it yourself, ask your Datadog admin to either grant you the scope or assign you a role that includes it.
>
> **What happens without `audit_logs_read`?** The key will still work, but Datadog audit events (dashboards created, monitors edited, etc.) won't appear in your recap. Everything else (incidents, other sources) is unaffected — it's totally fine to skip this scope if it's not worth the ask.
>
> To create your key:
> 1. Go to **Datadog → Organization Settings → Application Keys**
> 2. Click **New Key**, give it a name
> 3. Add the `audit_logs_read` scope (if available to you)
> 4. Copy the value
>
> Paste your Datadog Application Key here (or press Enter to skip Datadog):"

[Wait for user input. If empty, mark Datadog as `unavailable` and stop — do NOT prompt for remaining keys.]

If `DATADOG_USER_EMAIL` is not set:

> "**Your Datadog email address**
>
> This is the email you use to log in to Datadog. It's used to filter audit logs so you only see your own actions — not the whole org's.
>
> Paste your Datadog account email here (or press Enter to skip Datadog):"

[Wait for user input. If empty, mark Datadog as `unavailable` and continue.]

Whether keys were just collected above or were already in the environment, always verify them now:

```bash
HTTP_STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
  "https://api.datadoghq.com/api/v2/audit/events?page[limit]=1" \
  -H "DD-API-KEY: $DATADOG_API_KEY" \
  -H "DD-APPLICATION-KEY: $DATADOG_APP_KEY" 2>/dev/null) || HTTP_STATUS="000"
```

- If `$HTTP_STATUS` is `200` or `403`:
  - If `403`: tell the user "⚠️ Keys authenticated but you may be missing the `audit_logs_read` scope — Datadog audit events won't appear in recaps, but everything else will still work. Continuing anyway."
  - If keys were **just collected** in this session (not pre-existing in environment): tell the user "✅ Datadog keys verified!" and offer to save:

    > "Save these to your shell profile so you don't have to enter them again?
    > - **Yes** — I'll append them to your shell profile
    > - **No** — use for this session only"

    If **Yes**, append to the shell profile (using `$SHELL_PROFILE` and `escape_sq` defined in 1d):

    ```bash
    printf '\n# Datadog (added by recapper)\n' >> "$SHELL_PROFILE"
    printf "export DATADOG_API_KEY='%s'\n" "$(escape_sq "$DATADOG_API_KEY")" >> "$SHELL_PROFILE"
    printf "export DATADOG_APP_KEY='%s'\n" "$(escape_sq "$DATADOG_APP_KEY")" >> "$SHELL_PROFILE"
    printf "export DATADOG_USER_EMAIL='%s'\n" "$(escape_sq "$DATADOG_USER_EMAIL")" >> "$SHELL_PROFILE"
    ```

    Then tell the user:
    > "Saved to `{SHELL_PROFILE}`. Run `source {SHELL_PROFILE}` to apply in other terminals."

    If **No**, export the values for the current session so Phase 2 can use them.
  - If keys were **pre-existing** in environment: continue silently — no save prompt needed.

- If `$HTTP_STATUS` is `000` (curl failed): tell the user:
  > "⚠️ Couldn't reach Datadog — check your network connection."

  > "Paste corrected API key (or press Enter to skip Datadog):"

  [Wait for user input. If empty, mark Datadog as `unavailable` and continue — do NOT proceed.]

  > "Paste corrected App key (or press Enter to skip Datadog):"

  [Wait for user input. If empty, mark Datadog as `unavailable` and continue. If provided, update `DATADOG_API_KEY` and `DATADOG_APP_KEY` and re-run the HTTP status check above.]

- If `$HTTP_STATUS` is anything else (401, 400, etc.): tell the user:
  > "⚠️ Datadog keys don't seem valid (HTTP $HTTP_STATUS)."

  > "Paste corrected API key (or press Enter to skip Datadog):"

  [Wait for user input. If empty, mark Datadog as `unavailable` and continue — do NOT proceed.]

  > "Paste corrected App key (or press Enter to skip Datadog):"

  [Wait for user input. If empty, mark Datadog as `unavailable` and continue. If provided, update `DATADOG_API_KEY` and `DATADOG_APP_KEY` and re-run the HTTP status check above.]

### 1g. Announce

Build the source list from only the sources not already marked `unavailable` after steps 1b–1f. Then announce:

> "Collecting activity for **{TARGET_DATE}**. Fetching from {comma-separated list of available sources}..."

For example, if GitHub and Datadog were skipped:
> "Collecting activity for **2026-05-14**. Fetching from Slack, Linear, Notion, and Google Calendar..."

---

## Phase 2: Parallel Fetch

**Before fetching anything**, skip any source already marked `unavailable` from Phase 1 (steps 1b–1f) — do not attempt to fetch, fall back, or prompt for it. This ensures sources the user ignored forever or skipped this time are never contacted, even when credentials or MCP sessions happen to be present.

For sources that are available: run independent fetches concurrently where possible. If an MCP tool is unavailable or returns an error, fall back to the REST API. If both fail, mark the source as `unavailable`, show the relevant setup guidance below, and continue — never halt on a single source failure.

### 2a. Slack

Read the DM preference from config:
```bash
SLACK_INCLUDE_DMS=$(jq -r 'if .slackIncludeDMs == false then "false" else "true" end' "$RECAPPER_CONFIG" 2>/dev/null)
```

**Preferred: MCP**
- If `SLACK_INCLUDE_DMS` is `true`: use `mcp__claude_ai_Slack__slack_search_public_and_private`
- If `SLACK_INCLUDE_DMS` is `false`: use `mcp__claude_ai_Slack__slack_search_public` (channel messages only)

Search for messages sent by the user on the target date. Use these queries:
- `from:@me after:{TARGET_DATE} before:{NEXT_DAY}` — messages sent
- Look for threads where the user replied

From results, collect:
- Messages sent: text, channel name, timestamp, thread context (if reply)
- Channels where at least one message was sent
- Notable message bodies (substantive > emoji reactions)

**Fallback: REST API** (if MCP unavailable or unauthenticated):
```bash
# Append "-in:im -in:mpim" to exclude DMs if SLACK_INCLUDE_DMS is false
DM_FILTER=$( [ "$SLACK_INCLUDE_DMS" = "false" ] && echo " -in:im -in:mpim" || echo "" )
curl -s "https://slack.com/api/search.messages" \
  -H "Authorization: Bearer $SLACK_BOT_TOKEN" \
  --data-urlencode "query=from:<@${SLACK_USER_ID}> after:${TARGET_DATE} before:${NEXT_DAY}${DM_FILTER}" \
  --data-urlencode "count=100" \
  | jq '.messages.matches[] | {text, channel: .channel.name, ts}'
```

**On MCP failure and missing fallback env vars**:
- Check if `"slack"` is in `ignoredSources`. If yes, silently mark Slack as `unavailable` and continue.
- If not ignored, show:

> "⚠️ Slack MCP isn't available and the REST fallback isn't configured — Slack messages won't be included.
>
> What would you like to do?
> **a) Ignore forever** — don't remind me about Slack again
> **b) Ignore this time** — skip Slack now, remind me next run
> **c) Fix it** — walk me through setting up Slack"

If **a)**: add `"slack"` to `ignoredSources` in config, mark as `unavailable`, continue.
If **b)**: mark as `unavailable`, continue.
If **c)**: prompt:

> "You can fix this two ways:
>
> **Option A — Authenticate the Slack MCP** (recommended):
> Open Claude Code settings and authenticate the Slack integration, then re-run `/recapper`.
>
> **Option B — Set up the REST fallback**:
>
> **Step 1 — SLACK_USER_ID** — your personal Slack user ID:
> 1. Open Slack and click your profile picture (top right)
> 2. Click **Profile**
> 3. Click the **•••** menu → **Copy member ID**
> It looks like `U012AB3CD`.
>
> Paste your Slack User ID here (or press Enter to skip Slack):"

[Wait for user input. If empty, mark Slack as `unavailable` and continue — do NOT proceed to Step 2.]

> "**Step 2 — SLACK_BOT_TOKEN** — a Slack **User** OAuth Token with `search:read` scope:
> Create a Slack app at api.slack.com/apps, add the `search:read` **user** scope (under "OAuth & Permissions → User Token Scopes"), install the app to your workspace, and copy the **User OAuth Token** (starts with `xoxp-`).
>
> Paste your Slack Bot Token here (or press Enter to skip Slack):"

[Wait for user input. If empty, mark Slack as `unavailable` and continue.]

After collecting both values, offer to save them:

> "Save these to your shell profile so you don't have to enter them again?
> - **Yes** — I'll append them to your shell profile
> - **No** — use for this session only"

If **Yes**, append to the shell profile (using `$SHELL_PROFILE` and `escape_sq` defined in 1d):

```bash
printf '\n# Slack (added by recapper)\n' >> "$SHELL_PROFILE"
printf "export SLACK_USER_ID='%s'\n" "$(escape_sq "$SLACK_USER_ID")" >> "$SHELL_PROFILE"
printf "export SLACK_BOT_TOKEN='%s'\n" "$(escape_sq "$SLACK_BOT_TOKEN")" >> "$SHELL_PROFILE"
```

Then tell the user:
> "Saved to `{SHELL_PROFILE}`. Run `source {SHELL_PROFILE}` to apply in other terminals."

If **No**, export the values for the current session so Phase 2 can use them.

Mark Slack as available with the provided credentials, then proceed to fetch using the REST API fallback above.

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
  "url": "https://yourworkspace.slack.com/archives/C.../p..."
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

**On MCP failure and missing `LINEAR_API_KEY`**:
- Check if `"linear"` is in `ignoredSources`. If yes, silently mark Linear as `unavailable` and continue.
- If not ignored, show:

> "⚠️ Linear MCP isn't available and `LINEAR_API_KEY` isn't set — Linear issues won't be included.
>
> What would you like to do?
> **a) Ignore forever** — don't remind me about Linear again
> **b) Ignore this time** — skip Linear now, remind me next run
> **c) Fix it** — walk me through setting up Linear"

If **a)**: add `"linear"` to `ignoredSources` in config, mark as `unavailable`, continue.
If **b)**: mark as `unavailable`, continue.
If **c)**: prompt:

> "You can fix this two ways:
>
> **Option A — Authenticate the Linear MCP** (recommended):
> Open Claude Code settings and authenticate the Linear integration, then re-run `/recapper`.
>
> **Option B — Set up the REST fallback**:
> 1. Open Linear and go to **Settings → API → Personal API keys**
> 2. Click **Create key**, give it a name, and copy the value
>
> Paste your Linear API key here (or press Enter to skip):"

[Wait for user input. If empty, mark Linear as `unavailable` and continue.]

If provided, offer to save to shell profile:

> "Save this to your shell profile so you don't have to enter it again?
> - **Yes** — I'll append it to your shell profile
> - **No** — use for this session only"

If **Yes**, append to the shell profile (using `$SHELL_PROFILE` and `escape_sq` defined in 1d):
```bash
printf '\n# Linear (added by recapper)\n' >> "$SHELL_PROFILE"
printf "export LINEAR_API_KEY='%s'\n" "$(escape_sq "$LINEAR_API_KEY")" >> "$SHELL_PROFILE"
```

Then tell the user:
> "Saved to `{SHELL_PROFILE}`. Run `source {SHELL_PROFILE}` to apply in other terminals."

If **No**, export for the current session only.

Mark Linear as available with the provided key, then proceed to fetch using the GraphQL fallback above.

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

**On MCP failure and missing `NOTION_TOKEN`**:
- Check if `"notion"` is in `ignoredSources`. If yes, silently mark Notion as `unavailable` and continue.
- If not ignored, show:

> "⚠️ Notion MCP isn't available and `NOTION_TOKEN` isn't set — Notion pages won't be included.
>
> What would you like to do?
> **a) Ignore forever** — don't remind me about Notion again
> **b) Ignore this time** — skip Notion now, remind me next run
> **c) Fix it** — walk me through setting up Notion"

If **a)**: add `"notion"` to `ignoredSources` in config, mark as `unavailable`, continue.
If **b)**: mark as `unavailable`, continue.
If **c)**: prompt:

> "You can fix this two ways:
>
> **Option A — Authenticate the Notion MCP** (recommended):
> Open Claude Code settings and authenticate the Notion integration, then re-run `/recapper`.
>
> **Option B — Set up the REST fallback**:
> 1. Go to [notion.so/my-integrations](https://www.notion.so/my-integrations)
> 2. Click **New integration**, give it a name
> 3. Copy the **Internal Integration Token** (starts with `secret_`)
>
> Paste your Notion token here (or press Enter to skip):"

[Wait for user input. If empty, mark Notion as `unavailable` and continue.]

If provided, offer to save to shell profile:

> "Save this to your shell profile so you don't have to enter it again?
> - **Yes** — I'll append it to your shell profile
> - **No** — use for this session only"

If **Yes**, append to the shell profile (using `$SHELL_PROFILE` and `escape_sq` defined in 1d):
```bash
printf '\n# Notion (added by recapper)\n' >> "$SHELL_PROFILE"
printf "export NOTION_TOKEN='%s'\n" "$(escape_sq "$NOTION_TOKEN")" >> "$SHELL_PROFILE"
```

Then tell the user:
> "Saved to `{SHELL_PROFILE}`. Run `source {SHELL_PROFILE}` to apply in other terminals."

If **No**, export for the current session only.

Mark Notion as available with the provided token, then proceed to fetch using the REST API fallback above.

**Classify each page as:**
- `created` — `created_time` starts with TARGET_DATE
- `edited` — `last_edited_time` starts with TARGET_DATE but created earlier
- `commented` — user comment found from get-comments call

---

### 2e. Datadog

No Datadog MCP is available — always use the REST API. All calls use the same auth headers:

```bash
DD_HEADERS=(-H "DD-API-KEY: $DATADOG_API_KEY" -H "DD-APPLICATION-KEY: $DATADOG_APP_KEY")

# Audit trail — captures dashboard/monitor/notebook creates and edits attributed to your user
curl -s "https://api.datadoghq.com/api/v2/audit/events?filter[from]=${TARGET_DATE}T00:00:00Z&filter[to]=${TARGET_DATE}T23:59:59Z&page[limit]=100" \
  "${DD_HEADERS[@]}" 2>/dev/null > /tmp/recap-dd-audit.json

# Active incidents (created or updated today)
curl -s "https://api.datadoghq.com/api/v2/incidents?filter[created][start]=${TARGET_DATE}T00:00:00Z&filter[created][end]=${TARGET_DATE}T23:59:59Z&page[size]=50" \
  "${DD_HEADERS[@]}" 2>/dev/null > /tmp/recap-dd-incidents.json
```

If keys were not provided in Phase 1 and are still missing, skip Datadog silently (already handled in steps 1b and 1f).

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

Filter audit events to only those where `userEmail` matches `$DATADOG_USER_EMAIL`. (The `userId` field is a UUID — do not compare it against the email address.)

**For incidents**, check `attributes.created` and `attributes.resolved` timestamps against TARGET_DATE. Incidents link to `https://app.datadoghq.com/incidents/{id}`.

---

### 2f. Google Calendar

**Preferred: MCP** (`mcp__claude_ai_Google_Calendar__list_events`)

First, read the configured calendar IDs from config:
```bash
CALENDAR_IDS=$(jq -r '.calendarIds // [] | .[]' "$RECAPPER_CONFIG" 2>/dev/null)
```

If `CALENDAR_IDS` is empty, fetch from the primary calendar only. Otherwise, fetch events for each calendar ID and merge the results. For each calendar, call `list_events` with:
- `timeMin: TARGET_DATE + "T00:00:00Z"`
- `timeMax: TARGET_DATE + "T23:59:59Z"`
- `calendarId: <id>` (omit for primary)

**On MCP failure**:
- If Calendar was already marked `unavailable` in step 1b or 1c, skip this prompt entirely.
- Check if `"calendar"` is in `ignoredSources`. If yes, silently mark Calendar as `unavailable` and continue.
- If not ignored, show:

> "⚠️ Google Calendar MCP isn't authenticated — calendar events won't be included.
>
> What would you like to do?
> **a) Ignore forever** — don't remind me about Google Calendar again
> **b) Ignore this time** — skip Calendar now, remind me next run
> **c) Fix it** — open Claude Code settings and authenticate the Google Calendar integration, then re-run `/recapper`"

If **a)**: add `"calendar"` to `ignoredSources` in config, mark as `unavailable`, continue.
If **b)** or **c)**: mark as `unavailable` and continue (no REST fallback available).

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
| Slack MCP not authenticated | MCP session expired | Re-authenticate via Claude Code settings, or run through Slack fallback setup |
| Linear shows no issues | MCP not authenticated | Re-authenticate via Claude Code settings, or run through Linear fallback setup |
| GitHub shows 0 events | `gh` CLI not logged in | Run `gh auth login` |
| Datadog 403 | Missing or wrong keys | Re-run `/recapper` — Phase 1 will prompt you to re-enter |
| Datadog audit returns no user events | Wrong user filter | Check `userEmail` field in audit response against your email |
| Google Calendar empty | MCP not authenticated | Re-authenticate via Claude Code settings |
| Notion 401 | Token invalid or MCP expired | Re-authenticate via Claude Code settings, or run through Notion fallback setup |
| Date format error | Wrong date format | Use `YYYY-MM-DD` — e.g. `2026-05-14` not `May 14` |
