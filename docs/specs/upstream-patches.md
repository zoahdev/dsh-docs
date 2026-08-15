# Upstream patch queue — cherry-pick-ready branches

> Maintained by zoahdev · Base: `deepseek-ai/deepseek-harness@master` (47f9438, 2026-08-13) · Updated 2026-08-15
>
> When upstream reopens the PR channel, each branch below can be cherry-picked
> as-is. All branches are based on the same master commit and verified against
> the discussions they cite.

## Official test-suite verification (2026-08-15, Windows / Node 24 / pnpm 11)

Full official monorepo checkout at 47f9438, `pnpm install --frozen-lockfile`,
then the five patches applied with `git apply`:

| Stage | Test files | Tests | Result |
| --- | --- | --- | --- |
| Baseline (no patches) | 45 | 1140 | ✅ all passed |
| After all 5 patches | 46 | 1146 | ✅ all passed (exit 0) |

Packages covered: `@deepseek-ai/dsh-tools`, `@deepseek-ai/dsh-app-boot`,
`@deepseek-ai/dsh-terminal-bash`, `@deepseek-ai/dsh-llm-deepseek`,
`@deepseek-ai/dsh-client-ui-primitives`. The +6 tests are the
`duplicate-instance` regression suite carried by patch #1697.

Notes:
- All five patches apply cleanly to current master (`git apply --check` OK).
- Patch #1861 required updating 3 official adapter assertions (reasoning
  efforts list now includes `low`); that test update is part of the branch.
- Full log: `pnpm exec vitest run <5 test dirs>` → 1146 passed, exit 0.

## 1. #1697 — tool scheduler dual-instance crash (`undefined.prepare`)

- Branch: `fix/tool-runtime-scheduler-symbol-for`
- Files: `packages/core/tools/src/index.ts`, agent-loop protocol guard + regression tests
- Fix: `Symbol.for` shared key + `TOOL_RUNTIME_SCHEDULER_PROTOCOL_VERSION` guard (loud, actionable error on version skew instead of silent mismatch)
- Evidence: mechanism-level verification with two physical `dsh-tools@0.1.0-rc.6` copies (false → true); three-state matrix (same-version duplicate / version-skew / single instance)
- Discussion: https://github.com/deepseek-ai/deepseek-harness/discussions/1697

## 2. #1842 — UTF-8 BOM crashes `dsh web` at boot

- Branch: `fix/profile-manifest-bom-strip`
- File: `packages/boot/app-boot/src/profile.ts` (`readProfileManifest`)
- Fix: strip a leading `\uFEFF` before `JSON.parse` (one line)
- Evidence: local repro (BOM → `Unexpected token`, strip → works); companion `manifest-bom` check in dsh-plugin-doctor
- Discussion: https://github.com/deepseek-ai/deepseek-harness/discussions/1842

## 3. #1856 — Windows minimal preset bash cannot resolve `/bin/bash`

- Branch: `fix/terminal-bash-win32-shell`
- File: `packages/terminal/terminal-bash/src/index.ts`
- Fix: win32 + default `shellPath` → probe PATH / Git for Windows / LOCALAPPDATA for `bash.exe`; actionable error when missing
- Evidence: node-pty repro (`File not found`); companion `win-bash` check in dsh-plugin-doctor
- Discussion: https://github.com/deepseek-ai/deepseek-harness/discussions/1856

## 4. #1861 — `deepseek-official` adapter rejects `reasoning_effort: low`

- Branch: `fix/llm-deepseek-reasoning-low`
- File: `packages/llm/llm-deepseek/src/adapter.ts`
- Fix: add `low` to `REASONING_EFFORTS` (one const + one entry)
- Evidence: source whitelist confirmed (`off/high/max` only)
- Discussion: https://github.com/deepseek-ai/deepseek-harness/discussions/1861

## 5. #1869 — single tilde (`~x~`) wrongly renders as strikethrough

- Branch: `fix/markdown-single-tilde`
- File: `packages/client/ui-primitives/src/markdown/parse.ts` (both call sites)
- Fix: `gfm()` → `gfm({ singleTilde: false })` (micromark option; `marked` has no such option in v16 — verified)
- Evidence: micromark-extension-gfm@3 repro: `~x~` delete → literal with `singleTilde:false`; `~~y~~` stays delete
- Discussion: https://github.com/deepseek-ai/deepseek-harness/discussions/1869

## 6. #1891 — external session-directory deletion kills the process (ENOENT fatal)

- Branch: `fix/session-persistence-recreate-on-enoent`
- File: `packages/session/session-persistence-jsonl/src/index.ts` (`appendLines`)
- Fix: catch ENOENT at `open(path, 'a')`, `mkdir(dirname(path), { recursive: true })`, retry once; non-ENOENT errors unchanged
- Evidence: official suite `session-persistence-jsonl` — baseline 239/239 ✅, with patch 239/239 ✅ (no regression)
- Discussion: https://github.com/deepseek-ai/deepseek-harness/discussions/1891

## 7. #1919 - crypto.randomUUID crashes the Web UI on plain-HTTP LAN origins

- Branch: `fix/web-crypto-randomuuid-insecure-context`
- Files: new zero-dependency `packages/util/random-uuid` (+ 4 unit tests); browser-facing mints routed through it in `dsh-commands` (instance token), `dsh-client-ui-conversation` (draft attachment id), `dsh-host-apiproxy` (`mintRpcId`), `dsh-llm` (`MessageId`); `dsh-client-connection` re-exports the shared helper; `packages/client/tsdown.client.ts` registers it as `INLINE_SAFE` (pure browser-safe contract, no runtime identity)
- Fix: prefer `crypto.randomUUID()`, fall back to a `crypto.getRandomValues()`-backed RFC 4122 v4 generator on insecure origins (LAN HTTP / Tailscale IP)
- Evidence: `pnpm install --frozen-lockfile` passes (lockfile diff = importer entries only); `build:lib:host` and `build:lib:client` both pass; targeted vitest 782/782 (llm + client-connection + commands + random-uuid); built bundles contain no direct `crypto.randomUUID()` mints
- Full official suite on this Windows environment: 12731 passed / 80 failed (23 files) - every failing file is environment-specific (symlink/ACL sandbox, PTY/pwsh, worker-thread timeout, network-gated real-product tests) and outside the patch's package set; the six touched package suites are green
- Discussion: https://github.com/deepseek-ai/deepseek-harness/discussions/1919 (comment #discussioncomment-18030320)

## 8. #1944 - compaction summarizer misses the provider prefix cache (re-bills ~122k tokens per compaction)

- Branch: `fix/compaction-inherit-header-config`
- Files: `packages/compaction/compaction-basic/src/summarizer.ts` (+ tests)
- Fix: inherit the routed header config wholesale (same semantics as agent-loop's request proposal, adapter-defaulted fields dropped) instead of cherry-picking provider/model and forcing `maxTokens: 8192`; the compaction call keeps the exact provider cache key of normal turns
- Evidence: `tsc -b tsconfig.host.json` clean; compaction-basic 124/124; reporter session data: 122,558 input tokens / 0 cache -> 402 new input tokens + 220,928 `cacheReadTokens`
- Discussion: https://github.com/deepseek-ai/deepseek-harness/discussions/1944 (comment #discussioncomment-18030489)

## 9. #1954 - circular skill junctions crash dsh at startup (ELOOP fatal load failure)

- Branch: `fix/skill-filesystem-eloop-contained`
- Files: `packages/skill/skill-filesystem/src/index.ts` (+ watcher regression test)
- Fix: ELOOP-class watcher errors degrade only the affected root (warn + unhealthy + close partial watcher + skip) instead of rejecting provider load; scheduled rewatch retries and recovers when the cycle is fixed; non-ELOOP startup errors keep their existing contract
- Evidence: `tsc -b tsconfig.host.json` clean; skill-filesystem-watcher 11/11 including the new ELOOP regression test; remaining skill spec failures on Windows are pre-existing EPERM symlink-permission cases
- Discussion: https://github.com/deepseek-ai/deepseek-harness/discussions/1954 (comment #discussioncomment-18030568)

## 10. #1961 - purged %TEMP% spill directory crashes the whole service (ENOENT)

- Branch: `fix/subprocess-spill-recreate-on-enoent`
- Files: `packages/subprocess/subprocess-local/src/spawn.ts` (+ regression test in `spawn.spec.ts`)
- Fix: `spillAll` catches ENOENT, recreates the private spill dir (`mkdirSync recursive, mode 0700`), retries `openSync` once; if recreation fails, degrades to the in-memory tail instead of crashing from the stream `data` callback
- Evidence: host typecheck clean; runtime reproduction with a missing spill dir (tail + full spill preserved, no throw); regression test runs in the Linux CI lane (spawn.spec is excluded on Windows runners)
- Discussion: https://github.com/deepseek-ai/deepseek-harness/discussions/1961 (comment #discussioncomment-18030665)

## Submit checklist (when the channel opens)

1. `git fetch upstream && git merge-base --is-ancestor 47f9438 upstream/master` — rebase if master moved.
2. Run the touched package's tests (each branch carries its regression tests).
3. Open one PR per branch, title prefixed with the issue/discussion number.
4. Attach the evidence snippet from the corresponding discussion.
5. Offer to maintain the patch across release trains (zoahdev commits to this).
