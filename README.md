# 🧠 Memory Orchestrator

A five-layer memory system for [OpenClaw](https://github.com/openclaw/openclaw) agents — persistent, cross-session, cross-platform context that never forgets.

## Problem

OpenClaw agents lose context when:
- A long conversation exceeds the context window → old messages truncated
- You switch between chat platforms (WhatsApp → webchat) → separate sessions, no shared memory
- A new session starts → agent has no idea what happened before

## Solution

Memory Orchestrator coordinates five complementary memory layers:

| Layer | Scope | How | What It Does |
|-------|-------|-----|-------------|
| **LCM** | Session-internal | Auto DAG summaries | Compresses old messages into summaries instead of discarding them |
| **Task Files** | Cross-session | Manual + auto backfill | Dense snapshots of active work (like save points) |
| **Daily Logs** | Cross-session | Auto (cron) | Daily journal of events |
| **Mem0** | Cross-session | Auto capture/recall | Semantic fragment search across all conversations |
| **MEMORY.md** | Long-term | Weekly curation | High-level overview, lessons learned, project status |

## Quick Start

### 1. Install plugins

```bash
# Lossless Context Management
openclaw plugins install @martian-engineering/lossless-claw

# Semantic memory (get API key from https://mem0.ai)
openclaw plugins install @mem0/openclaw-mem0
```

### 2. Configure

Add to your `openclaw.json` (via `openclaw configure` or manual edit):

```jsonc
{
  "plugins": {
    "slots": {
      "contextEngine": "lossless-claw",  // manages context window
      "memory": "openclaw-mem0"           // manages cross-session recall
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
      "openclaw-mem0": {
        "enabled": true,
        "config": {
          "apiKey": "YOUR_MEM0_API_KEY",
          "userId": "your-username",
          "autoRecall": true,
          "autoCapture": true,
          "topK": 5
        }
      }
    }
  },
  "memory": {
    "backend": "qmd"   // built-in full-text search, no install needed
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

### 4. Set up cron jobs (recommended)

See [`references/cron-templates.md`](references/cron-templates.md) for ready-to-use templates:

- **Daily Memory Sync** (23:00) — scans sessions, writes daily log, backfills task files
- **Weekly Compound** (Sunday 22:00) — distills weekly logs into MEMORY.md
- **Brain Index** (07:30) — refreshes full-text search index

## How It Works

```
┌─────────────────────────────────────────────────┐
│                  Each Turn                       │
│                                                  │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐      │
│  │   LCM    │  │  Mem0    │  │  Daily   │      │
│  │ summaries│  │ top-5    │  │  logs    │      │
│  │ + recent │  │ memories │  │ today+   │      │
│  │ messages │  │ injected │  │ yesterday│      │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘      │
│       └──────────────┼──────────────┘            │
│                      ▼                           │
│              Agent Context Window                │
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

## Tuning

See [`references/lcm-tuning.md`](references/lcm-tuning.md) for LCM parameter tuning, including three ready-made profiles:

- **Conservative** — large context window, infrequent compaction
- **Aggressive** — smaller window or high-volume conversations  
- **Cost-Optimized** — use free/cheap models for everything

## File Reference

```
memory-orchestrator/
├── SKILL.md                        # main skill (agent reads this)
└── references/
    ├── architecture.md             # full five-layer design
    ├── lcm-tuning.md              # LCM parameters & profiles
    ├── cron-templates.md          # ready-to-use cron jobs
    └── cross-platform.md          # multi-platform continuity
```

## Requirements

- [OpenClaw](https://github.com/openclaw/openclaw) (2026.3.12+)
- [Lossless Claw](https://github.com/Martian-Engineering/lossless-claw) plugin (`@martian-engineering/lossless-claw`)
- [Mem0](https://mem0.ai) plugin (`@mem0/openclaw-mem0`) + API key
- Node.js 20+

## License

MIT
