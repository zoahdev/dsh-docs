# Reproduction note: compaction request misses the provider prefix cache

> Discussion #1944 路 Reporter-verified session data 路 Maintainer: zoahdev

## Setup

- Gateway: `opencode-go` (cache key includes `reasoning_effort` / `max_tokens`)
- Model: `deepseek-v4-flash`
- Reasoning effort: `high`
- Trigger: auto `compactIfNeeded` or manual `/compact` on a long session

## Before the fix

```json
{"inputTokens":122558,"outputTokens":1584}
```

No `cacheReadTokens` - 100% cache miss on the compaction request; ~122k tokens
re-priced on every compaction.

## After the fix (inherit header config, no forced maxTokens)

```json
{"inputTokens":402,"outputTokens":3548,"cacheReadTokens":220928}
```

99.7% of the prefix is reused; only the compaction instruction is new input.

## How reviewers can verify without a long session

1. Start a session with `reasoningEffort: high` and push enough context to
   cross the compaction threshold (or run `/compact`).
2. Record the compaction event's `usage` before and after the fix.
3. The tell: before the fix `inputTokens` is near the full context and
   `cacheReadTokens` is absent/zero; after the fix `cacheReadTokens` is large
   and `inputTokens` is small (only the instruction).

## Patch

https://github.com/zoahdev/deepseek-harness/tree/fix/compaction-inherit-header-config
