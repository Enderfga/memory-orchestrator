# 🧠 Memory Orchestrator

A three-layer memory system for [OpenClaw](https://github.com/openclaw/openclaw) agents — persistent, cross-session, cross-platform context that never forgets, with automatic **Dream consolidation** to prevent memory bloat.

## Problem

OpenClaw agents lose context when:
- A long conversation exceeds the context window → old messages truncated
- You switch between chat platforms (WhatsApp → webchat) → separate sessions, no shared memory
- A new session starts → agent has no idea what happened before

## Solution

Memory Orchestrator coordinates three complementary memory layers:

| Layer | Scope | How | What It Does |
|-------|-------|-----|-------------|
| **LCM** | Session-internal | Auto DAG summaries | Compresses old messages into retrievable summaries |
| **Task Files + Daily Logs** | Cross-session | Manual + cron | Dense work snapshots + daily event journals |
| **MEMORY.md** | Long-term | Weekly curation | High-level overview, lessons learned, project status |

## Quick Start

### 1. Install plugins

```bash
# Lossless Context Management
openclaw plugins install @martian-engineering/lossless-claw
```

### 2. Configure

Add to your `openclaw.json` (via `openclaw configure` or manual edit):

```jsonc
{
  "plugins": {
    "slots": {
      "contextEngine": "lossless-claw"
    },
    "entries": {
      "lossless-claw": {
        "enabled": true,
        "config": {
          "freshTailCount": 32,           // keep last 32 messages raw
          "contextThreshold": 0.75,       // compress at 75% context usage
          "incrementalMaxDepth": -1,       // unlimited compression depth
          "summaryModel": "anthropic/claude-haiku-4-5"  // cheap model for summaries
        }
      },
      "memory-core": {
        "enabled": true,
        "config": {
          "dreaming": {
            "enabled": true,              // native dream consolidation (2026.4.5+)
            "frequency": "0 4 * * *"      // daily at 04:00
          }
        }
      }
    }
  }
}
```

### 3. Set up workspace files

```
your-workspace/
├── MEMORY.md              # curated long-term memory
└── memory/
    ├── 2026-03-20.md      # daily logs (auto-created)
    └── tasks/             # dense task snapshots
        ├── my-project.md
        └── another-task.md
```

### 4. Set up cron jobs

Two cron jobs keep the memory layers in sync. Without them, daily logs and MEMORY.md won't auto-update.

**Daily Memory Sync** (every night 23:00) — scans all sessions, writes `memory/YYYY-MM-DD.md`, backfills missing task files:

```bash
openclaw cron add \
  --name "Daily Memory Sync" \
  --cron "0 23 * * *" \
  --tz "Asia/Singapore" \
  --session isolated \
  --model "google/gemini-3-flash-preview" \
  --timeout-seconds 300 \
  --no-deliver \
  --message 'Daily memory sync. 1) Use sessions_list to find today'\''s active sessions. 2) For each session with substance, summarize key topics and outcomes. 3) Write/append to memory/YYYY-MM-DD.md. 4) Check if any work topics lack a task file in memory/tasks/ — if so, create a lightweight one. 5) Do NOT overwrite existing content.'
```

**Weekly Memory Compound** (Sunday 22:00) — distills the week's daily logs into MEMORY.md:

```bash
openclaw cron add \
  --name "Weekly Memory Compound" \
  --cron "0 22 * * 0" \
  --tz "Asia/Singapore" \
  --session isolated \
  --timeout-seconds 300 \
  --no-deliver \
  --message 'Weekly memory compound. 1) Read memory/YYYY-MM-DD.md for the past 7 days. 2) Read current MEMORY.md. 3) Identify significant events, decisions, lessons, and project status changes. 4) Append a new weekly summary section to MEMORY.md. 5) Remove outdated entries if appropriate. 6) Do NOT overwrite the entire file.'
```

> Adjust `--tz` to your timezone. See [`references/cron-templates.md`](references/cron-templates.md) for JSON format templates.

## How It Works

```
┌─────────────────────────────────────────────────┐
│                  Each Turn                       │
│                                                  │
│  ┌──────────┐  ┌──────────┐                     │
│  │   LCM    │  │  Daily   │                     │
│  │ summaries│  │  logs    │                     │
│  │ + recent │  │ today+   │                     │
│  │ messages │  │ yesterday│                     │
│  └────┬─────┘  └────┬─────┘                     │
│       └──────────────┘                           │
│                  ▼                               │
│          Agent Context Window                    │
│                                                  │
│  Task files loaded on-demand when mentioned      │
│  MEMORY.md loaded at session start (private)     │
└─────────────────────────────────────────────────┘
```

When the context window fills up, LCM automatically:
1. Groups old messages into chunks
2. Summarizes each chunk (using a cheap model like Haiku)
3. Builds a DAG of summaries (summaries of summaries)
4. Replaces old messages with compact summaries
5. Agent can drill back into any summary with `lcm_grep` / `lcm_expand`

## Multi-Platform Setup

If you use WhatsApp + webchat (or any multiple surfaces):

```jsonc
{
  "session": {
    "resetByChannel": {
      "webchat": {
        "mode": "idle",
        "idleMinutes": 10080  // 7 days — prevents orphaned sessions
      }
    }
  }
}
```

The task file workflow bridges platforms:
1. Do deep work on webchat
2. Say "write task" → agent saves dense snapshot to `memory/tasks/`
3. Switch to WhatsApp → "continue the project" → agent reads task file, picks up seamlessly

See [`references/cross-platform.md`](references/cross-platform.md) for details.

## Dream Consolidation

As of OpenClaw 2026.4.5, the platform ships with **built-in dreaming** — no cron jobs needed for core memory maintenance.

Native dreaming runs three phases (light / deep / REM) automatically:
- Promotes high-value short-term memories to long-term storage
- Applies recall decay (`recencyHalfLifeDays`, `maxAgeDays`)
- Chunks nearby daily-note lines for better evidence quality
- Writes narrative output to `dreams.md` + machine state to `memory/.dreams/`
- Replay-safe: reruns reconcile instead of duplicating
- `/dreaming` command for on-demand runs + Dreams UI panel

**Enable it:**

```jsonc
{
  "plugins": {
    "entries": {
      "memory-core": {
        "enabled": true,
        "config": {
          "dreaming": {
            "enabled": true,
            "frequency": "0 4 * * *"  // cron expr, default "0 3 * * *"
          }
        }
      }
    }
  }
}
```

That's it — two fields. Phases, thresholds, and scheduling are handled internally.

For MEMORY.md compression and task file archival beyond what native dreaming provides, see [`references/dream-consolidation.md`](references/dream-consolidation.md) for a custom cron-based approach.

## Tuning

See [`references/lcm-tuning.md`](references/lcm-tuning.md) for LCM parameter tuning, including three ready-made profiles:

- **Conservative** — large context window, infrequent compaction
- **Aggressive** — smaller window or high-volume conversations  
- **Cost-Optimized** — use free/cheap models for everything

## File Reference

```
memory-orchestrator/
├── SKILL.md                        # main skill (agent reads this)
├── references/
│   ├── architecture.md             # three-layer design
│   ├── lcm-tuning.md              # LCM parameters & profiles
│   ├── cron-templates.md          # ready-to-use cron + dream jobs
│   ├── cross-platform.md          # multi-platform continuity
│   └── dream-consolidation.md     # dream cycle deep-dive
└── scripts/                        # (reserved for future automation)
```

## Requirements

- [OpenClaw](https://github.com/openclaw/openclaw) (2026.4.5+)
- [Lossless Claw](https://github.com/Martian-Engineering/lossless-claw) plugin (`@martian-engineering/lossless-claw`)
- Node.js 20+

## License

MIT
