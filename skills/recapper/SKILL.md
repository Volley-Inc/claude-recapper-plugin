---
name: recapper
description: Use when the user wants to recap their day, collect daily work activity, generate a daily summary, or populate their Impact Archive. Collects activity from Slack, Linear, GitHub, Notion, Datadog, and Google Calendar and outputs a conversational journal summary plus structured JSON. Triggers on "recap my day", "daily recap", "what did I do today", "impact archive", "recapper", or "/recapper".
version: 1.1.0
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

## Using with /recap-session

For work done in Cursor, VS Code, Claude Code, or any other AI coding tool, run `/recap-session` at the end of each session to log a structured summary. Those entries are automatically picked up as a 7th source when you run `/recapper`. You can run `/recap-session` multiple times per day — each call appends a new entry to `~/.config/recapper/sessions/YYYY-MM-DD.json`.

## Workflow

```
Phase 1: Setup          → Resolve date, check config, prompt for anything missing
Phase 2: Parallel Fetch → Collect from up to 7 sources concurrently (Slack, Linear, GitHub, Notion, Datadog, Google Calendar, AI Sessions)
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
# Validate format and that the date is real (date normalizes invalid dates like Feb 30,
# so check that the parsed result matches the input)
PARSED=$(date -d "$TARGET_DATE" +%Y-%m-%d 2>/dev/null || \
         date -j -f "%Y-%m-%d" "$TARGET_DATE" +%Y-%m-%d 2>/dev/null)
if [[ ! "$TARGET_DATE" =~ ^[0-9]{4}-[0-9]{2}-[0-9]{2}$ ]] || [ -z "$NEXT_DAY" ] || [ "$PARSED" != "$TARGET_DATE" ]; then
  echo "Error: invalid date '$TARGET_DATE'. Use YYYY-MM-DD — e.g. $(date +%Y-%m-%d)"
  exit 1
fi
```

### 1b. Load ignored-sources config

```bash
RECAPPER_CONFIG="${HOME}/.config/recapper/config.json"
mkdir -p "${HOME}/.config/recapper"
if [ ! -f "$RECAPPER_CONFIG" ]; then
  echo '{"ignoredSources":[],"calendarIds":[],"slackIncludeDMs":true,"onboardingComplete":false,"sessionReminder":null}' > "$RECAPPER_CONFIG"
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
> For each source, reply with a number:
> **1** — include this source every run
> **2** — exclude this run, include automatically next run
> **3** — never include this source"

Prompt for each source **individually**. **Prompt all six sources regardless of their current `ignoredSources` status** — this is the user's opportunity to change their mind from any prior partial run. Track all config changes in context — they will be written together in a single bash command at the end of onboarding. Do **not** write to `$RECAPPER_CONFIG` during these prompts. Answer **1** or **2** overrides step 1b's unavailable marking for that source (in-memory only — no bash required).

**Slack:**

> "**Slack** [1/2/3]:"

[Wait for input. Track this choice — do not write to config yet:
- **1**: track `"slack"` for removal from `ignoredSources`; mark as available.
- **2**: mark as unavailable for this run only; track `"slack"` for removal from `ignoredSources`.
- **3**: track `"slack"` for addition to `ignoredSources`; mark as unavailable.]

If **1** or **2**, immediately follow up with the DM preference (skip users will have Slack included on their next run and need this set):

> "**Include Direct Messages?** Should DMs appear in your Slack recap?
> 1) Yes — include DMs alongside channel messages
> 2) No — channel messages only (recommended for work recaps)
> 3) Ask me each time I run /recapper"

[Wait for input. Track the DM preference — do not write to config yet.]

Do **not** ask this for **3** — Slack will never be fetched.

**Linear:**

> "**Linear** [1/2/3]:"

[Wait for input. Track this choice — do not write to config yet:
- **1**: track `"linear"` for removal from `ignoredSources`; mark as available.
- **2**: mark as unavailable for this run only; track `"linear"` for removal from `ignoredSources`.
- **3**: track `"linear"` for addition to `ignoredSources`; mark as unavailable.]

**GitHub:**

> "**GitHub** [1/2/3]:"

[Wait for input. Track this choice — do not write to config yet:
- **1**: track `"github"` for removal from `ignoredSources`; mark as available.
- **2**: mark as unavailable for this run only; track `"github"` for removal from `ignoredSources`.
- **3**: track `"github"` for addition to `ignoredSources`; mark as unavailable.]

**Notion:**

> "**Notion** [1/2/3]:"

[Wait for input. Track this choice — do not write to config yet:
- **1**: track `"notion"` for removal from `ignoredSources`; mark as available.
- **2**: mark as unavailable for this run only; track `"notion"` for removal from `ignoredSources`.
- **3**: track `"notion"` for addition to `ignoredSources`; mark as unavailable.]

**Datadog:**

> "**Datadog** [1/2/3]:"

[Wait for input. Track this choice — do not write to config yet:
- **1**: track `"datadog"` for removal from `ignoredSources`; mark as available.
- **2**: mark as unavailable for this run only; track `"datadog"` for removal from `ignoredSources`.
- **3**: track `"datadog"` for addition to `ignoredSources`; mark as unavailable.]

**Google Calendar:**

> "**Google Calendar** [1/2/3]:"

[Wait for input. Track this choice — do not write to config yet:
- **1**: track `"calendar"` for removal from `ignoredSources`; mark as available.
- **2**: mark as unavailable for this run only; track `"calendar"` for removal from `ignoredSources`.
- **3**: track `"calendar"` for addition to `ignoredSources`; mark as unavailable.]

If **1** for Google Calendar and the Calendar MCP is available, call `mcp__claude_ai_Google_Calendar__list_calendars` and show:

> "You have access to the following calendars:
> [list each calendar with a number, e.g. "1. Work (primary)", "2. Team Meetings", "3. Personal"]
>
> Which would you like to include in your recaps? Enter the numbers separated by commas (or press Enter to use your primary calendar only):"

[Wait for input. Parse the numbers and track the selected calendar IDs — do not write to config yet.]

If the user presses Enter without selecting, track only the primary calendar ID. If the MCP is unavailable, skip calendar selection and track only the primary calendar ID.

**Session Log Reminder** (only if `sessionReminder` is `null` — skip if it's already set from a prior interrupted run):

> "Would you like a reminder to log your AI coding sessions before each recap? Running `/recap-session` at the end of a Cursor, VS Code, or Claude Code session captures work done in those tools so it shows up in your daily recap.
> 1) Yes — remind me if no sessions are logged when I run /recapper
> 2) No — I'll manage this myself"

[Wait for input. Track the session reminder preference — do not write to config yet.]

After all six sources are answered **and the Session Log Reminder preference is noted**, write all onboarding choices to config in **one consolidated bash command** — this is the only config write during onboarding. Substitute the actual values from the answers collected above:

```bash
tmp="$(mktemp)" && jq \
  --argjson ignoredSources '[]' \
  --arg slackIncludeDMs "true" \
  --argjson calendarIds '[]' \
  --argjson sessionReminder 'true' \
  '.ignoredSources = $ignoredSources | .slackIncludeDMs = (if $slackIncludeDMs == "true" then true elif $slackIncludeDMs == "false" then false else $slackIncludeDMs end) | .calendarIds = $calendarIds | .sessionReminder = $sessionReminder | .onboardingComplete = true' \
  "$RECAPPER_CONFIG" > "$tmp" && mv "$tmp" "$RECAPPER_CONFIG"
```

Substitute these values:
- `$ignoredSources` — JSON array of source slugs that received answer **3** (e.g., `'["datadog","calendar"]'`). Sources with answers **1** or **2** are excluded from this list.
- `$slackIncludeDMs` — the DM preference as a string: `"true"`, `"false"`, or `"ask"`. If Slack was answered **3**, read the existing value: `jq -r '.slackIncludeDMs // "true" | tostring' "$RECAPPER_CONFIG"`.
- `$calendarIds` — JSON array of selected calendar IDs (e.g., `'["primary","cal123@group.calendar.google.com"]'`). If Calendar was answered **2** or **3**, preserve the existing value: run `jq '.calendarIds // []' "$RECAPPER_CONFIG"` and use that array.
- `$sessionReminder` — `true` or `false` (JSON boolean, no quotes). If the Session Log Reminder prompt was skipped because `sessionReminder` was already set, read the existing value: `jq '.sessionReminder' "$RECAPPER_CONFIG"`.

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

If the command fails or reports "not logged in", show:

> "⚠️ GitHub CLI isn't authenticated — GitHub activity won't be included.
>
> What would you like to do?
> 1) Ignore forever — don't remind me about GitHub again
> 2) Ignore this time — skip GitHub now, remind me next run
> 3) Fix it — I'll walk you through logging in"

If **1**: add `"github"` to `ignoredSources` in config, mark as `unavailable`, continue.
If **2**: mark as `unavailable`, continue.
If **3**: tell the user:
> "Run this in your terminal and follow the prompts, then re-run `/recapper` to include GitHub:
> ```
> gh auth login
> ```"
Mark as `unavailable` and continue — the rest of this recap will run without GitHub.

### 1f. Check Datadog keys

If Datadog was already marked `unavailable` in step 1b or 1c, skip this step entirely.

```bash
echo "${DATADOG_API_KEY:+set}" && echo "${DATADOG_APP_KEY:+set}" && echo "${DATADOG_USER_EMAIL:+set}"
```

If any of the three are missing, show:

> "⚠️ Datadog isn't configured — dashboards, monitors, and incidents won't be included.
>
> What would you like to do?
> 1) Ignore forever — don't remind me about Datadog again
> 2) Ignore this time — skip Datadog now, remind me next run
> 3) Fix it — I'll walk you through getting your API keys"

If **1**: add `"datadog"` to `ignoredSources` in config, mark as `unavailable`, continue — do NOT proceed to the key prompts below.
If **2**: mark as `unavailable`, continue — do NOT proceed to the key prompts below.
If **3**: only prompt for keys that are actually missing — skip any step whose key is already set in the environment. Set `DATADOG_KEYS_JUST_COLLECTED=true` only after a key is successfully entered (not at the start of this flow):

If `DATADOG_API_KEY` is not set:

> "**Datadog API Key**
> 1. Go to **Datadog → Organization Settings → API Keys** (or ask your admin)
> 2. Click **New Key**, give it a name, and copy the value
>
> Paste your Datadog API Key here (or press Enter to skip Datadog):"

[Wait for user input. If empty, mark Datadog as `unavailable` and stop — do NOT prompt for remaining keys. If provided, set `DATADOG_API_KEY` to the entered value, export it for the current session, and set `DATADOG_KEYS_JUST_COLLECTED=true`.]

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

[Wait for user input. If empty, mark Datadog as `unavailable` and stop — do NOT prompt for remaining keys. If provided, set `DATADOG_APP_KEY` to the entered value, export it for the current session, and set `DATADOG_KEYS_JUST_COLLECTED=true`.]

If `DATADOG_USER_EMAIL` is not set:

> "**Your Datadog email address**
>
> This is the email you use to log in to Datadog. It's used to filter audit logs so you only see your own actions — not the whole org's.
>
> Paste your Datadog account email here (or press Enter to skip Datadog):"

[Wait for user input. If empty, mark Datadog as `unavailable` and continue. If provided, set `DATADOG_USER_EMAIL` to the entered value, export it for the current session, and set `DATADOG_KEYS_JUST_COLLECTED=true`.]

If Datadog is not already marked `unavailable`, verify the keys now. (`DATADOG_KEYS_JUST_COLLECTED` defaults to false if the Fix-it path was not taken.)

```bash
HTTP_STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
  "https://api.datadoghq.com/api/v2/audit/events?page[limit]=1" \
  -H "DD-API-KEY: $DATADOG_API_KEY" \
  -H "DD-APPLICATION-KEY: $DATADOG_APP_KEY" 2>/dev/null) || HTTP_STATUS="000"
```

- If `$HTTP_STATUS` is `200` or `403`:
  - If `403`: tell the user "⚠️ Keys authenticated but you may be missing the `audit_logs_read` scope — Datadog audit events won't appear in recaps, but everything else will still work. Continuing anyway."
  - If `DATADOG_KEYS_JUST_COLLECTED` is true (any key was entered in this session): tell the user "✅ Datadog keys verified!" and offer to save:

    > "Save these to your shell profile so you don't have to enter them again?
    > 1) Yes — append to shell profile
    > 2) No — use for this session only"

    If **1**, append to the shell profile (using `$SHELL_PROFILE` and `escape_sq` defined in 1d):

    ```bash
    printf '\n# Datadog (added by recapper)\n' >> "$SHELL_PROFILE"
    printf "export DATADOG_API_KEY='%s'\n" "$(escape_sq "$DATADOG_API_KEY")" >> "$SHELL_PROFILE"
    printf "export DATADOG_APP_KEY='%s'\n" "$(escape_sq "$DATADOG_APP_KEY")" >> "$SHELL_PROFILE"
    printf "export DATADOG_USER_EMAIL='%s'\n" "$(escape_sq "$DATADOG_USER_EMAIL")" >> "$SHELL_PROFILE"
    ```

    Then tell the user:
    > "Saved to `{SHELL_PROFILE}`. Run `source {SHELL_PROFILE}` to apply in other terminals."

    Also export the values for the current session so Phase 2 can use them immediately without restarting.

    If **2**, export the values for the current session so Phase 2 can use them.
  - If `DATADOG_KEYS_JUST_COLLECTED` is false (all keys were pre-existing): continue silently — no save prompt needed.

- If `$HTTP_STATUS` is `000` (curl failed): tell the user:
  > "⚠️ Couldn't reach Datadog — check your network connection."

  > "Paste corrected API key (or press Enter to skip Datadog):"

  [Wait for user input. If empty, mark Datadog as `unavailable` and continue — do NOT show the App key prompt below.]

  If a corrected API key was provided, set `DATADOG_API_KEY` to the entered value, export it for the current session, and set `DATADOG_KEYS_JUST_COLLECTED=true`, then:

  > "Paste corrected App key (or press Enter to skip Datadog):"

  [Wait for user input. If empty, mark Datadog as `unavailable` and continue. If provided, set `DATADOG_APP_KEY` to the entered value, export it for the current session, and re-run the HTTP status check above. If the re-check returns `000` again, tell the user "⚠️ Still can't reach Datadog — skipping for now. Re-run `/recapper` once your connection is restored.", mark Datadog as `unavailable`, and continue — do NOT loop again.]

- If `$HTTP_STATUS` is anything else (401, 400, etc.): tell the user:
  > "⚠️ Datadog keys don't seem valid (HTTP $HTTP_STATUS)."

  > "Paste corrected API key (or press Enter to skip Datadog):"

  [Wait for user input. If empty, mark Datadog as `unavailable` and continue — do NOT show the App key prompt below.]

  If a corrected API key was provided, set `DATADOG_API_KEY` to the entered value, export it for the current session, and set `DATADOG_KEYS_JUST_COLLECTED=true`, then:

  > "Paste corrected App key (or press Enter to skip Datadog):"

  [Wait for user input. If empty, mark Datadog as `unavailable` and continue. If provided, set `DATADOG_APP_KEY` to the entered value, export it for the current session, and re-run the HTTP status check above.]

### 1g. Session log reminder

If `sessionReminder` is `null` (not yet set — existing install that predates this feature), prompt once and save the answer now, then apply it for this run:

> "Would you like a reminder to log your AI coding sessions before each recap? Run `/recap-session` at the end of a Cursor, VS Code, or Claude Code session to capture that work.
> 1) Yes — remind me if no sessions are logged when I run /recapper
> 2) No — I'll manage this myself"

[Wait for input. Save preference to config:]

```bash
# If 1:
tmp="$(mktemp)" && jq '.sessionReminder = true' "$RECAPPER_CONFIG" > "$tmp" && mv "$tmp" "$RECAPPER_CONFIG"
# If 2:
tmp="$(mktemp)" && jq '.sessionReminder = false' "$RECAPPER_CONFIG" > "$tmp" && mv "$tmp" "$RECAPPER_CONFIG"
```

If `sessionReminder` is `true` in config, check whether the session file for `$TARGET_DATE` has any entries:

```bash
SESSION_FILE="${HOME}/.config/recapper/sessions/${TARGET_DATE}.json"
SESSION_COUNT=$(jq 'if type == "array" then length else 0 end' "$SESSION_FILE" 2>/dev/null || echo "0")
```

If `SESSION_COUNT` is `0` (file missing or empty), show:

> "💡 No session logs found for **{TARGET_DATE}**. If you've been working in Cursor, VS Code, or another AI coding tool, run `/recap-session{date_arg}` to capture that work before your recap.
> 1) Continue anyway
> 2) Wait — I'll run /recap-session first"

Where `{date_arg}` is ` {TARGET_DATE}` if it differs from today, or empty if it is today.

[Wait for input. If **2**: stop here so the user can run `/recap-session` first. If **1** or empty: continue.]

If `SESSION_COUNT` is greater than `0`, or `sessionReminder` is `false`: skip this step silently. (The `null` case is handled above and will not reach this point.)

### 1h. Announce

Check for session entries independently of the `sessionReminder` preference (users may have run `/recap-session` even if reminders are off):

```bash
SESSION_FILE="${HOME}/.config/recapper/sessions/${TARGET_DATE}.json"
SESSION_COUNT=$(jq 'if type == "array" then length else 0 end' "$SESSION_FILE" 2>/dev/null || echo "0")
```

Build the source list from only the sources not already marked `unavailable` after steps 1b–1f. Include "AI Sessions" in the list if `SESSION_COUNT` is greater than `0`. Then announce:

> "Collecting activity for **{TARGET_DATE}**. Fetching from {comma-separated list of available sources}..."

For example, if GitHub and Datadog were skipped but sessions were logged:
> "Collecting activity for **2026-05-14**. Fetching from Slack, Linear, Notion, Google Calendar, and AI Sessions..."

---

## Phase 2: Parallel Fetch

**Before fetching anything**, skip any source already marked `unavailable` from Phase 1 (steps 1b–1f) — do not attempt to fetch, fall back, or prompt for it. This ensures sources the user ignored forever or skipped this time are never contacted, even when credentials or MCP sessions happen to be present.

For sources that are available: run independent fetches concurrently where possible. If an MCP tool is unavailable or returns an error, fall back to the REST API. If both fail, mark the source as `unavailable`, show the relevant setup guidance below, and continue — never halt on a single source failure.

### 2a. Slack

Read the DM preference from config:
```bash
SLACK_INCLUDE_DMS=$(jq -r 'if .slackIncludeDMs == false then "false" elif .slackIncludeDMs == "ask" then "ask" else "true" end' "$RECAPPER_CONFIG" 2>/dev/null || echo "true")
```

If `SLACK_INCLUDE_DMS` is `"ask"`, prompt the user now:

> "**Include Direct Messages in today's recap?**
> 1) Yes — include DMs alongside channel messages
> 2) No — channel messages only"

[Wait for input. Resolve `SLACK_INCLUDE_DMS` to `"true"` if **1**, `"false"` if **2** — do not save to config. All subsequent steps use this resolved value.]

**Preferred: MCP**
- If `SLACK_INCLUDE_DMS` is `"true"`: use `mcp__claude_ai_Slack__slack_search_public_and_private`
- If `SLACK_INCLUDE_DMS` is `"false"`: use `mcp__claude_ai_Slack__slack_search_public` (channel messages only)

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

**On MCP failure and REST fallback failure** (whether env vars are missing or set but invalid):
- Check if `"slack"` is in `ignoredSources`. If yes, silently mark Slack as `unavailable` and continue.
- If not ignored, show:

> "⚠️ Slack MCP isn't available and the REST fallback failed (missing or invalid credentials) — Slack messages won't be included.
>
> What would you like to do?
> 1) Ignore forever — don't remind me about Slack again
> 2) Ignore this time — skip Slack now, remind me next run
> 3) Fix it — walk me through setting up Slack"

If **1**: add `"slack"` to `ignoredSources` in config, mark as `unavailable`, continue.
If **2**: mark as `unavailable`, continue.
If **3**: ask the user to choose:

> "You can fix this two ways:
>
> 1) Authenticate the Slack MCP (recommended) — Open Claude Code settings and authenticate the Slack integration, then re-run `/recapper`.
> 2) Set up the REST fallback — I'll collect your Slack credentials now.
>
> Enter 1 or 2 (or press Enter to skip Slack):"

[Wait for input. If empty: mark Slack as `unavailable` and continue. If **1**: mark as `unavailable` and continue — the rest of this recap will run without Slack; user can re-run after authenticating. If **2**: proceed below.]

If user chose **2**, re-collect both values — since REST already failed, any existing values may be invalid:

> "Your Slack User ID (open your profile → **•••** → **Copy member ID**, looks like `U012AB3CD`):
> Paste here (or press Enter to skip Slack):"

[Wait for user input. If empty, mark Slack as `unavailable` and continue — do NOT collect the token below. If provided, set `SLACK_USER_ID` to the entered value, export it for the current session, and set `SLACK_CREDS_JUST_COLLECTED=true`.]

> "Your Slack User OAuth Token with `search:read` scope (starts with `xoxp-`):
> Create a Slack app at api.slack.com/apps → OAuth & Permissions → User Token Scopes → add `search:read` → install → copy User OAuth Token.
>
> Paste here (or press Enter to skip Slack):"

[Wait for user input. If empty, mark Slack as `unavailable` and continue — do NOT proceed to the save and availability steps below. If provided, set `SLACK_BOT_TOKEN` to the entered value, export it for the current session, and set `SLACK_CREDS_JUST_COLLECTED=true`.]

If both SLACK_USER_ID and SLACK_BOT_TOKEN are now set **and `SLACK_CREDS_JUST_COLLECTED` is true** (at least one was entered this session), offer to save:

> "Save these to your shell profile so you don't have to enter them again?
> 1) Yes — append to shell profile
> 2) No — use for this session only"

If **1**, append to the shell profile (using `$SHELL_PROFILE` and `escape_sq` defined in 1d):

```bash
printf '\n# Slack (added by recapper)\n' >> "$SHELL_PROFILE"
printf "export SLACK_USER_ID='%s'\n" "$(escape_sq "$SLACK_USER_ID")" >> "$SHELL_PROFILE"
printf "export SLACK_BOT_TOKEN='%s'\n" "$(escape_sq "$SLACK_BOT_TOKEN")" >> "$SHELL_PROFILE"
```

Then tell the user:
> "Saved to `{SHELL_PROFILE}`. Run `source {SHELL_PROFILE}` to apply in other terminals."

Also export the values for the current session so Phase 2 can use them immediately.

If **2**, export the values for the current session so Phase 2 can use them.

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

**On MCP failure and GraphQL fallback failure** (whether `LINEAR_API_KEY` is missing or set but invalid):
- Check if `"linear"` is in `ignoredSources`. If yes, silently mark Linear as `unavailable` and continue.
- If not ignored, show:

> "⚠️ Linear MCP isn't available and the GraphQL fallback failed (missing or invalid `LINEAR_API_KEY`) — Linear issues won't be included.
>
> What would you like to do?
> 1) Ignore forever — don't remind me about Linear again
> 2) Ignore this time — skip Linear now, remind me next run
> 3) Fix it — walk me through setting up Linear"

If **1**: add `"linear"` to `ignoredSources` in config, mark as `unavailable`, continue.
If **2**: mark as `unavailable`, continue.
If **3**: ask the user to choose:

> "You can fix this two ways:
>
> 1) Authenticate the Linear MCP (recommended) — Open Claude Code settings and authenticate the Linear integration, then re-run `/recapper`.
> 2) Set up the REST fallback — I'll collect your API key now.
>
> Enter 1 or 2 (or press Enter to skip Linear):"

[Wait for input. If empty: mark Linear as `unavailable` and continue. If **1**: mark as `unavailable` and continue — the rest of this recap runs without Linear; user can re-run after authenticating. If **2**: proceed below.]

> "Open Linear → **Settings → API → Personal API keys** → Create key → copy value.
>
> Paste your Linear API key here (or press Enter to skip):"

[Wait for user input. If empty, mark Linear as `unavailable` and continue. If provided, set `LINEAR_API_KEY` to the entered value and export it for the current session.]

If provided, offer to save to shell profile:

> "Save this to your shell profile so you don't have to enter it again?
> 1) Yes — append to shell profile
> 2) No — use for this session only"

If **1**, append to the shell profile (using `$SHELL_PROFILE` and `escape_sq` defined in 1d):
```bash
printf '\n# Linear (added by recapper)\n' >> "$SHELL_PROFILE"
printf "export LINEAR_API_KEY='%s'\n" "$(escape_sq "$LINEAR_API_KEY")" >> "$SHELL_PROFILE"
```

Then tell the user:
> "Saved to `{SHELL_PROFILE}`. Run `source {SHELL_PROFILE}` to apply in other terminals."

Also export the value for the current session so Phase 2 can use it immediately.

If **2**, export for the current session only.

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

**On MCP failure and REST fallback failure** (whether `NOTION_TOKEN` is missing or set but invalid):
- Check if `"notion"` is in `ignoredSources`. If yes, silently mark Notion as `unavailable` and continue.
- If not ignored, show:

> "⚠️ Notion MCP isn't available and the REST fallback failed (missing or invalid `NOTION_TOKEN`) — Notion pages won't be included.
>
> What would you like to do?
> 1) Ignore forever — don't remind me about Notion again
> 2) Ignore this time — skip Notion now, remind me next run
> 3) Fix it — walk me through setting up Notion"

If **1**: add `"notion"` to `ignoredSources` in config, mark as `unavailable`, continue.
If **2**: mark as `unavailable`, continue.
If **3**: ask the user to choose:

> "You can fix this two ways:
>
> 1) Authenticate the Notion MCP (recommended) — Open Claude Code settings and authenticate the Notion integration, then re-run `/recapper`.
> 2) Set up the REST fallback — I'll collect your token now.
>
> Enter 1 or 2 (or press Enter to skip Notion):"

[Wait for input. If empty: mark Notion as `unavailable` and continue. If **1**: mark as `unavailable` and continue — the rest of this recap runs without Notion; user can re-run after authenticating. If **2**: proceed below.]

> "Go to [notion.so/my-integrations](https://www.notion.so/my-integrations) → New integration → copy the Internal Integration Token (starts with `secret_`).
>
> Paste your Notion token here (or press Enter to skip):"

[Wait for user input. If empty, mark Notion as `unavailable` and continue. If provided, set `NOTION_TOKEN` to the entered value and export it for the current session.]

If provided, offer to save to shell profile:

> "Save this to your shell profile so you don't have to enter it again?
> 1) Yes — append to shell profile
> 2) No — use for this session only"

If **1**, append to the shell profile (using `$SHELL_PROFILE` and `escape_sq` defined in 1d):
```bash
printf '\n# Notion (added by recapper)\n' >> "$SHELL_PROFILE"
printf "export NOTION_TOKEN='%s'\n" "$(escape_sq "$NOTION_TOKEN")" >> "$SHELL_PROFILE"
```

Then tell the user:
> "Saved to `{SHELL_PROFILE}`. Run `source {SHELL_PROFILE}` to apply in other terminals."

Also export the value for the current session so Phase 2 can use it immediately.

If **2**, export for the current session only.

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

If the filtered result is empty but the raw response contains audit events (i.e., there is activity but none matched your email), tell the user:
> "⚠️ Datadog audit events were found but none matched `$DATADOG_USER_EMAIL`. If your Datadog login email is different, correct it with:
> ```
> export DATADOG_USER_EMAIL=correct@example.com
> ```
> then re-run `/recapper`."

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
> 1) Ignore forever — don't remind me about Google Calendar again
> 2) Ignore this time — skip Calendar now, remind me next run
> 3) Fix it — open Claude Code settings and authenticate the Google Calendar integration, then re-run `/recapper`"

If **1**: add `"calendar"` to `ignoredSources` in config, mark as `unavailable`, continue.
If **2**: mark as `unavailable` and continue.
If **3**: tell the user to authenticate the Google Calendar integration in Claude Code settings, then re-run `/recapper` to include Calendar. Mark as `unavailable` and continue — the rest of this recap will run without Calendar.

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

### 2g. AI Sessions

Read the session log file for `$TARGET_DATE`:

```bash
SESSION_FILE="${HOME}/.config/recapper/sessions/${TARGET_DATE}.json"
```

Check entry count first, then set the flag and parse only if entries exist:

```bash
SESSIONS_FOUND=false
SESSION_COUNT=$(jq 'if type == "array" then length else 0 end' "$SESSION_FILE" 2>/dev/null || echo "0")
if [ "$SESSION_COUNT" -gt 0 ]; then
  SESSIONS_FOUND=true
  jq 'if type == "array" then .[] else empty end' "$SESSION_FILE" 2>/dev/null || true
fi
```

If `SESSION_COUNT` is `0` (file missing, empty, or non-array), skip this source silently — no prompt, no error.

Each entry has: `id`, `title`, `description`, `type`, `category`, `logged_at`.

Map each entry to a contribution:

```json
{
  "id": "{id from entry}",
  "date": "{TARGET_DATE}",
  "source": "ai_session",
  "sources": ["ai_session"],
  "type": "{type from entry}",
  "category": "{category from entry}",
  "title": "{title from entry}",
  "description": "{description from entry}",
  "url": null,
  "metadata": {
    "logged_at": "{logged_at from entry}"
  }
}
```

---

## Phase 3: Synthesize

After all fetches complete:

### 3a. Deduplicate

The same work may surface in multiple sources (e.g., a merged PR appears in GitHub events AND may be mentioned in a Slack message). Identify duplicates by:
- Matching URLs across sources
- Matching PR/issue titles with high similarity
- Consolidate into a single contribution entry with `sources: ["github", "slack"]`

**`ai_session` entries are never deduplicated** — check `source === "ai_session"` before applying any dedup logic and skip those entries entirely. Each `/recap-session` entry is intentionally distinct; do not merge them with each other or with entries from any other source, even if titles appear similar.

### 3b. Classify each item

Assign a `category` to every contribution **except `ai_session` source entries** — check `source === "ai_session"` and pass those through unchanged. Their `category` field was confirmed by the user in `/recap-session` and must not be reclassified:

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

> **Note:** Carry `SESSIONS_FOUND` forward from Phase 2g — you will need it in the cleanup step below.

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
      "content": "...",
      "url": "https://..."
    }
  ]
}
```

### Phase 4 cleanup

After both the summary and JSON have been written successfully, if `SESSIONS_FOUND=true` (set in Phase 2g when entries were actually parsed), offer to delete it:

> "Session log used. Delete `~/.config/recapper/sessions/{TARGET_DATE}.json` to keep things tidy?
> 1) Yes — delete it
> 2) No — keep it"

[Wait for input.]

If **1**:
```bash
rm -f "${HOME}/.config/recapper/sessions/${TARGET_DATE}.json"
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
