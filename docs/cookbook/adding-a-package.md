# Adding a package to a profile

> Draft ready for official adoption (2026-08-15). Commands verified on Windows (Node 24, pnpm 11, dsh 0.1.0-rc.6).

## Install sources

```sh
# npm package
dsh plugin --profile web add <npm-package>

# git repository (runs the package's prepare script)
dsh plugin --profile web add github:owner/repo

# local tarball
dsh plugin --profile web add ./my-plugin-1.0.0.tgz
```

## Verify the install

```sh
dsh --profile web --dump-config   # the plugin id must appear in the composed config
```

## If a git install is blocked

pnpm 11 blocks git-hosted `prepare` scripts by default. The dsh CLI prints the exact `allowBuilds` key to add to the profile's `pnpm-workspace.yaml`; add it and re-run.

## Before opening a PR to a curated list

Run `dsh-plugin-doctor preflight .` (community implementation of `dsh plugin check`). Lists such as awesome-dsh-plugin and dsh-marketplace require a single `dsh plugin add` install path.
