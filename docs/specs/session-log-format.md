# Session-log container format (concatenated Zstandard frames)

> Canonical write-up of discussion #2328. This documents the on-disk session
> artifact so future session-tool authors stop reverse-engineering it
> independently.

## The artifact is not one Zstandard stream

A DeepSeek Harness session log (`session.jsonl.zstd`) is a **concatenation of
independently decodable Zstandard frames**, not a single zstd stream. Each frame
holds a prefix of the JSONL event stream (frames are flushed as events are
written).

This is load-bearing and currently undocumented: a one-shot decompressor
(`zstdDecompressSync` from `node:zlib`, or the `zstd` CLI's naive single-frame
path) silently returns **only the first frame** and drops every later event.

## Two consumption tiers

### Tier 1 — structural scan (audit / read tools)

Walk the byte stream and cut frame boundaries structurally:

- validate the frame magic `0xFD2FB528`
- parse the frame-header descriptor (single-segment + frame-content-size)
- parse each block header to find the block payload and the last-block flag
- stop at the first incomplete frame

Reference: `scanZstdFrames` in the upstream `session-persistence-jsonl` format
module, and mirrored by
[dsh-replay](https://github.com/zoahdev/dsh-replay/blob/master/engine/replay.js).

### Tier 2 — crash-tail recovery (repair / rescue tools)

After a crash or power loss, the last frame is often **torn** (written only
partway). A repair tool should tolerate it: decompress every structurally
complete frame, then either drop or best-effort prefix-decode the torn tail.

Reference: `dsh-shelf`
([engine/shelf.js](https://github.com/zoahdev/dsh-shelf/blob/main/engine/shelf.js))
skips a corrupt final frame and still exports the complete frames before it.

## Rules

1. The artifact is a concatenation of independently decodable zstd frames.
2. Consumers must either scan frame boundaries structurally (`scanZstdFrames`)
   or stream-decompress; one-shot decompressors stop at the first frame by
   design.
3. Torn final frames occur after crashes and may be prefix-decoded (or skipped
   by a repair tool) instead of treated as fatal corruption.

## Why this is the standard onboarding wall

At least three community tools independently reverse-engineered this container
before it was written down: `dsh-doctor`'s `log_health`, `dsh-report-studio`'s
`report_week`, and `dsh-replay`. This page exists so the next author reads it
instead of spending the same multi-hour debugging session.

Mirror this into the official `session-persistence` docs once the PR channel
opens.
