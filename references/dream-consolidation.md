# Dream Consolidation — Deep Dive

## OpenClaw Native Dreaming (2026.4.5+)

As of OpenClaw 2026.4.5, the platform ships with a built-in dreaming system under `memory-core`. It runs three cooperative phases (light, deep, REM) and handles:

- **Recall promotion**: Weighted short-term memory → long-term promotion
- **Aging/decay**: Configurable `recencyHalfLifeDays` and `maxAgeDays`
- **Daily-note chunking**: Groups nearby lines for better evidence quality
- **Output**: `dreams.md` (human-readable narrative) + `memory/.dreams/` (machine state)
- **Replay-safe**: Reruns reconcile instead of duplicating `MEMORY.md` entries
- **UI**: `/dreaming` command + Dream Diary panel

**Config** (only two fields):
```json
{
  "plugins": {
    "entries": {
      "memory-core": {
        "enabled": true,
        "config": {
          "dreaming": {
            "enabled": true,
            "frequency": "0 4 * * *"
          }
        }
      }
    }
  }
}
```

Native dreaming and the custom Dream cron complement each other — native handles recall promotion/decay, the custom cron handles MEMORY.md compression and task archival.

---

## Custom Dream Cron

## The Problem

Memory systems without periodic maintenance follow a predictable decay curve:

1. **Week 1-2**: Everything works. MEMORY.md is concise, task files are current.
2. **Week 3-4**: Stale entries accumulate. MEMORY.md has completed items, dead projects, relative dates that no longer make sense.
3. **Month 2+**: Crisis. MEMORY.md bloated context on every session start. Task files pile up with completed work.

## The Solution: Custom Dream Cycle

The custom Dream cycle is a periodic self-maintenance routine that keeps memory healthy. It covers what native dreaming doesn't touch: MEMORY.md compression and task archival.

### Four Steps

#### 1. Orientation
Read MEMORY.md. Count sections, lines, last update date. Identify what needs attention.

#### 2. Consolidation
The core step:
- Update "当前状态" / status section with current project states
- Remove completed items (they're already in weekly logs)
- Convert any relative dates ("yesterday", "last week") to absolute `YYYY-MM-DD`
- Delete contradicted or superseded entries
- Compress weekly reports older than 2 weeks to one-line summaries
- Merge overlapping entries
- Target: keep MEMORY.md under 120 lines

#### 3. Task File Review
Scan `memory/tasks/*.md`:
- Completed tasks (`[DONE]` or clearly finished) → move to `memory/tasks/archive/`
- Stale tasks (>2 weeks no update, no longer relevant) → flag for user review

#### 4. Index
Update the "上次整理" timestamp at the bottom of MEMORY.md. Verify line count is under target.

### Scheduling

| Job | Schedule | Model | Cost/Run |
|-----|----------|-------|----------|
| Memory Dream | Sunday 04:00 | Haiku 4.5 | ~$0.01 |
| Daily Memory Sync | Daily 23:00 | Gemini Flash | ~$0.005 |
| Weekly Compound | Sunday 22:00 | Default | ~$0.05 |

Dream runs *before* Weekly Compound so MEMORY.md is clean when the compound adds new content.

### Absolute Date Rule

A subtle but critical rule: **never write relative dates to persistent memory.**

"Yesterday we decided X" becomes meaningless in 48 hours. Always write "On 2026-03-28 we decided X."

This applies to:
- MEMORY.md
- Task files (`memory/tasks/*.md`)
- Daily logs (these are dated by filename, but content should still use absolute dates)

## Customization

### Adjusting Frequency
- High-volume setups (>50 messages/day): Consider bi-weekly Dream (Wednesday + Sunday)
- Low-volume setups: Monthly may suffice — change cron to `0 4 1 * *`

### Adjusting Aggressiveness
- **Conservative**: Never compress MEMORY.md below 150 lines
- **Aggressive**: Compress MEMORY.md to 80 lines, archive task files after 1 week of completion
- **Default**: As described above — balanced for most setups
