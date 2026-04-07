---
name: memory-orchestrator
description: Multi-layer memory system for OpenClaw agents with automatic Dream consolidation. Orchestrates LCM (lossless context), Mem0 (semantic recall), QMD (full-text search), task files, and daily logs into a coherent five-layer memory architecture with periodic self-maintenance. Use when setting up a new OpenClaw instance's memory system, when the user asks about memory/context management, or when configuring plugins for persistent agent memory. Covers plugin installation, config, cron jobs, Dream consolidation, file conventions, and cross-platform (WhatsApp/webchat) context continuity.
---

# Memory Orchestrator

Five-layer memory system that gives OpenClaw agents persistent, cross-session, cross-platform memory — with automatic Dream consolidation to prevent memory bloat.

## Architecture Overview

```
Layer 1: LCM (session-internal)    → auto DAG summaries, never loses context
Layer 2: Task files (cross-session) → dense snapshots of active work
Layer 3: Daily logs (cross-session)  → daily event journal
Layer 4: Mem0 (cross-session)        → semantic fragment recall
Layer 5: MEMORY.md (long-term)       → curated high-level overview
```

Each layer serves a different density/scope tradeoff. See `references/architecture.md` for the full design.

## Quick Setup

### 1. Install plugins

```bash
# Lossless Context Management (session-internal, DAG summaries)
openclaw plugins install @martian-engineering/lossless-claw

# Mem0 (cross-session semantic recall)
openclaw plugins install @mem0/openclaw-mem0
```

QMD ships with OpenClaw — no install needed.

### 2. Configure

Apply this config patch (adjust `summaryModel`, `userId`, `apiKey` to your setup):

```json
{
  "plugins": {
    "slots": {
      "contextEngine": "lossless-claw",
      "memory": "openclaw-mem0"
    },
    "entries": {
      "lossless-claw": {
        "enabled": true,
        "config": {
          "freshTailCount": 32,
          "contextThreshold": 0.75,
          "incrementalMaxDepth": -1,
          "ignoreSessionPatterns": ["agent:*:cron:**"],
          "summaryModel": "anthropic/claude-haiku-4-5"
        }
      },
      "openclaw-mem0": {
        "enabled": true,
        "config": {
          "apiKey": "<YOUR_MEM0_API_KEY>",
          "userId": "<YOUR_USER_ID>",
          "autoRecall": true,
          "autoCapture": true,
          "topK": 5
        }
      },
      "memory-core": { "enabled": false },
      "memory-lancedb": { "enabled": false }
    }
  },
  "memory": {
    "backend": "qmd",
    "qmd": {
      "searchMode": "search",
      "includeDefaultMemory": true,
      "sessions": { "enabled": true },
      "update": { "interval": "15m" }
    }
  },
  "agents": {
    "defaults": {
      "compaction": {
        "mode": "safeguard",
        "memoryFlush": { "enabled": true }
      }
    }
  }
}
```

### 3. File structure

Create these directories in your workspace:

```
workspace/
├── MEMORY.md              # Long-term curated memory (weekly updates)
├── memory/
│   ├── YYYY-MM-DD.md      # Daily logs
│   ├── tasks/             # Dense task snapshots
│   │   ├── project-a.md
│   │   └── project-b.md
│   └── heartbeat-state.json  # Heartbeat check timestamps
```

### 4. Cron jobs (recommended)

Set up these automated memory maintenance jobs:

- **Daily Memory Sync** (23:00) — Scan all sessions, write daily log, backfill missing task files. Use a cheap/fast model (e.g., Gemini Flash).
- **Weekly Compound** (Sunday 22:00) — Distill weekly logs into MEMORY.md updates.
- **Brain Index Update** (07:30) — Refresh QMD full-text search index.

See `references/cron-templates.md` for ready-to-use cron job definitions.

## Layer Details

### Layer 1: LCM (Lossless Context Management)

**What**: Every message persisted to SQLite. Old messages auto-summarized into a DAG. Context assembled each turn = summaries + recent raw messages.

**When it kicks in**: Context usage hits `contextThreshold` (default 75%).

**Agent tools**: `lcm_grep` (search history), `lcm_describe` (inspect summary), `lcm_expand` (drill into details).

**Key config**: See `references/lcm-tuning.md` for parameter explanations.

### Layer 2: Task Files

**What**: Dense markdown snapshots of active work, stored in `memory/tasks/`.

**Convention**:
- Create when user says "write task" or at end of substantial work sessions
- Naming: kebab-case (`ta-leaderboard.md`, `openvla-exp.md`)
- Load on-demand only (when task is mentioned)
- Mark `[DONE]` when complete, never delete
- Daily Sync auto-creates lightweight tasks for uncovered work

### Layer 3: Daily Logs

**What**: `memory/YYYY-MM-DD.md` — chronological event journal.

**Convention**:
- Load today + yesterday at session start
- Append throughout the day
- Source for weekly compound → MEMORY.md

### Layer 4: Mem0

**What**: Cloud-hosted semantic memory fragments. Auto-captured from every conversation turn, auto-recalled as `relevant-memories` injection.

**Tuning**: `topK` controls how many fragments inject per turn (default 5).

### Layer 5: MEMORY.md

**What**: Curated high-level overview — project status, lessons learned, weekly summaries.

**Convention**:
- Update weekly (manually or via Weekly Compound cron)
- Only load in main/private sessions (contains personal context)
- Periodically review and prune stale entries

## Cross-Platform Continuity

For setups with multiple chat surfaces (e.g., WhatsApp + webchat):

1. **Webchat idle reset**: Set `session.resetByChannel.webchat.idleMinutes` to 7+ days to prevent orphaned sessions
2. **Task handoff**: Complete deep work on webchat → "write task" → continue on WhatsApp seamlessly
3. **Daily Sync backfill**: Catches webchat work that wasn't manually task-filed

See `references/cross-platform.md` for the full WhatsApp + webchat cooperation model.

## Dream Consolidation

### OpenClaw Native Dreaming (2026.4.5+)

OpenClaw now ships with built-in dreaming under `memory-core`. Enable it with:

```json
{
  "plugins": {
    "entries": {
      "memory-core": {
        "enabled": true,
        "config": {
          "dreaming": { "enabled": true, "frequency": "0 4 * * *" }
        }
      }
    }
  }
}
```

Native dreaming handles: recall promotion (weighted short-term → long-term), decay aging, daily-note chunking, dreams.md output, and `/dreaming` command. It does NOT handle Mem0 pruning or MEMORY.md compression.

### Custom Dream Cron (v6) — Complements Native

For Mem0 noise cleanup, MEMORY.md line-count enforcement, and task file archival, the custom Dream cron is still valuable:

Without periodic maintenance, memory systems degrade:
- Mem0 fills with noise (gateway status, timestamps, exec logs) → recall quality drops
- MEMORY.md accumulates stale entries → bloated context on every session start
- Task files pile up after completion → wastes scanning time
- Daily logs from months ago never get reviewed → dead weight

### How Custom Dream Works

The Dream cycle runs weekly (or on-demand) with six steps:

1. **Orientation** — Read MEMORY.md, assess current state
2. **Gather Signal** — Scan recent daily logs + Mem0 for high-value memories
3. **Consolidation** — Update statuses, remove completed items, compress old reports, enforce absolute dates
4. **Mem0 Pruning** — Search and delete noise memories matching known patterns
5. **Task File Review** — Archive completed tasks to `memory/tasks/archive/`
6. **Index** — Ensure MEMORY.md stays under target line count, update timestamp

### Setup Dream Cron

See `references/cron-templates.md` for the ready-to-use Dream cron job definition.

### Mem0 Write Filtering Rules

Add these rules to your AGENTS.md to prevent noise from entering Mem0:

```markdown
### 🚫 Mem0 Write Filtering (hard rules)

DO NOT write to Mem0:
- Gateway connect/disconnect status (WhatsApp, WeChat, etc.)
- Current timestamps ("current time is...")
- Exec completed/failed status (belongs in session logs)
- Message IDs and conversation metadata
- Heartbeat check records
- Scheduled cleanup task triggers

DO write to Mem0:
- User preferences and behavior patterns
- Architecture decisions and technical choices
- Project status changes (new/completed/abandoned)
- Lessons learned and post-mortems
- Contact info and account details
```

### Absolute Date Rule

All memory files (MEMORY.md, task files, Mem0 entries) must use `YYYY-MM-DD` format. Never write "today", "yesterday", or "last week" — these become meaningless after the session ends.

## Maintenance

The Dream cron handles most maintenance automatically. For manual maintenance during heartbeats:
1. Review recent `memory/YYYY-MM-DD.md` files
2. Distill significant events into MEMORY.md
3. Check `memory/tasks/` for stale ACTIVE tasks (>3 days no update)
4. Prune outdated MEMORY.md entries

## Troubleshooting

- **LCM not triggering**: Check if context window is large enough (200k+ may rarely hit 75%). Lower `contextThreshold` to 0.6 for testing.
- **Mem0 not recalling**: Verify `autoRecall: true` and check Mem0 dashboard for stored memories.
- **Task files empty**: Daily Sync may not be running. Check cron job status.
- **Webchat context lost**: Verify `resetByChannel.webchat.idleMinutes` is set high enough.
