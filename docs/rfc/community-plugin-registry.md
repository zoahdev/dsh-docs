# RFC: Community plugin registry contract + first-class plugin checks

> Status: seeking maintainer feedback · Posted upstream at
> https://github.com/deepseek-ai/deepseek-harness/discussions/1846 ·
> 2026-08-15

## Summary

Three small, upstream-able pieces that make the plugin ecosystem safer to
use and easier to build on:

1. **Community Plugin Registry Contract v2** — one shared JSON schema for
   plugin catalogs, so a single entry drives web storefronts, CLI sync tools,
   and in-harness agent tools identically.
2. **`dsh plugin check <dir|tarball>`** — a pre-publish gate with stable exit
   codes (0 pass / 1 fixable / 2 not a plugin).
3. **`dsh doctor [--profile P]`** — runtime diagnostics for a broken profile,
   with machine-readable JSON output.

All three have running, CI-green reference implementations and are verified
against the current release train (0.1.0-rc.6).

## Problem

- The ecosystem has multiple marketplaces that each re-invent how a plugin
  entry is described (dsh-market, dsh-subscribe, DSH-Plugins-Marketplace,
  PR-based registries). Interop is impossible without a shared contract.
- Broken plugins (missing manifest, missing build artifact, wrong peer
  ranges) reach users because authors have no standard pre-publish gate.
- Support threads start from "plugin won't load" with no standard way to
  collect diagnostics.

## Proposal

### 1. Community Plugin Registry Contract v2

`registry.json` v2 (full schema: see
[community-market-registry.md](../specs/community-market-registry.md)):

- `id` unique and stable (owner-suffix disambiguation for duplicate names)
- `install.spec` is the authoritative pnpm spec
  (`github:owner/repo[#path:/sub]` or npm name) — never guessed from
  `homepage`
- `verified: true` is a narrow claim (curator exercised CI + release +
  install + runtime smoke), never a security audit
- verified entries must declare a `version`
- `source` preserves provenance for mirrored entries
- `count === plugins.length` as a cheap structural invariant

Validation exists today: `scripts/check-registry.mjs` in
[dsh-subscribe](https://github.com/zoahdev/dsh-subscribe).

### 2. `dsh plugin check`

Exit codes: `0` pass, `1` fixable issues, `2` not a plugin. Checks that exist
in reference implementations today:

- `cordis.patch.yml` exists and inserts the declared id
- peer ranges match the tested release train
- build artifact exists and matches `main`
- tool schemas compile under the real runtime
- packed tarball loads and registers tools through
  `apply()`/`ctx.tools.register`

### 3. `dsh doctor`

- exit `0` healthy / `1` warnings / `2` broken
- `--json` output with check groups (env / profile / session)
- shadow-profile detection, module-drift detection, `allowBuilds` hints

Reference: [dsh-plugin-doctor](https://github.com/zoahdev/dsh-plugin-doctor).

## Reference implementations & evidence

| Piece | Reference | Status |
| --- | --- | --- |
| Registry contract v2 + validator | [dsh-subscribe](https://github.com/zoahdev/dsh-subscribe) | 536 plugins, 20 verified, validator CI-green |
| In-harness marketplace consuming the contract | [dsh-subscribe v0.3](https://github.com/zoahdev/dsh-subscribe/releases/tag/v0.3.0) | real one-click install via HTTP verified in CI |
| `dsh plugin check` equivalent | [dsh-plugin-doctor](https://github.com/zoahdev/dsh-plugin-doctor) preflight | CI-green |
| `dsh doctor` equivalent | [dsh-plugin-doctor](https://github.com/zoahdev/dsh-plugin-doctor) `--profile --env --port` | CI-green |
| Template with runtime peer guard | [dsh-plugin-template](https://github.com/zoahdev/dsh-plugin-template) | CI-green |

Verification runs (all executed, not claimed):

```text
node scripts/check-registry.mjs           → registry OK: 536 plugins, 20 verified
node --test tests/cli.test.mjs            → 4/4 pass
node --test test/plugin.test.mjs test/market-server.test.mjs → 9/9 pass
pnpm pack + runtime-smoke                → packed tarball loads, tools + 12 routes execute
fresh DSH profile → dsh web boot → POST /dsh-subscribe/install → real dsh CLI exit 0
```

## Maintenance commitment

If upstream adopts any piece, zoahdev commits to: keeping the reference
implementations aligned with each release train, running compatibility
reports after every official release, and handing over clean patches the
moment the PR channel opens.

## Adoption path

1. Discussion feedback on this RFC (upstream #1846).
2. Clean patches submitted when upstream reopens PRs, or earlier if
   maintainers prefer discussion-first contributions.
3. dsh-docs keeps PR-ready documents mirrored to the official docs structure.

Related upstream threads: #1629, #1719, #1814, #1828.
