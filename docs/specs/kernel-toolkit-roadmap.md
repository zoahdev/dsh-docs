# dsh kernel toolkit — the "100 tools" roadmap

> zoahdev · maintained · DeepSeek Harness's ecosystem-infrastructure layer.
> A "kernel-layer tool" here means a tool that sits on a dsh **internal** — the
> session format, the Cordis composition, the sandbox policy, the preset/bundle
> wiring, or the plugin lifecycle — not a user-facing plugin. Each tool is
> zero-dependency or near-zero-dependency, test-covered, and CI-runnable.

## Status legend

- ✅ shipped · 🔨 in progress · ⬜ planned

## 1. Session layer (`session.jsonl.zstd`)

- ✅ dsh-shelf — lifecycle (list/stats/export/archive/trash/search/rescue)
- ✅ dsh-replay — trajectory decode + HTML timeline + stats + diff
- 🔨 dsh-session-stats — tokens/cost/latency analytics from event logs
- ⬜ dsh-session-diff — semantic turn-level diff (beyond tool-name lists)
- ⬜ dsh-log-rotator — archive/compress stale sessions by age/size
- ⬜ dsh-migrate — move sessions between jsonl/sqlite backends
- ⬜ dsh-session-anonymize — scrub secrets/PII from a log before sharing

## 2. Composition / preset layer (`cordis.patch.yml`, `agent.cordis.yml`)

- ✅ dsh-compose-viz — render groups/isolate realms/rows as HTML
- ✅ dsh-sandbox-audit — policy-consistency audit (HIGH/MEDIUM findings)
- ⬜ dsh-preset-diff — diff two presets or a preset vs a fork
- ⬜ dsh-patch-lint — validate `insert`/`id`/`isolate` well-formedness
- ⬜ dsh-config-audit — lint `--dump-config` output for known failure shapes
- ⬜ dsh-compose-graph — Mermaid/Graphviz dependency graph of realms/services

## 3. Sandbox layer

- ✅ dsh-sandbox-audit (shared with #2)
- ⬜ dsh-sandbox-repro — reproduce a `FS_SANDBOX_DENIED` in isolation
- ⬜ dsh-destructive-guard — shell-layer `destructive-ops: ask` gate (#149)
- ⬜ dsh-sandbox-matrix — test one action across bwrap/landlock/seatbelt/acl

## 4. Plugin / market layer

- ✅ dsh-subscribe — 569-plugin registry + one-command install
- ✅ dsh-plugin-doctor — preflight (manifest/patch/build/pack/install/boot)
- ✅ dsh-plugin-template — verified template + runtime peer guard
- ⬜ dsh-plugin-graph — resolve a plugin's transitive deps + bundle layers
- ⬜ dsh-plugin-lock — pin verified plugin versions for reproducible profiles
- ⬜ dsh-plugin-bench — cold-boot + first-call latency per plugin

## 5. Observability layer

- ⬜ dsh-trace — full per-session trace (reasoning/tools/tokens/cost) + UI
- ⬜ dsh-profiler — agent-loop and tool-scheduler hot-path profiling
- ⬜ dsh-error-digest — cluster session failures into recurring signatures

## 6. Shell / subprocess layer

- ⬜ dsh-shell-profiler — bash/pwsh startup + per-command latency
- ⬜ dsh-terminal-replay — replay a persistent shell's PTY stream
- ⬜ dsh-spawn-inspect — dump argv/env of a sandboxed spawn for debugging

## 7. Release / ecosystem layer

- ✅ dsh-ecosystem — map, weekly report, release-compat watch
- ✅ dsh-docs — PR-ready docs + 28-patch queue
- ⬜ dsh-rc-report — automated per-rc compatibility report (machine ready)
- ⬜ dsh-registry-health — dead-link / drift / version-lag scan of the registry

## Count

- shipped: **11** · in progress: **2** · planned: **26**
- Running total once the full roadmap ships: **39 kernel-layer tools**.

The 100-tool ceiling is the same idea applied one level deeper (per-event-type
decoders, per-backend matrixes, per-tool-catalog analyzers). Each increment is
small and independently useful, which is what makes the toolkit defensible.
