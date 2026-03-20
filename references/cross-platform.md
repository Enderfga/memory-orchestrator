# Cross-Platform Context Continuity

## Problem

When using multiple chat surfaces (WhatsApp, webchat, Discord, etc.), work done on one platform is invisible to conversations on another. User says "continue the leaderboard work" on WhatsApp, but that work happened in a webchat session.

## Solution: Three-Layer Bridge

### 1. Task Files (manual, highest quality)

```
Webchat: [deep work on leaderboard]
User: "write task"
Agent: creates memory/tasks/ta-leaderboard.md (dense snapshot)
---
WhatsApp: "continue leaderboard"
Agent: reads memory/tasks/ta-leaderboard.md → seamless continuation
```

Best for: substantial work sessions. User explicitly triggers the snapshot.

### 2. Daily Sync (automatic, medium quality)

The 23:00 cron job scans ALL sessions (including webchat) and writes daily logs. Even if the user forgot to "write task", the Daily Sync catches it.

Key technique for webchat: scan orphaned `.jsonl` session files that may not be indexed in `sessions.json` due to idle resets.

### 3. Mem0 (automatic, fragment quality)

Mem0 autoCapture runs on every turn across all platforms. Semantic fragments are available everywhere via autoRecall.

Best for: facts, preferences, small decisions. Not for detailed work context.

## Webchat Session Preservation

**Problem**: Default `session.reset.idleMinutes = 30` orphans webchat sessions after 30 minutes of inactivity. The session gets a new UUID, and the old conversation becomes invisible to the API.

**Fix**: 
```json
{
  "session": {
    "resetByChannel": {
      "webchat": {
        "mode": "idle",
        "idleMinutes": 10080
      }
    }
  }
}
```

This gives webchat sessions 7 days before reset, matching how users actually work (come back to a topic days later).

## Platform Strengths

| Platform | Best For | Memory Strategy |
|----------|----------|-----------------|
| WhatsApp/Signal | Quick questions, mobile, always-on | Rely on Mem0 + task files for context |
| Webchat | Deep work, long sessions | Native long context + LCM for compression |
| Discord | Group collaboration, threads | Thread-bound sessions, mention-triggered |

## Recommendations

1. Set webchat idle reset to 7+ days
2. Encourage "write task" habit after deep work sessions
3. Configure Daily Sync to scan orphaned session files
4. Use Mem0 as the universal semantic bridge
