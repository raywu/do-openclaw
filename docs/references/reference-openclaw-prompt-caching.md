# OpenClaw Prompt Caching Configuration

> **Purpose:** Guide for configuring Anthropic prompt caching in OpenClaw to
> reduce token costs on system prompt re-sends.

## Prerequisites

Upgrade OpenClaw to **v2026.2.23+**:

```bash
openclaw update
openclaw --version   # confirm >= 2026.2.23
```

v2026.2.23 adds: per-agent `cacheRetention` overrides, bootstrap caching per
session (reduces invalidations from mid-session memory writes), and cache trace
diagnostics.

---

## How It Works

OpenClaw auto-applies `cacheRetention: "short"` (5-min TTL) for Anthropic
API-key auth. For agents with gaps >5 minutes between interactions, the cache
expires and the full system prompt is re-sent every request. Extending the TTL
keeps the system prompt cached across typical interaction gaps.

**Note:** Prompt caching is an Anthropic-only feature. `cacheRetention` is
silently ignored for non-Anthropic providers (OpenAI, Google, etc.).

---

## openclaw.json Configuration

### Model-Level Cache Retention

**JSON path:** `agents.defaults.models`

```json
{
  "agents": {
    "defaults": {
      "models": {
        "anthropic/claude-sonnet-4-6": {
          "params": {
            "cacheRetention": "long"
          }
        }
      }
    }
  }
}
```

This extends the cache TTL so the system prompt stays cached across typical
10-60 minute gaps between interactions.

### Per-Agent Override (Optional)

Cron jobs that complete within a single turn benefit from the default `"short"`
(5-min) TTL. No override needed. To explicitly disable caching for specific jobs:

```bash
openclaw cron edit <uuid> --params '{"cacheRetention": "none"}'
```

### Cache Retention Options

| Value | TTL | Use Case |
|-------|-----|----------|
| `"short"` | 5 minutes | Default for Anthropic. Fine for rapid-fire cron jobs |
| `"long"` | Extended | Main sessions with 10-60 min gaps between interactions |
| `"none"` | Disabled | Debugging, or jobs where caching adds no value |

---

## Verification

After applying config changes:

```bash
# 1. Restart gateway to pick up config changes
openclaw gateway restart

# 2. Enable cache tracing (v2026.2.23+) — temporary, for verification only
OPENCLAW_CACHE_TRACE=1 openclaw gateway start

# 3. Trigger a few interactions, then check trace output for:
#    - cacheCreationInputTokens (cache write — first request pays this)
#    - cacheReadInputTokens (cache hit — subsequent requests save here)
#    Look for cacheReadInputTokens > 0 on the second request in a session.

# 4. Once verified, remove OPENCLAW_CACHE_TRACE=1 from the environment
```

### Expected Savings

| Token type | Before (default `"short"`) | After (`"long"`) |
|---|---|---|
| System prompt | ~15K tokens re-sent every request | Cached after first request per session |
| Cache write cost | 1.25x input price (first request) | Same |
| Cache read cost | N/A (5-min TTL expired) | 0.1x input price (subsequent requests) |
| Net effect | ~0% savings (TTL too short for typical gaps) | ~80-90% savings on system prompt tokens |

---

## Known Limitation

OpenClaw packs the entire system prompt into a **single cache block**. If any
component changes (including auto-injected timestamps), the whole block
invalidates and must be re-cached. This is tracked upstream as
[openclaw/openclaw#6540](https://github.com/openclaw/openclaw/issues/6540).
The v2026.2.23 "bootstrap caching per session" partially mitigates this for
mid-session memory writes.

**Practical implication:** Any workspace file edit (SOUL.md, MEMORY.md, skill
changes) invalidates the cache for all sessions. Plan workspace edits during
low-traffic windows to minimize re-caching cost.

---

## Provider-Specific Behavior

| Provider | `cacheRetention` | Auto-Applied Headers | Notes |
|---|---|---|---|
| Anthropic | Honored (`short`/`long`/`none`) | `cache_control: { type: "ephemeral" }`, `extended-cache-ttl-2025-04-11` beta flag | Full support |
| OpenAI | Silently ignored | None | Parameter accepted without error but no caching behavior applied |
| Google | N/A | Provider manages caching internally | Not configurable via OpenClaw |

When using mixed model providers across cron tiers (e.g., Anthropic for
customer-facing, OpenAI for automation), only Anthropic-model jobs benefit from
`cacheRetention`. This is expected — do not add `cacheRetention` to
non-Anthropic model configs (it wastes config lines but has no effect).
