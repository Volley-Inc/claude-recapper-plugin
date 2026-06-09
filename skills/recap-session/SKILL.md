---
name: recap-session
description: Log a summary of the current AI coding session so it's included in today's /recapper recap. Use at the end of a Cursor, Claude Code, VS Code, or any other AI-assisted coding session. Triggers on "recap session", "log session", "recap-session", "log this session", "capture this session", "summarize this session".
version: 1.1.0
---

# Recap Session

Log a summary of the current AI coding session to `~/.config/recapper/sessions/YYYY-MM-DD.json`. Entries are automatically picked up by `/recapper` as a 7th source when you run your daily recap.

## Usage

```
/recap-session [date]
```

**Arguments:**
- `date` (optional): Target date in `YYYY-MM-DD` format. Defaults to today. Use this to backfill a session you forgot to log the previous day.

Run this at the end of any AI coding session — Cursor, Claude Code, VS Code, or any IDE where you've been working with an AI assistant. You can run it multiple times per day; each call appends a new entry.

---

## Workflow

### Step 1: Resolve session file path

```bash
SESSION_DATE="${1:-$(date +%Y-%m-%d)}"
# Validate format and that the date is real (mirrors /recapper validation)
PARSED=$(date -d "$SESSION_DATE" +%Y-%m-%d 2>/dev/null || \
         date -j -f "%Y-%m-%d" "$SESSION_DATE" +%Y-%m-%d 2>/dev/null)
if [[ ! "$SESSION_DATE" =~ ^[0-9]{4}-[0-9]{2}-[0-9]{2}$ ]] || [ "$PARSED" != "$SESSION_DATE" ]; then
  echo "Error: invalid date '$SESSION_DATE'. Use YYYY-MM-DD — e.g. $(date +%Y-%m-%d)"
  exit 1
fi
SESSION_DIR="${HOME}/.config/recapper/sessions"
SESSION_FILE="${SESSION_DIR}/${SESSION_DATE}.json"
mkdir -p "$SESSION_DIR"
```

### Step 2: Summarize the session

Review the current conversation and produce a structured summary. Ask the user to confirm or adjust before saving:

> "Here's a summary of what we worked on this session:
>
> **Title:** {concise title — what was the main thing accomplished or explored?}
> **Description:** {1–3 sentences — what was done, what was found, what changed?}
> **Type:** {one of: feature, bugfix, refactor, investigation, review, planning, documentation, configuration, learning}
> **Category:** {one of: shipped, in_progress, collaborated, incident, planned}
>
> Does this look right? You can adjust any field or say 'looks good' to save."

[Wait for user input. Apply any corrections, then proceed to save.]

**Guidelines for summarizing:**
- **Title**: Action-oriented, specific. "Fixed Datadog session export bug" not "Worked on recapper"
- **Description**: What was actually done or discovered, not just what was discussed. Include the outcome.
- **Type**: Pick the closest match:
  - `feature` — new functionality built or designed
  - `bugfix` — fixed a broken thing
  - `refactor` — restructured without behavior change
  - `investigation` — researched, debugged, explored a problem
  - `review` — reviewed code, design, or a PR
  - `planning` — roadmap, scoping, design discussions
  - `documentation` — wrote docs, READMEs, specs
  - `configuration` — setup, tooling, env, infrastructure
  - `learning` — read, experimented, understood something new
- **Category**: Pick the closest match:
  - `shipped` — completed, merged, done
  - `in_progress` — started but not finished
  - `collaborated` — worked with others (reviews, discussions)
  - `incident` — debugged or responded to a production issue
  - `planned` — scoped or designed but not started

### Step 3: Append to the session file

Read the existing file (if it exists) and append the new entry:

```bash
# Read existing entries — fall back to [] if file is missing, empty, invalid JSON, or not an array
EXISTING=$(jq 'if type == "array" then . else [] end' "$SESSION_FILE" 2>/dev/null || echo "[]")
[ -z "$EXISTING" ] && EXISTING="[]"

# Append new entry (agent fills in actual values)
LOGGED_AT=$(date -u +%Y-%m-%dT%H:%M:%SZ)
ENTRY_ID="session-${LOGGED_AT}-$(LC_ALL=C tr -dc 'a-z0-9' < /dev/urandom | head -c 6)"
tmp="$(mktemp)"
echo "$EXISTING" | jq --arg id "$ENTRY_ID" \
  --arg title "TITLE" \
  --arg description "DESCRIPTION" \
  --arg type "TYPE" \
  --arg category "CATEGORY" \
  --arg logged_at "$LOGGED_AT" \
  '. += [{id: $id, title: $title, description: $description, type: $type, category: $category, logged_at: $logged_at}]' \
  > "$tmp" && mv "$tmp" "$SESSION_FILE"
```

Replace `TITLE`, `DESCRIPTION`, `TYPE`, and `CATEGORY` with the confirmed values.

### Step 4: Confirm

Tell the user:

> "✅ Session logged to `{SESSION_FILE}`.
>
> Run `/recapper{date_arg}` to include this in your recap. You can run `/recap-session` again after your next session to add more entries."

Where `{date_arg}` is ` {SESSION_DATE}` if `SESSION_DATE` differs from today's date, or empty if it is today. For example, if backfilling yesterday: "Run `/recapper 2026-06-08`".

---

## Output Format

Each entry in `~/.config/recapper/sessions/YYYY-MM-DD.json`:

```json
{
  "id": "session-2026-06-09T17:32:00Z-a4f3k2",
  "title": "Fixed Datadog session export bug",
  "description": "Identified that freshly entered Datadog keys weren't being exported to the current session before curl verification. Added explicit session exports per key in the Fix-it wizard.",
  "type": "bugfix",
  "category": "shipped",
  "logged_at": "2026-06-09T17:32:00Z"
}
```

The file is a JSON array — multiple entries per day are supported.

---

## Notes

- The sessions file is **local only** — it's never committed to git or sent anywhere
- You can delete it after running `/recapper` if you don't want it to persist
- You can manually edit the file if you want to adjust an entry after logging
- If you forget to run `/recap-session`, you can run it the next morning for the previous day — `/recapper` reads the file by date, not by when it was written
