# Memory Architecture — Full Design

## Three-Layer Density Gradient

| Layer | Storage | Density | Load Timing | Purpose |
|-------|---------|---------|-------------|---------|
| LCM | SQLite DAG | Lossless | Auto (every turn) | Session-internal context, never lost |
| Task files | `memory/tasks/*.md` | Dense | On-demand (when mentioned) | Active work snapshots |
| Daily logs | `memory/YYYY-MM-DD.md` | Medium | Session start (today+yesterday) | Daily event journal |
| MEMORY.md | Local file | Sparse | Main session only | High-level overview, lessons |

## How They Cooperate

```
Session starts
  ├── LCM: loads summary DAG + recent raw messages (automatic)
  ├── Daily logs: reads today + yesterday
  └── MEMORY.md: loaded if main/private session
  
During conversation
  ├── LCM: persists every message, compacts when threshold hit
  └── Agent: updates task files and daily logs as needed

End of day (23:00 cron)
  ├── Daily Sync: scans all sessions → writes daily log
  └── Backfill: creates task files for uncovered work

Weekly (Sunday 22:00 cron)
  └── Compound: distills weekly logs → updates MEMORY.md
```

## LCM DAG Structure

```
Raw messages (stored in SQLite)
  ↓ chunk (leafMinFanout=8 msgs per chunk)
Leaf summaries (~1200 tokens each)
  ↓ condense (condensedMinFanout=4 summaries)
Higher-level summaries (~2000 tokens each)
  ↓ condense (recursive as needed)
Root summary
```

Every turn's context = root/mid-level summaries + freshTailCount raw messages.

Agent can drill back: `lcm_grep` → find summary → `lcm_expand` → see original messages.

## Plugin Slot

```
contextEngine slot → lossless-claw (manages what goes into the context window)
```

## Session Reset Strategy

| Channel | Reset Mode | Idle Timeout | Rationale |
|---------|-----------|-------------|-----------|
| WhatsApp | idle | 30 min | Short conversations, fresh context each time |
| Webchat | idle | 7 days (10080 min) | Long-running deep work sessions |
| Cron | n/a | n/a | Isolated, no persistence needed |

Webchat sessions use localStorage for persistent sessionId. Without the extended timeout, idle sessions get reset → orphaned jsonl files → lost context.
