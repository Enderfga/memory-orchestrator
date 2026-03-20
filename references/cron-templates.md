# Cron Job Templates

Ready-to-use cron definitions for memory maintenance. Adapt the prompts and models to your setup.

## Daily Memory Sync (23:00)

Scans all sessions from the day, writes daily log, backfills missing task files.

```json
{
  "name": "Daily Memory Sync",
  "schedule": {
    "kind": "cron",
    "expr": "0 23 * * *",
    "tz": "Asia/Singapore"
  },
  "payload": {
    "kind": "agentTurn",
    "message": "Daily memory sync. 1) Use sessions_list to find today's active sessions. 2) For each session with substance, summarize key topics and outcomes. 3) Write/append to memory/YYYY-MM-DD.md. 4) Check if any work topics lack a task file in memory/tasks/ — if so, create a lightweight one. 5) Do NOT overwrite existing content in daily logs or task files.",
    "model": "google/gemini-3-flash-preview",
    "timeoutSeconds": 300
  },
  "sessionTarget": "isolated",
  "delivery": { "mode": "none" }
}
```

## Weekly Memory Compound (Sunday 22:00)

Distills the week's daily logs into MEMORY.md updates.

```json
{
  "name": "Weekly Memory Compound",
  "schedule": {
    "kind": "cron",
    "expr": "0 22 * * 0",
    "tz": "Asia/Singapore"
  },
  "payload": {
    "kind": "agentTurn",
    "message": "Weekly memory compound. 1) Read memory/YYYY-MM-DD.md for the past 7 days. 2) Read current MEMORY.md. 3) Identify significant events, decisions, lessons, and project status changes. 4) Append a new weekly summary section to MEMORY.md. 5) Remove outdated entries if appropriate. 6) Do NOT overwrite the entire file.",
    "timeoutSeconds": 300
  },
  "sessionTarget": "isolated",
  "delivery": { "mode": "none" }
}
```

## Brain Index Update (07:30)

Refreshes QMD full-text search index over workspace files.

```json
{
  "name": "Brain Index Update",
  "schedule": {
    "kind": "cron",
    "expr": "30 7 * * *",
    "tz": "Asia/Singapore"
  },
  "payload": {
    "kind": "agentTurn",
    "message": "Update the QMD brain index. Run: exec qmd update && qmd embed (if embed is available). Report any errors.",
    "model": "google/gemini-3-flash-preview",
    "timeoutSeconds": 120
  },
  "sessionTarget": "isolated",
  "delivery": { "mode": "none" }
}
```

## Notes

- **Timezone**: Adjust `tz` to your local timezone
- **Model**: Daily Sync and Brain Index use cheap models (Flash/Haiku). Weekly Compound can use the default (smarter) model for better distillation.
- **Delivery**: Set to `"announce"` if you want notifications when jobs complete
- **Timeout**: 300s is generous for Daily Sync; reduce if your sessions are light
