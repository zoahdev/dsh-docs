# Community Plugin Registry Contract v2

> Status: proposal · Candidate for upstream documentation · Reference implementation: [zoahdev/dsh-subscribe](https://github.com/zoahdev/dsh-subscribe) (`registry.json`)

## Why

The ecosystem already has several marketplaces (dsh-market, dsh-subscribe,
DSH-Plugins-Marketplace, PR-based registries). Each one re-invents how a
plugin entry is described and installed. A shared, minimal registry contract
makes every marketplace interoperable: one catalog entry can feed a web
storefront, a CLI sync tool, and in-harness agent tools.

## Contract (JSON Schema, v2)

```json
{
  "version": 2,
  "updated": "YYYY-MM-DD",
  "count": 535,
  "verifiedCount": 19,
  "note": "human-readable provenance and trust note",
  "categories": {
    "ui": { "en": "UI Enhancements", "zh": "UI 增强" }
  },
  "plugins": [
    {
      "id": "dsh-plugin-doctor",
      "name": "Plugin Doctor",
      "author": "zoahdev",
      "category": "dev",
      "description": "English description",
      "description_zh": "中文描述（可选）",
      "install": { "target": "git", "spec": "github:zoahdev/dsh-plugin-doctor" },
      "version": "1.5.0",
      "homepage": "https://github.com/zoahdev/dsh-plugin-doctor",
      "verified": true,
      "stars": 0,
      "tags": ["doctor", "diagnostics"],
      "source": { "name": "zoahdev", "url": "https://github.com/zoahdev/dsh-subscribe" }
    }
  ]
}
```

## Rules that make it safe to consume

1. **`id` is unique and stable.** Duplicate ids break subscription lists,
   localStorage keys, and tool lookups. When a catalog mirrors a source that
   has duplicate names (e.g. two `dsh-usage-stats`), disambiguate with an
   owner suffix (`dsh-usage-stats--Ychris12138`) and keep it stable.
2. **`install.spec` is a pnpm spec** — either `github:owner/repo[#path:/sub]`
   or an npm package name. Consumers must not guess from `homepage`; the
   registry declares the exact spec.
3. **`verified` is a narrow claim.** `true` means the curator exercised CI,
   release, install path, and a runtime smoke test. It is not a security
   audit. `false` means "listed for discovery".
4. **Verified entries declare a version.** No version, no verified badge.
5. **`source` preserves provenance.** Mirrored entries keep the original
   catalog name/URL so users can trace where an entry came from.
6. **`count` must equal `plugins.length`.** Cheap structural validation
   catches truncated writes.

## Validation

`node scripts/check-registry.mjs` in dsh-subscribe implements the contract
checks (unique ids/specs, verified-requires-version, valid spec shape,
count consistency). A future upstream version could ship this as
`dsh plugin registry check` or reuse it in marketplace CI.

## Adoption path

1. Upstream documents this contract in the official plugin guide.
2. `awesome-dsh-plugin.com/plugins.json` exposes both `en`/`zh` descriptions
   and install specs (already true today).
3. Marketplaces consume the same contract — a plugin accepted by the awesome
   list automatically appears everywhere with the same id and install spec.

## 中文摘要

这是给社区插件市场共用的注册表契约 v2：统一的 `id`/`install.spec`/
`verified`/`source` 语义，保证一个条目同时驱动网页商店、CLI 同步和
in-harness agent 工具。参考实现：dsh-subscribe `registry.json`。
