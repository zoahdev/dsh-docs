# Troubleshooting

> Draft ready for official adoption (2026-08-15). All cases verified on Windows/Linux with dsh 0.1.0-rc.6.

## Every tool call crashes: `Cannot read properties of undefined (reading 'prepare')`

**Root cause**: two physical copies of `@deepseek-ai/dsh-tools` in one process. A profile-installed plugin's hoisted copy shadows the host copy; the internal scheduler key is a per-module-instance `Symbol`, so agent-loop reads `undefined`. See discussion #1697/#1763/#1782.

**Diagnose**:

```sh
dsh-plugin-doctor --profile ~/.dsh/profiles/web
pnpm why @deepseek-ai/dsh-tools
```

**Workaround**: pin the host copy in the profile's `package.json` as a `link:` dependency (path to the host's dsh-tools), then `pnpm install` in the profile.

**Fix (staged)**: `Symbol.for` + scheduler protocol-version guard — https://github.com/zoahdev/deepseek-harness/tree/fix/tool-runtime-scheduler-symbol-for

## Install fails with `ERR_PNPM_GIT_DEP_PREPARE_NOT_ALLOWED`

pnpm blocks git-hosted `prepare` scripts. Add the exact key the dsh CLI prints to the profile's `pnpm-workspace.yaml` `allowBuilds`, then re-run.

## Install fails with peer conflict / ERESOLVE

Check the actual dsh-tools version (`pnpm why @deepseek-ai/dsh-tools`). npm's `latest` tag may point at the old `0.0.1-rc.1` train; install `@deepseek-ai/dsh-tools@next` or upgrade dsh to 0.1.0-rc.6 explicitly.

## `dsh web` fails to boot on Linux CI: missing pty.node

The upstream npm CLI lacks the linux-x64 `pty.node` prebuild (#1686). Run the web boot smoke on a Windows runner, or limit Linux smoke to install + `--dump-config`.

## Windows restricted mode breaks TLS clients

`CreateRestrictedToken` strips schannel credentials, so Windows `curl.exe`, .NET HttpClient, and `Invoke-WebRequest` fail with `SEC_E_NO_CREDENTIALS` (#1789). Node-based clients use OpenSSL and are unaffected; prefer `node -e "fetch(...)"` or an OpenSSL-built curl inside the sandbox.
