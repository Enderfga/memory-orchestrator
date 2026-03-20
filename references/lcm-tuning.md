# LCM Tuning Guide

## Config Parameters

| Parameter | Default | Description | Tuning Advice |
|-----------|---------|-------------|---------------|
| `freshTailCount` | 32 | Recent messages kept raw (not summarized) | Higher = more context for recent conversation, more tokens used. 20-50 is typical. |
| `contextThreshold` | 0.75 | Fraction of context window that triggers compaction | Lower = more aggressive compaction. With 200k window, 0.75 = compaction at 150k. |
| `incrementalMaxDepth` | 0 | How deep compaction goes per cycle (0=leaf only, -1=unlimited) | -1 for aggressive compression; 0 for minimal per-turn overhead. |
| `leafMinFanout` | 8 | Min raw messages per leaf summary | Lower = more frequent, smaller summaries. |
| `condensedMinFanout` | 4 | Min leaf summaries per condensed node | Lower = deeper DAG, more compression. |
| `summaryModel` | (default model) | Model for compaction summaries | Use a cheap/fast model (Haiku, Flash). Summaries don't need Opus. |
| `expansionModel` | (default model) | Model for `lcm_expand_query` sub-agent | Can be the same cheap model. |
| `ignoreSessionPatterns` | `[]` | Glob patterns for sessions to exclude | Exclude cron: `["agent:*:cron:**"]` |

## Environment Variables (override config)

| Variable | Default | Notes |
|----------|---------|-------|
| `LCM_PRUNE_HEARTBEAT_OK` | false | Auto-delete HEARTBEAT_OK turns from storage |
| `LCM_AUTOCOMPACT_DISABLED` | false | Disable auto-compaction (for debugging) |
| `LCM_DATABASE_PATH` | `~/.openclaw/lcm.db` | SQLite database location |
| `LCM_FRESH_TAIL_COUNT` | 32 | Overrides config `freshTailCount` |
| `LCM_CONTEXT_THRESHOLD` | 0.75 | Overrides config `contextThreshold` |

## Recommended Profiles

### Conservative (large context window, infrequent compaction)
```json
{
  "freshTailCount": 48,
  "contextThreshold": 0.85,
  "incrementalMaxDepth": 0,
  "summaryModel": "anthropic/claude-haiku-4-5"
}
```

### Aggressive (smaller window or high-volume conversations)
```json
{
  "freshTailCount": 16,
  "contextThreshold": 0.6,
  "incrementalMaxDepth": -1,
  "summaryModel": "anthropic/claude-haiku-4-5"
}
```

### Cost-Optimized (use free/cheap models for everything)
```json
{
  "freshTailCount": 32,
  "contextThreshold": 0.75,
  "incrementalMaxDepth": -1,
  "summaryModel": "google/gemini-3-flash-preview"
}
```

## Monitoring

- **DB size**: `ls -lh ~/.openclaw/lcm.db` — grows with all messages stored
- **Compaction frequency**: Check logs for `[lcm] Compaction` entries
- **Summary quality**: Use `lcm_describe` on a summary to review content
