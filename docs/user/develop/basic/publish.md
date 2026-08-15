# Publishing a DeepSeek Harness plugin

> Draft ready for official adoption (2026-08-15). Commands verified on Windows (Node 24, pnpm 11, dsh 0.1.0-rc.6).

## Bundle manifest

Every installable plugin declares a bundle manifest in `package.json`:

```json
{
  "main": "lib/index.js",
  "files": ["lib", "cordis.patch.yml", "README.md", "LICENSE"],
  "scripts": { "prepare": "pnpm run build", "build": "tsc -p tsconfig.json" },
  "dsh": { "bundle": { "patch": "./cordis.patch.yml" } }
}
```

`cordis.patch.yml` inserts the plugin row into a profile composition:

```yaml
- insert:
    - id: my-plugin
      name: my-plugin
```

## Preflight checklist (run every release)

```sh
# structural checks: manifest / patch / entry / files
dsh-plugin-doctor check .        # 'check' = RFC #1846 surface; 'preflight' is the same pipeline
```

`check`/`preflight` also runs build, pack, installs into a **fresh** DSH profile, and verifies the plugin id appears in the composed config. Exit `0` = pass, `1` = fixable issues, `2` = not a plugin, JSON via `--json`.

## Known install traps

### pnpm blocks git-installed `prepare` scripts

`dsh plugin add github:owner/repo` may fail with `ERR_PNPM_GIT_DEP_PREPARE_NOT_ALLOWED`. Add the exact key the CLI prints to the profile's `pnpm-workspace.yaml`:

```yaml
allowBuilds:
  my-plugin@https://codeload.github.com/owner/repo/tar.gz/<sha>: true
```

Then re-run the add.

### npm `latest` dist-tag may be stale

As of 2026-08-15, `@deepseek-ai/dsh-tools` has `latest` = `0.1.0-rc.6` (the broken `0.0.1-rc.1` tag was retired). Plugin authors should still declare peers explicitly (`^0.1.0-rc.6`), document the tested version, and re-verify on every release train (see the ecosystem release-compat reports).

### Peer version mismatch is silent on pnpm

pnpm may link an older RC into a plugin's peer slot with only a generic warning. Add a runtime guard in `apply()` that resolves the actual linked `@deepseek-ai/dsh-tools` version and throws an actionable error on mismatch (see dsh-plugin-template).

## Distribution

After the package is installable, get it listed where users look:

1. **awesome-dsh-plugin** — open a PR adding one line to `README.md` and `README.zh.md` under the matching category (see `contributing.md`). The site and most marketplaces mirror this list.
2. **Community registries** — dsh-subscribe mirrors the awesome snapshot automatically; entries with a curated `verified` layer can request audit via issue.
3. **Marketplaces** — PR-based registries such as DSH Marketplace accept a `plugin.json` per plugin; run their validator before submitting.
4. **npm** — publish the prebuilt tarball so installs skip the `allowBuilds` build-approval step.

## Checklist

- [ ] `dsh.bundle` + `dsh.bundle.patch` declared
- [ ] `files` allowlist includes build output
- [ ] `prepare` script present
- [ ] at least one model-facing tool registered
- [ ] `dsh-plugin-doctor preflight .` passes (or documented exception)
- [ ] CI installs the packed tarball into a fresh profile and boots `dsh web`
