# RTK and Prompt Caching

**TL;DR**: RTK does not break Claude's prompt cache. It filters command output once at execution time, stores the result in conversation history, and the cache works normally on every subsequent API call. Smaller outputs mean cheaper cache writes and reads.

## What RTK Modifies

| Component | Modified by RTK? | When? | Cache impact |
|-----------|-----------------|-------|-------------|
| Tool definitions (Bash, Read, Write...) | No | Never | None |
| System prompt (`@RTK.md` block) | Yes | Once at `rtk init` | One-shot invalidation, then stable |
| PreToolUse hook (`settings.json`) | Yes | Once at `rtk init` | Client-side only, never sent to API |
| Tool results (command output) | Yes | Each execution | Stored once in history, prefix unchanged |

The hook in `settings.json` rewrites commands before they run — it never touches the API request. Claude sees the filtered output, not the rewrite logic.

## How the Cache Works Turn by Turn

Prompt caching uses prefix matching. Every API call sends the full conversation history. As long as the prefix matches a previous call, Anthropic charges the cache read rate (0.1x) instead of the input rate (1x).

```
Turn 1:  system + @RTK.md + [no history]
         → Cache WRITE on system block (1.25x, one time)

Turn 2:  system + @RTK.md + [turn1: git status → "M src/main.rs"]
         → Cache READ on system block (0.1x)
         → Cache WRITE on turn1 result (1.25x, one time)

Turn 3:  system + @RTK.md + [turn1] + [turn2: cargo test → "1 failed: runner"]
         → Cache READ on system + turn1 (0.1x)
         → Cache WRITE on turn2 result (1.25x, one time)
```

The key point: RTK filters `cargo test` output from 25,000 tokens to 2,500 tokens. That result goes into history at 2,500 tokens, gets written to cache once (1.25x on 2,500), and read from cache every subsequent turn (0.1x on 2,500). Without RTK, the same slot costs 12.5x more to write and 10x more to read.

## Cost Comparison (30-min Session)

Based on the session data from the README (118k vs 23.9k tokens):

| Scenario | Input tokens | Cache write (1.25x) | Cache reads (0.1x × 5 turns avg) | Total |
|----------|-------------|---------------------|----------------------------------|-------|
| Without RTK | 118,000 | +147,500 | 59,000 | ~325k token-equivalent |
| With RTK | 23,900 | +29,875 | 11,950 | ~66k token-equivalent |
| **Savings** | | | | **~80%** |

The ratio holds because RTK reduces the size of what gets cached, not just what gets generated.

Pricing reference: [Anthropic Prompt Caching docs](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching).

## Verify It Yourself

`rtk gain` tracks `cache_create` and `cache_read` token metrics alongside the standard input/output savings:

```bash
rtk gain          # Global stats including cache metrics
rtk gain -p       # Per-project breakdown
```

If you see cache hit rates increasing over a session, the cache is working. RTK does not interfere with that trend.
