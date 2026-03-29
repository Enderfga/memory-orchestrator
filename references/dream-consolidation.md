# Dream Consolidation — Deep Dive

## The Problem

Memory systems without periodic maintenance follow a predictable decay curve:

1. **Week 1-2**: Everything works. Mem0 recalls are relevant, MEMORY.md is concise.
2. **Week 3-4**: Noise accumulates. Auto-captured Mem0 entries include gateway status, timestamps, exec logs. Recall quality degrades as noise competes with signal.
3. **Month 2+**: Crisis. Hundreds of junk entries dominate recall. MEMORY.md has stale "completed" items, dead projects, relative dates that no longer make sense. The system actively hurts more than it helps.

In our production deployment, unchecked growth over ~5 weeks produced:
- 1,091 Mem0 entries (60%+ noise)
- 139-line MEMORY.md with 9 stale completed items
- 15 active task files (4 already done)
- 42 daily logs (14 from 2+ months ago, never referenced)

## The Solution: Dream Cycle

Inspired by [Claude Code's Auto Dream](https://docs.anthropic.com/), the Dream cycle is a periodic self-maintenance routine that keeps memory healthy.

### Six Steps

#### 1. Orientation
Read MEMORY.md. Count sections, lines, last update date. Identify what needs attention.

#### 2. Gather Signal
Read the past week's daily logs (`memory/YYYY-MM-DD.md`). Search Mem0 for recent high-value memories. Build a picture of what happened this week.

#### 3. Consolidation
The core step:
- Update "当前状态" / status section with current project states
- Remove completed items (they're already in weekly logs)
- Convert any relative dates ("yesterday", "last week") to absolute `YYYY-MM-DD`
- Delete contradicted or superseded entries
- Compress weekly reports older than 2 weeks to one-line summaries
- Merge overlapping entries
- Target: keep MEMORY.md under 120 lines

#### 4. Mem0 Pruning
Search for and delete memories matching noise patterns:
- `WhatsApp/WeChat gateway connected/disconnected` — status logs, not memory
- `current time is` / `notes the current time` — timestamp noise
- `Exec completed/failed` — session-level logs
- `message_id` / `conversation metadata` — routing noise
- `heartbeat` check records
- `scheduled task to clean uploads` — recurring cleanup noise

Use `memory_search` to find candidates, verify they're noise, then `memory_forget` to delete.

#### 5. Task File Review
Scan `memory/tasks/*.md`:
- Completed tasks (`[DONE]` or clearly finished) → move to `memory/tasks/archive/`
- Stale tasks (>2 weeks no update, no longer relevant) → flag for user review

#### 6. Index
Update the "上次整理" timestamp at the bottom of MEMORY.md. Verify line count is under target.

### Scheduling

| Job | Schedule | Model | Cost/Run |
|-----|----------|-------|----------|
| Memory Dream | Sunday 04:00 | Haiku 4.5 | ~$0.01 |
| Daily Memory Sync | Daily 23:00 | Gemini Flash | ~$0.005 |
| Weekly Compound | Sunday 22:00 | Default | ~$0.05 |

Dream runs *before* Weekly Compound so MEMORY.md is clean when the compound adds new content.

### Prevention vs Cleanup

Dream handles cleanup, but prevention is equally important. Add write-filtering rules to AGENTS.md:

```markdown
### 🚫 Mem0 Write Filtering

DO NOT write to Mem0:
- Gateway connect/disconnect status
- Current timestamps
- Exec completed/failed status
- Message IDs and conversation metadata
- Heartbeat check records

DO write to Mem0:
- User preferences and behavior patterns
- Architecture decisions and technical choices
- Project status changes
- Lessons learned and post-mortems
- Contact info and account details
```

This is the "brush your teeth" approach — daily prevention reduces how much the weekly "dentist visit" (Dream) needs to do.

### Absolute Date Rule

A subtle but critical rule: **never write relative dates to persistent memory.**

"Yesterday we decided X" becomes meaningless in 48 hours. Always write "On 2026-03-28 we decided X."

This applies to:
- MEMORY.md
- Task files (`memory/tasks/*.md`)
- Mem0 entries (when manually storing)
- Daily logs (these are dated by filename, but content should still use absolute dates)

## Customization

### Adjusting Frequency
- High-volume setups (>50 messages/day): Consider bi-weekly Dream (Wednesday + Sunday)
- Low-volume setups: Monthly may suffice — change cron to `0 4 1 * *`

### Adjusting Aggressiveness
- **Conservative**: Only delete exact-match noise patterns, never compress MEMORY.md below 150 lines
- **Aggressive**: Add more noise patterns, compress MEMORY.md to 80 lines, archive task files after 1 week of completion
- **Default**: As described above — balanced for most setups

### Custom Noise Patterns
If your setup generates specific noise (e.g., home automation status, CI/CD notifications), add those patterns to the Dream cron prompt and to your AGENTS.md filter rules.
