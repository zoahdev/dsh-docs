# `dsh env explain` - secret-safe environment provenance contract

> Status: community reference (discussion #1953) 路 Maintainer: zoahdev 路 Version: 1

## Purpose

Explain which layer won for one environment key without exposing the value,
its length, a prefix, a suffix, or any hash. Only key name, source layer,
resolution state, and non-sensitive reasons are returned.

## Command surface

```console
dsh env explain <KEY> [--profile <dir>] [--json]
```

`--profile <dir>` resolves the `project .env` layer from that directory (the
profile root). The `user .env` layer resolves from the OS home directory.

## Layer model and precedence

| Precedence | Layer | Source |
| --- | --- | --- |
| 1 (highest) | `launch environment` | the process environment |
| 2 | `project .env` | `<profile-or-cwd>/.env` |
| 3 | `user .env` | `<home>/.env` |

Empty-value rule (discussion #981): an explicitly empty value is reported as
`empty` and resolution CONTINUES to lower layers; the skipped layer is
preserved in the diagnostics. An empty value never masks a valid lower-layer
credential. An empty `launch environment` value is authoritative (it cannot
be overridden from below) and is reported as `empty` with no winner.

## States

`selected` 路 `absent` 路 `empty` 路 `lower-precedence` 路
`non-regular-path` 路 `unreadable` 路 `rejected-protected` (reserved)

- `rejected-protected` is reserved for harness-internal protected-key rules
  (bootstrap-only filtering, deny lists). A standalone tool SHOULD NOT
  fabricate it without the official contract; when emitted it must carry a
  non-sensitive reason.

## JSON envelope

```json
{
  "key": "DEEPSEEK_API_KEY",
  "resolved": true,
  "source": "launch environment",
  "layers": [
    { "layer": "launch environment", "state": "selected", "reason": "selected (highest precedence)" },
    { "layer": "project .env", "state": "lower-precedence", "reason": "present but shadowed" },
    { "layer": "user .env", "state": "lower-precedence", "reason": "present but lower precedence" }
  ],
  "value": "[redacted]"
}
```

`value` is the constant string `[redacted]` - never derived from the real
value, never a hash, never a length hint.

## Security requirements

- The raw value never enters output, error messages, or logs.
- No prefix/suffix extraction, no hashing, no length disclosure.
- The redaction marker is constant across all outputs.
- The command is read-only: it never modifies `.env` files or the process
  environment.

## Reference implementation

- Repo: [zoahdev/dsh-plugin-doctor](https://github.com/zoahdev/dsh-plugin-doctor) v1.13.0+
- Command: `npx dsh-plugin-doctor env explain <KEY> [--json]`
- Tests: 40/40 (env-explain suite asserts the real value string never appears
  in the serialized report)

## Open items

- Emit `rejected-protected` once the harness's protected-key rules are public.
- Surface the same contract inside `dsh doctor`'s env section via a shared
  internal `env.explain(key)` API when the official CLI exists.
