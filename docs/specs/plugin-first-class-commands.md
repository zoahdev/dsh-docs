# Proposal: first-class `dsh plugin check` and `dsh doctor`

> Status: proposal · Aligns with deepseek-harness #1629 (plugin scaffolding RFC), #1719 (doctor spec), #1814 (adoption proposal) · Reference implementations: [dsh-plugin-template](https://github.com/zoahdev/dsh-plugin-template), [dsh-plugin-doctor](https://github.com/zoahdev/dsh-plugin-doctor)

## Problem

Plugin authors currently learn about broken manifests, missing build
artifacts, peer mismatches, and runtime tool failures only after users report
them. There is no standard, machine-readable pre-publish gate, and no
standard way for users to diagnose a broken profile.

## Proposal

### 1. `dsh plugin check <dir|tarball>`

Pre-publish validation with stable exit codes:

- `0` — all checks pass
- `1` — fixable issues (missing manifest, bad peer range, missing build artifact)
- `2` — fatal (not a plugin at all)

Checks (each one exists in the reference implementation):

| Check | Reference |
| --- | --- |
| `cordis.patch.yml` exists and inserts the declared id | dsh-plugin-template CI |
| `package.json` peer ranges match the tested release train (`@deepseek-ai/dsh-tools` `0.1.0-rc.6`) | template `assertPeerCompatible` |
| Build artifact (`lib/index.js`) exists and matches `main` | template CI pack + install |
| Tool schema compiles under the real runtime (e.g. optional params must omit `required:false`) | dsh-subscribe runtime smoke |
| Packed tarball loads and registers tools through `apply()`/`ctx.tools.register` | template integration test |

### 2. `dsh doctor [--profile P] [--env] [--port]`

Runtime diagnostics for an installed profile:

- exit `0` healthy, `1` warnings, `2` broken
- `--json` machine-readable output with check groups (env/profile/session)
- shadow-profile detection, module-drift detection, `allowBuilds` hints

Reference: `dsh-plugin-doctor` implements this today (`preflight`,
`--profile`, `--env`, `--port`, JSON output, shadow detection).

## Why first-class

1. Authors get a gate in CI before `dsh plugin add` reaches users.
2. Support threads collapse to `dsh doctor --json` output.
3. Marketplaces (see the registry contract spec) can reject entries that
   fail `dsh plugin check` automatically.

## Adoption path

The PR channel for deepseek-harness is currently closed; the reference
implementations are release-ready and CI-green, so a clean patch can be
submitted the moment the channel opens. Forks with the proposed change are
kept current:
[zoahdev/deepseek-harness](https://github.com/zoahdev/deepseek-harness) ·
`fix/tool-runtime-scheduler-symbol-for`.

## 中文摘要

把 `dsh plugin check`（发布前检查）和 `dsh doctor`（运行时诊断）做成一等
CLI 命令，统一退出码与 JSON 输出，让插件作者在 CI 里把关、用户一条命令
排障、市场端自动过滤坏条目。参考实现已可运行并通过 CI。
