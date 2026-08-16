# Upstream patch queue — cherry-pick-ready branches

> Maintained by zoahdev · Base: `deepseek-ai/deepseek-harness@master` (47f9438, 2026-08-13) · Updated 2026-08-16
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

## 11. #1993 - source launch duplicates dsh-typert-protocol, silently disabling out-of-tree plugin Remote layers

- Branch: `fix/typert-remote-markers-shared-registry`
- Files: `packages/typert/protocol/src/index.ts` (+ cross-copy regression test)
- Fix: Remote decorator markers move from a module-private WeakMap to a `Symbol.for`-keyed registry on globalThis, shared by every physical package copy (src under tsx vs lib for out-of-tree plugins); WeakMap semantics and conflict validation preserved
- Evidence: `tsc -b tsconfig.host.json` clean; protocol.spec 10/10 including a cross-instance test; same mechanism as the #1697 fix
- Discussion: https://github.com/deepseek-ai/deepseek-harness/discussions/1993 (comment #discussioncomment-18031268)

## 12. #1992 - custom pi-ai routes lose catalog-known modalities (inheritance keyed by provider route, not model id)

- Branch: `fix/pi-ai-catalog-model-id-inheritance`
- Files: `packages/llm/llm-pi-ai/src/catalog.ts` (+ regression test)
- Fix: `resolveRouteModels` adds an id-level catalog fallback for input modalities (declared entry > provider catalog > global id catalog > route default); api/baseUrl stay route-owned so a foreign catalog entry never leaks its wire protocol; global id index cached per model id
- Evidence: `tsc -b tsconfig.host.json` clean; catalog.spec 53/53 with a custom-route vision-model regression test
- Discussion: https://github.com/deepseek-ai/deepseek-harness/discussions/1992 (comment #discussioncomment-18031307)

## 13. #2002 - one corrupt Zstandard session log crash-loops `dsh web` at boot

- Branch: `fix/session-list-isolate-corrupt`
- Files: `packages/session/session-persistence-jsonl/src/index.ts` (+ isolation tests)
- Fix: `listArtifacts` isolates per-file corruption (unreadable header frame, identity mismatch) with a loud warning and skips the file; targeted `load` still rejects; encoding mismatch and duplicate ids stay fatal
- Evidence: `tsc -b tsconfig.host.json` clean; session-persistence-jsonl 239/239 with a "good session stays reachable next to a corrupt one" assertion
- Discussion: https://github.com/deepseek-ai/deepseek-harness/discussions/2002 (comment #discussioncomment-18031351)

## 14. #1997 - Windows stop loses aborted-turn semantics (non-JSON-serializable AbortSignal.reason)

- Branch: `fix/agent-abort-reason-json-safe`
- Files: `packages/core/agent-loop/src/agent.ts` (+ cancel.spec regression test)
- Fix: normalize abort reasons before persisting turn/end; typed causes (user/parent/hook/disposed) pass through, DOMException/Error/unknown flatten to `{ kind: 'user' }`
- Evidence: `tsc -b tsconfig.host.json` clean; cancel.spec 32/32 with a DOMException-abort regression test
- Discussion: https://github.com/deepseek-ai/deepseek-harness/discussions/1997 (comment #discussioncomment-18031394)

## 15. #2009 - port-less browser Origins 403 on every /api POST (trust fence)

- Branch: `fix/api-trust-origin-hostname-portless`
- Files: `packages/client/connection/src/api-request-trust.ts` (+ regression test)
- Fix: Origin comparison uses `.hostname` instead of exact `.host`, mirroring the trustedHosts port-less convention; cross-hostname and opaque origins stay refused
- Evidence: `tsc -b tsconfig.host.json` clean; api-request-trust 11/11 with a #2009 regression case
- Discussion: https://github.com/deepseek-ai/deepseek-harness/discussions/2009 (comment #discussioncomment-18031625)

## 16. #2023 - pi-ai compat cannot override developer-role/store/reasoning-content switches

- Branch: `fix/pi-ai-compat-expose-role-store`
- Files: `packages/llm/llm-pi-ai/src/catalog.ts`, `src/config.ts` (+ regression test)
- Fix: expose `supportsDeveloperRole`, `supportsStore`, `requiresReasoningContentOnAssistantMessages` in PiAiCompatProfile/schema/resolution with model > route > catalog precedence
- Evidence: `tsc -b tsconfig.host.json` clean; catalog.spec 53/53 with a qwen-token-plan-style regression
- Discussion: https://github.com/deepseek-ai/deepseek-harness/discussions/2023 (comment #discussioncomment-18031734)

## 17. #2060 - session.prompt aborted by the fixed 30s unary timeout under host load

- Branch: `fix/prompt-user-paced-no-deadline`
- Files: `packages/host/apiproxy/src/fetch/client.ts` (+ regression test)
- Fix: classify session.prompt/subagent.prompt as `caller-signal-only` (user-paced, same as pickDirectory) so the fixed 30s deadline no longer applies; caller/connection aborts remain; other unary methods keep the bounded timeout
- Evidence: `tsc -b tsconfig.host.json` clean; fetch-carrier.spec 36/36 with a prompt-finishes-after-30s regression
- Discussion: https://github.com/deepseek-ai/deepseek-harness/discussions/2060 (comment #discussioncomment-18032208)

## 18. #2066 - minimal preset's str_replace_editor ignores the permission mode (bare fs)

- Branch: `fix/str-replace-editor-bare-fs-policy`
- Files: `packages/fs/tool-str-replace-editor/src/index.ts` (+ regression tests in `tests/tools.spec.ts`)
- Fix: resolve `ctx.sandboxPolicy` unconditionally; when the mounted fs does not confine (`fs-local`, `sandboxMode === undefined`), the editor enforces the per-call policy itself (read-only denies all three write commands; workspace-write contains to the `writableRoots` set) before delegating to the bare backend; confining backends keep delegating to `writeText`
- Evidence: `pnpm typecheck` clean; `tool-str-replace-editor` tools.spec 16/16 (14 existing + 2 new: read-only and workspace-write on a bare backend)
- Discussion: https://github.com/deepseek-ai/deepseek-harness/discussions/2066 (comment #discussioncomment-18032410)

## 19. #2075 - approval rejection loops the agent (rejection rendered as an ordinary error)

- Branch: `fix/approval-reject-stop-instruction`
- Files: `packages/core/tools/src/index.ts` (+ test assertion in `tests/tools.spec.ts`)
- Fix: `serviceAsk` maps `rejected` to an explicit stop-and-ask reason ("Stop and ask the user... do not retry or work around the rejection") instead of a generic tool error, so the model hands the turn back rather than re-requesting approval. Loop-level yield-on-rejection remains a separate follow-up.
- Evidence: `core/tools` tools.spec 136/136 with the updated rejection assertion
- Discussion: https://github.com/deepseek-ai/deepseek-harness/discussions/2075 (comment #discussioncomment-18032459)

> Superseded by #25 (`fix/approval-reject-conclude-turn`), which adds the loop-level yield.

## 20. #2081 - startup fails with a bare stripTypeScriptTypes error (Node/bun floor)

- Branch: `fix/node-version-startup-gate`
- Files: `apps/cli/src/bin.ts`, `apps/cli/src/runtime-check.ts` (+ `apps/cli/tests/runtime-check.spec.ts`)
- Fix: add a startup gate mirroring the declared `engines.node` range (`^22.19.0 || >=24.0.0`); unsupported runtimes (old Node, Node 23, bun) get a readable message instead of `Export named 'stripTypeScriptTypes' not found`. `--help`/`--version` still resolve before the gate.
- Evidence: `apps/cli` runtime-check.spec 5/5 + args.spec 6/6 green
- Discussion: https://github.com/deepseek-ai/deepseek-harness/discussions/2081 (comment #discussioncomment-18032592)

## 21. #2081 - second `dsh web` prints a raw EADDRINUSE stack

- Branch: `fix/webserver-eaddrinuse-message`
- Files: `packages/host/webserver/src/index.ts` (+ regression assertion in `tests/webserver.spec.ts`)
- Fix: special-case the `EADDRINUSE` listen error into an actionable message (`dsh web is already running: host:port is in use...`) while keeping the original errno as the error `cause`
- Evidence: `host/webserver` webserver.spec 2/2 green (including the fail-loud activation case)
- Discussion: https://github.com/deepseek-ai/deepseek-harness/discussions/2081 (comment #discussioncomment-18032613)

## 22. #2068 - a single duplicate seq makes the whole session log unloadable

- Branch: `fix/session-log-duplicate-seq-tolerance`
- Files: `packages/session/session-persistence-jsonl/src/format.ts` (+ regression test in `tests/jsonl.spec.ts`)
- Fix: tolerate an event whose seq equals the last accepted event's seq (a single reopen/append duplicate) by skipping it and recording it in `skippedDuplicateSeqs`, instead of refusing the whole log; a real seq gap and out-of-order corruption still fail
- Evidence: session-persistence-jsonl jsonl.spec 152/152 green with a duplicate-seq regression
- Discussion: https://github.com/deepseek-ai/deepseek-harness/discussions/2068

## 23. #2090 - streaming tool-call name/id blanked by explicit-null deltas

- Branch: `fix/llm-deepseek-toolcall-null-name`
- Files: `packages/llm/llm-deepseek/src/translate.ts` (+ regression test in `tests/translate.spec.ts`)
- Fix: guard tool-call `id`/`name` accumulation with `!= null` so providers that repeat `name: null`/`id: null` on continuation chunks (SGLang and other OpenAI-compatible streams) don't overwrite the first-chunk values; the same guard covers the emitted delta name
- Evidence: `llm-deepseek` translate.spec 31/31 green with an explicit-null regression
- Discussion: https://github.com/deepseek-ai/deepseek-harness/discussions/2090 (also #1713)

## 24. #2068 - sqlite backend parity for the duplicate-seq tolerance

- Branch: `fix/session-log-duplicate-seq-tolerance-sqlite`
- Files: `packages/session/session-persistence-sqlite/src/schema.ts` (+ regression test in `tests/sqlite.spec.ts`)
- Fix: mirror the jsonl loader tolerance in the sqlite committed-region scanner (skip one duplicate seq, track the expected-seq cursor separately from the row index so a skip is not misread as a torn tail)
- Evidence: sqlite scanRows describe 9/9 green with a duplicate-seq regression (one pre-existing Windows symlink test is unrelated)
- Discussion: https://github.com/deepseek-ai/deepseek-harness/discussions/2068

## 25. #2075 - a user-rejected approval ends the turn (loop-level)

- Branch: `fix/approval-reject-conclude-turn`
- Files: `packages/core/tools/src/index.ts` (+ test assertion in `tests/tools.spec.ts`)
- Fix: the `deny` decision for a user rejection now carries `concludesTurn`, and error results can forward that marker, so the agent loop commits the result and yields the turn instead of auto-continuing into a reject/retry loop; the stop-and-ask message from #19 is folded in. A sandbox denial still does not conclude (escalation stays available)
- Evidence: `core/tools` tools.spec 136/136 green with `concludesTurn: true` on the rejection result; `pnpm typecheck` clean
- Discussion: https://github.com/deepseek-ai/deepseek-harness/discussions/2075

## 26. #2106 - session persistence fails on Android/Termux (SELinux denies link)

- Branch: `fix/jsonl-link-eacces-rename-fallback`
- Files: `packages/session/session-persistence-jsonl/src/index.ts` (+ regression test in `tests/jsonl.spec.ts`)
- Fix: `materializePosix` falls back to `rename()` when `link()` is denied (`EACCES`/`EPERM`/`EXDEV`/`ENOTSUP`/`ENOSYS`), keeping `link()` as the primary no-clobber publish elsewhere; `rejectExistingLog` still runs first
- Evidence: session-persistence-jsonl jsonl.spec 152/152 green with a mock-link-EACCES fallback regression
- Discussion: https://github.com/deepseek-ai/deepseek-harness/discussions/2106

## 27. #1404 - `dsh plugin add` promotes a user-patch-inserted bundle and breaks the next boot

- Branch: `fix/plugin-reconcile-user-patch-dup-id`
- Files: `apps/cli/src/plugin.ts` (+ regression test in `apps/cli/tests/built-bin.e2e.ts`)
- Fix: `reconcilePlugins` reads the user patch layer's insert ids and skips promoting a bundle whose own patch inserts an id already present there (avoiding "duplicate loader entry id"); new deps and the "update gained dsh.bundle" case still promote
- Evidence: built-bin e2e #1404 regression + "update gained bundle" both pass; `pnpm typecheck` clean
- Discussion: https://github.com/deepseek-ai/deepseek-harness/discussions/1404

## 28. #1377 - `dsh plugin install` silently drops an unresolvable declared bundle

- Branch: `fix/plugin-reconcile-user-patch-dup-id` (second commit, same reconcile root as #27)
- Files: `apps/cli/src/plugin.ts` (+ regression test in `apps/cli/tests/built-bin.e2e.ts`)
- Fix: split resolution into bundle/plain/unresolvable and preserve an unresolvable declared bundle with a loud warning instead of splicing it out while `install` reports success
- Evidence: built-bin e2e #1377 regression passes; `pnpm typecheck` clean
- Discussion: https://github.com/deepseek-ai/deepseek-harness/discussions/1377

## 29. #2189 - typert-loader skips not-yet-mounted entries, breaking `/api/commands/list`

- Branch: `fix/typert-loader-activation-fiber-gate`
- Files: `packages/typert/loader/src/index.ts` (+ regression test in `packages/typert/loader/tests/loader.spec.ts`)
- Fix: `qualifies()` required `entry.fiber !== undefined`, so an entry whose plugin fiber had not mounted when typert-loader ran its activation scan was silently skipped; for `@deepseek-ai/dsh-commands` this meant the TYPERT manifest never registered and `/api/commands/list` returned 404. Registration is a wire-schema contract independent of the plugin service lifecycle, so `qualifies()` now accepts a declared, enabled entry while its teardown counter is zero, and only rejects an entry that is mid-dispose (preserving withdraw-on-unmount).
- Evidence: `packages/typert/loader` loader.spec 17/17 green with a "declared entry whose fiber is detached before activation" regression
- Discussion: https://github.com/deepseek-ai/deepseek-harness/discussions/2189 (first reported as #1740)

## 30. #2202 - complete zstd frame ending in a torn JSONL record is a false positive

- Branch: `fix/zstd-torn-record-tail-tolerate`
- Files: `packages/session/session-persistence-jsonl/src/index.ts` (+ updated test in `tests/zstd.spec.ts`)
- Fix: `readZstdPrefix` returns the committed prefix instead of throwing when all frames are structurally complete but the last frame ends mid-record (a transient `ZSTD_e_flush` writer state that self-heals); real corruption is still caught by `consumeEventLine`, and the incomplete-final-frame recovery path is unchanged
- Evidence: session-persistence-jsonl zstd.spec 80/80 green with the #2202 regression
- Discussion: https://github.com/deepseek-ai/deepseek-harness/discussions/2202

## 31. #1688 - a `__proto__` settings key mutates the prototype (silent data loss)

- Branch: `fix/settings-proto-key-own-property`
- Files: `packages/settings/settings/src/index.ts` (+ regression test in `tests/settings.spec.ts`)
- Fix: `cloneJsonShaped` and `mergeLayers` assign keys via `Object.defineProperty` instead of `target[key] = value`, so a `__proto__` key (valid JSON from a parsed body/config) stays an own data property instead of mutating the prototype; `mergeLayers` is exported for the test
- Evidence: `settings` settings.spec 90/90 green with a `__proto__` own-key regression
- Discussion: https://github.com/deepseek-ai/deepseek-harness/discussions/1688

## 32. #2271 - `eval --` bashism breaks every non-bash persistent shell

- Branch: `fix/persistent-bash-eval-portable`
- Files: `packages/shell/tool-bash-persistent/src/index.ts` (+ regression test in `tests/tools.spec.ts`)
- Fix: `wrapCommand` uses a leading space inside the quoted word (`eval $' <command>'`) instead of `eval --`, blocking option parsing portably; busybox ash no longer treats `--` as the command name, and dash-prefixed commands still aren't parsed as `eval` options
- Evidence: tool-bash-persistent wrapCommand regression (2 new cases) green
- Discussion: https://github.com/deepseek-ai/deepseek-harness/discussions/2271

## 33. #2273 - SDK runtime closure omits `dsh-mcp-client`

- Branch: `fix/sdk-runtime-mcp-client-closure`
- Files: `python/sdk-runtime/package.json` (+ `pnpm-lock.yaml`)
- Fix: add `@deepseek-ai/dsh-mcp-client: workspace:^` to the Python SDK node-mode deploy root, so a custom composition that mounts an external MCP server loads on the shipped runtime
- Evidence: `verify-runtime-closure` passes (110 workspace packages, closed graph)
- Discussion: https://github.com/deepseek-ai/deepseek-harness/discussions/2273

## 34. #1613 - Windows restricted-token runner surfaces opaque fast-fail exit codes

- Branch: `fix/windows-acl-fast-fail-hint`
- Files: `packages/sandbox/sandbox-windows-acl/src/runner.ts`, `src/fast-fail.ts` (+ `tests/fast-fail.spec.ts`)
- Fix: decode the high-signal NTSTATUS fast-fail family (`STATUS_DLL_INIT_FAILED`/`STATUS_STACK_BUFFER_OVERRUN`/`STATUS_ACCESS_VIOLATION`) into a stderr hint pointing at the restricted-token cause and the `danger-full-access`/per-tool override workaround; the exit code is still mirrored unchanged
- Evidence: `sandbox-windows-acl` fast-fail.spec 2/2 green; `pnpm typecheck` clean
- Discussion: https://github.com/deepseek-ai/deepseek-harness/discussions/1613

## 35. #2358 - max-tokens sourceEventSeqs expansion overflows the call stack (RangeError)

- Branch: `fix/apiproxy-paginate-group-start-loop`
- File: `packages/host/apiproxy/src/api-proxy.ts` (`paginate`)
- Fix: replace `Math.min(event.seq, ...sources)` with a bounded loop (`let groupStart = event.seq; for (const source of sources) if (source < groupStart) groupStart = source`)
- Evidence: a single max-tokens-truncated `assistant/message` can carry ~255,939 `sourceEventSeqs`; `Math.min(...sources)` exceeds V8's argument/call-stack limit → `RangeError: Maximum call stack size exceeded`, permanently breaking `session.history` (recurrence of #1593). The poster verified the loop over 2M elements returns the correct minimum without a stack error.
- Discussion: https://github.com/deepseek-ai/deepseek-harness/discussions/2358

## 36. #2342 - repair-path liveness re-check (prepareCore TOCTOU)

- Branch: `fix/repair-path-liveness-check`
- File: `packages/session/session-persistence/src/coordinator.ts` (`prepareCore`)
- Fix: re-check `ctx.sessions.get(id)` at the top of `prepareCore` (inside the per-id serialize reservation) so a session that becomes live after the public `prepare`/`load`/`inspect` check cannot receive synthetic repair closers that collide with the live writer's next real seq.
- Evidence: `build:lib:host` + `typecheck:contracts-ready` green (pre-push hook). Runtime verified: `session-persistence-jsonl` coordinator-contract suite **151/151 passed** (load/prepare/inspect/repair paths); sqlite suite 99/100 (the 1 failure is a Windows `symlink` EPERM environment issue, unrelated).
- Discussion: https://github.com/deepseek-ai/deepseek-harness/discussions/2342

## Session-corruption family analysis (in progress — argszero, #2342)

Root cause (argszero, building on #2342): the repair path (`prepareCore` →
`interruptedTurnClosers` → `commitRepair`) has no liveness check, so it assumes
"log balanced = process live" and can splice a synthetic `step/end` +
`turn/end {reason: interrupted}` closer onto a session a live writer is still
appending to — the synthetic seq (`last.seq + 1`) collides with the writer's
next real seq, and the strict `SessionLogScanner` check then refuses the log.

Verified locations (checked against `master` 47f9438):
- `packages/core/session/src/repair.ts:27` — `interruptedTurnClosers` synthesizes
  the closers with `seq = last.seq + 1`.
- `packages/session/session-persistence/src/coordinator.ts:892` — `prepareCore`
  calls `interruptedTurnClosers` without re-checking `ctx.sessions` liveness
  (the public `prepare`/`load` check it earlier, but `prepareCore` runs later
  inside the per-id serialize reservation — a TOCTOU gap).
- `packages/session/session-persistence/src/coordinator.ts` — `commitPrepared`
  checks `states.get(id).owner` but not the live `ctx.sessions` registry.
- Backends: `session-persistence-jsonl/src/index.ts:436` and
  `session-persistence-sqlite/src/index.ts:309` (`commitRepair`).

Family: #1333/#1452 (duplicate seq), #1497 (torn tail), #1473 (corruption),
#1586 (restore-writer vs live-writer), #2167 (duplicate append), #2342
(repair-writer vs live-writer). Shared defect: append/repair has no persistent
"who owns the log" contract.

Proposed fix order:
1. repair-path liveness/lease check — kill the class at the source (**landed**,
   patch #36, verified 151/151);
2. persistent seq-ownership check — **substantially already implemented**: the
   live append path asserts the "Contiguity contract" (`appendCore`,
   `coordinator.ts` ~699: `event.seq === state.cursor + i`), and the repair path
   is guarded by `isPreparedSourceCurrent` (`readStoredRevision === source.revision`).
   Remaining gap is narrow (a cross-process, revision-stable collision at
   `commitRepair` → `appendLines` write time);
3. reader self-heal for the synthetic-tail collision — still open, mirrors #2167.

Fix #3 precise pattern (reader self-heal):
- The synthetic tail ends with `turn/end` carrying `data.reason.kind === 'interrupted'`
  (and the immediately preceding synthetic `step/end` / interrupted `tool/result`
  rows share the same `time` and contiguous `seq = last.seq + 1`).
- On load, the collision shows as a **backwards seq** in `SessionLogScanner.consumeEventLine`
  (`format.ts`): after consuming the synthetic tail, the next real event has a
  seq less than `this.events.length`.
- Safe heuristic: when a backwards-seq gap is detected and the just-consumed
  events end in a synthetic `turn/end {reason: interrupted}`, drop only that
  synthetic tail (it is regenerable — `interruptedTurnClosers` rebuilds it) and
  continue consuming the real events. Never drop events that are not part of
  the synthetic tail.

Proposed minimal fix for the TOCTOU gap (fix #1, unverified — needs the
`session-persistence` coordinator-contract suite to run):

```ts
// coordinator.ts — inside prepareCore, before loadStored():
if (this.ctx.sessions.get(id) !== undefined) {
  throw new Error(`cannot prepare session "${id}" while it is live`)
}
```

Caveat: `prepare`/`load`/`inspect` already check liveness and retry in a loop;
throwing here must be reconciled with the per-id `serialize`/`reserve` retry
semantics (a throw may abort the loop instead of retrying). Verify against
`coordinator-contract.ts` before shipping.

Status: fix #1 (repair-path liveness) is landed as patch #36 (branch pushed;
build + typecheck + 151 coordinator-contract tests green). Fixes #2
(seq-ownership) and #3 (reader self-heal) are still analysis-only.

## Submit checklist (when the channel opens)

1. `git fetch upstream && git merge-base --is-ancestor 47f9438 upstream/master` — rebase if master moved.
2. Run the touched package's tests (each branch carries its regression tests).
3. Open one PR per branch, title prefixed with the issue/discussion number.
4. Attach the evidence snippet from the corresponding discussion.
5. Offer to maintain the patch across release trains (zoahdev commits to this).
