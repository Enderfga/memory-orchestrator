# Memory Architecture — Full Design

## Five-Layer Density Gradient

| Layer | Storage | Density | Load Timing | Purpose |
|-------|---------|---------|-------------|---------|
| LCM | SQLite DAG | Lossless | Auto (every turn) | Session-internal context, never lost |
| Task files | `memory/tasks/*.md` | Dense | On-demand (when mentioned) | Active work snapshots |
| Daily logs | `memory/YYYY-MM-DD.md` | Medium | Session start (today+yesterday) | Daily event journal |
| Mem0 | Cloud | Fragments | Auto (top-K per turn) | Semantic cross-session recall |
| MEMORY.md | Local file | Sparse | Main session only | High-level overview, lessons |

## How They Cooperate

```
Session starts
  ├── LCM: loads summary DAG + recent raw messages (automatic)
  ├── Daily logs: reads today + yesterday
  ├── Mem0: injects top-5 relevant memories per message
  └── MEMORY.md: loaded if main/private session
  
During conversation
  ├── LCM: persists every message, compacts when threshold hit
  ├── Mem0: auto-captures facts from each turn
  └── Agent: updates task files and daily logs as needed

End of day (23:00 cron)
  ├── Daily Sync: scans all sessions → writes daily log
  ├── Backfill: creates task files for uncovered work
  └── QMD: re-indexes for full-text search

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

## Plugin Slot Separation

```
contextEngine slot → lossless-claw (manages what goes into the context window)
memory slot       → openclaw-mem0  (manages cross-session semantic recall)
built-in          → qmd            (full-text BM25 search over workspace files)
```

These three are orthogonal. Each can be replaced independently.

## Session Reset Strategy

| Channel | Reset Mode | Idle Timeout | Rationale |
|---------|-----------|-------------|-----------|
| WhatsApp | idle | 30 min | Short conversations, fresh context each time |
| Webchat | idle | 7 days (10080 min) | Long-running deep work sessions |
| Cron | n/a | n/a | Isolated, no persistence needed |

Webchat sessions use localStorage for persistent sessionId. Without the extended timeout, idle sessions get reset → orphaned jsonl files → lost context.
