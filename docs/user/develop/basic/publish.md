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
dsh-plugin-doctor preflight .
```

`preflight` also runs build, pack, installs into a **fresh** DSH profile, and verifies the plugin id appears in the composed config. Exit `0` = pass, `1` = fail, JSON via `--json`.

## Known install traps

### pnpm blocks git-installed `prepare` scripts

`dsh plugin add github:owner/repo` may fail with `ERR_PNPM_GIT_DEP_PREPARE_NOT_ALLOWED`. Add the exact key the CLI prints to the profile's `pnpm-workspace.yaml`:

```yaml
allowBuilds:
  my-plugin@https://codeload.github.com/owner/repo/tar.gz/<sha>: true
```

Then re-run the add.

### npm `latest` dist-tag may be stale

`@deepseek-ai/dsh-tools` currently has `latest` = `0.0.1-rc.1` (broken train) while the ecosystem runs `0.1.0-rc.6` under `next`. Plugin authors should declare peers explicitly (`^0.1.0-rc.6`) and document the tested version. Users should install `@deepseek-ai/dsh-tools@next` when a plugin requires it.

### Peer version mismatch is silent on pnpm

pnpm may link an older RC into a plugin's peer slot with only a generic warning. Add a runtime guard in `apply()` that resolves the actual linked `@deepseek-ai/dsh-tools` version and throws an actionable error on mismatch (see dsh-plugin-template).

## Checklist

- [ ] `dsh.bundle` + `dsh.bundle.patch` declared
- [ ] `files` allowlist includes build output
- [ ] `prepare` script present
- [ ] at least one model-facing tool registered
- [ ] `dsh-plugin-doctor preflight .` passes (or documented exception)
- [ ] CI installs the packed tarball into a fresh profile and boots `dsh web`
