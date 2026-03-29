# 🧠 Memory Orchestrator

A five-layer memory system for [OpenClaw](https://github.com/openclaw/openclaw) agents — persistent, cross-session, cross-platform context that never forgets, with automatic **Dream consolidation** to prevent memory bloat.

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

## Dream Consolidation (v6)

Without periodic maintenance, memory systems degrade fast:
- **Mem0** fills with noise (gateway status logs, timestamps, exec results) → recall quality tanks
- **MEMORY.md** accumulates stale entries → bloated context every session
- **Task files** pile up after completion → dead weight

Dream consolidation runs weekly (via cron) and performs six automated steps:

```
┌─────────────────────────────────────────────────┐
│              Weekly Dream Cycle                  │
│                                                  │
│  1. Orientation    → Read MEMORY.md state        │
│  2. Gather Signal  → Scan week's daily logs      │
│  3. Consolidation  → Update, prune, compress     │
│  4. Mem0 Pruning   → Delete noise memories       │
│  5. Task Review    → Archive completed tasks     │
│  6. Index          → Keep MEMORY.md < 120 lines  │
└─────────────────────────────────────────────────┘
```

### Setup

```bash
# Add the Dream cron job (runs Sunday 04:00)
openclaw cron add \
  --name "Memory Dream (Weekly Consolidation)" \
  --cron "0 4 * * 0" \
  --tz "Asia/Singapore" \
  --session isolated \
  --model "anthropic/claude-haiku-4-5" \
  --timeout-seconds 300 \
  --no-deliver \
  --message 'Execute weekly memory dream consolidation. Steps:
1. ORIENTATION: Read MEMORY.md, count lines
2. GATHER SIGNAL: Read last 7 days of daily logs. Search Mem0 for recent memories
3. CONSOLIDATION: Update status, remove completed items, convert relative dates to YYYY-MM-DD, compress old reports, keep under 120 lines
4. MEM0 PRUNING: Search and delete noise (gateway status, timestamps, exec status) via memory_forget
5. TASK FILES: Move completed tasks to archive/
6. Update timestamp. Output brief summary.'
```

### Mem0 Noise Prevention

Add write-filtering rules to your `AGENTS.md` to stop noise at the source:

**Never write to Mem0:**
- Gateway connect/disconnect status
- Current timestamps
- Exec completed/failed status
- Message IDs and conversation metadata

**Always write to Mem0:**
- User preferences and behavior patterns
- Architecture decisions
- Project status changes
- Lessons learned

### Real-World Impact

From our production deployment (1 month of unchecked growth → first Dream run):

| Metric | Before | After |
|--------|--------|-------|
| Mem0 entries | 1,091 | 114 (-89%) |
| MEMORY.md lines | 139 | 83 (-40%) |
| Active task files | 15 | 10 |
| Noise hit rate | ~60% | <10% |

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
│   ├── architecture.md             # full five-layer design
│   ├── lcm-tuning.md              # LCM parameters & profiles
│   ├── cron-templates.md          # ready-to-use cron + dream jobs
│   ├── cross-platform.md          # multi-platform continuity
│   └── dream-consolidation.md     # dream cycle deep-dive
└── scripts/                        # (reserved for future automation)
```

## Requirements

- [OpenClaw](https://github.com/openclaw/openclaw) (2026.3.12+)
- [Lossless Claw](https://github.com/Martian-Engineering/lossless-claw) plugin (`@martian-engineering/lossless-claw`)
- [Mem0](https://mem0.ai) plugin (`@mem0/openclaw-mem0`) + API key
- Node.js 20+

## License

MIT
