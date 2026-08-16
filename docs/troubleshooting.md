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

## `dsh web` crashes at boot with `Unexpected token` (profile manifest BOM)

**Root cause**: the profile's `package.json` starts with a UTF-8 BOM (`EF BB BF`).
`readProfileManifest` parses with `JSON.parse(readFileSync(path, 'utf8'))`
(`packages/boot/app-boot/src/profile.ts`), and a leading `U+FEFF` is rejected —
the error message gives no hint that the file has a BOM (#1842). Files saved by
PowerShell `>` redirects, Notepad "UTF-8 with BOM", or editor defaults are
common sources.

**Diagnose**:

```sh
dsh-plugin-doctor --profile ~/.dsh/profiles/web --json
# → manifest-bom: fail
```

**Fix**: re-save `package.json` as UTF-8 without BOM (or strip the first three
bytes). Staged upstream patch:
https://github.com/zoahdev/deepseek-harness/tree/fix/profile-manifest-bom-strip

## A session gets stuck after a tool error: "must be followed by tool messages responding to each tool_call_id"

**Root cause**: this is the second-order effect of the #1697 dual-instance
crash family. When `prepare` throws (`undefined.prepare`), the assistant turn
is left with an unpaired `tool_call`; the session state machine then rejects
every following message (#1841).

**Diagnose**:

```sh
dsh-plugin-doctor --profile ~/.dsh/profiles/web --json
# → profile-shadow: fail means the #1697 precondition is present
```

**Fix**: apply the #1697 fix (staged branch above), reinstall the profile so
the host `@deepseek-ai/*` copy resolves first, then start a new session — the
stuck session cannot unwind the unpaired tool call.

## `npm install` inside `profiles/node_modules` wipes the runtime dependency tree

**Root cause**: `$DSH_HOME/profiles/node_modules` is a package.json-less
fallback tree that `healProfilesModuleFallback` re-links to the deployment
anchor on every launch. Running `npm install <pkg>` in that directory makes npm
treat every existing package as extraneous and **prune** it (510+ packages in
one report), after which `dsh web` fails with `ERR_MODULE_NOT_FOUND` and does
not self-heal (#2081).

**Do not** run `npm install`/`pnpm install` directly in a profile directory.
Install plugins through the CLI so the profile tree is updated safely:

```sh
dsh plugin --profile web add <package>       # npm / git / tarball
```

**Recover**: restore the pruned tree from the deployment anchor (the install
that owns `dsh`) by re-running the deployment install, or recreate the profile
and re-add plugins. `dsh-plugin-doctor --profile <dir>` flags a pruned tree via
the `profile-deps` check before the next boot.

## npm 11 blocks native postinstall scripts (`allow-scripts`)

npm 11 defaults `allow-scripts=false`, so native modules (`koffi`, `node-pty`,
`dsh-subprocess-local`) never build and `dsh web` misbehaves with no clear
message (#2081). Install with the exact allow-list instead:

```sh
npm install -g --allow-scripts=@deepseek-ai/dsh-subprocess-local,koffi,node-pty,@google/genai,protobufjs @deepseek-ai/dsh
```

The `dsh plugin` path uses pnpm, which has the same protection under a
different name (`allowBuilds` in the profile's `pnpm-workspace.yaml`); the CLI
prints the exact key to add when a git-hosted `prepare` script is blocked.

## `stripTypeScriptTypes` missing: Node too old, or running under bun

`dsh` depends on `node:module.stripTypeScriptTypes`, which needs Node >= 22.6
(the project declares `^22.19.0 || >=24.0.0`). Under bun, or on an older Node,
the failure surfaces as a bare `Export named 'stripTypeScriptTypes' not found
in module 'node:module'` (#2081).

**Fix**: run `dsh` under Node 22.19+ (or 24+). bun is not supported. Verify
with `node --version`.

## `dsh web` again prints a raw `EADDRINUSE` stack

When the port is already in use by another `dsh web`, the listen error is not
special-cased and surfaces as a bare Node stack (#2081). Until a friendly
single-instance message lands upstream, check the port first:

```sh
dsh-plugin-doctor --env --port 3080   # reports whether 127.0.0.1:3080 is in use
```

Either stop the existing instance or launch with a different port.

## Windows sandbox: Schannel HTTPS fails with `SEC_E_NO_CREDENTIALS`

Under the default Windows sandbox, any client that uses the Windows credential
subsystem for TLS (Schannel) fails inside the confined process, while
OpenSSL-based clients work fine (#2184):

| Client | TLS backend | Result |
|---|---|---|
| `curl.exe` | Schannel | exit 35 `SEC_E_NO_CREDENTIALS` |
| `Invoke-WebRequest` | Schannel | connection reset |
| node / python | OpenSSL | 200 OK |

This is the restricted token, not a network policy (plain HTTP and TCP are
unaffected). `sandbox-windows-acl/src/token.ts` creates the token with the
`DISABLE_MAX_PRIVILEGE` flag, which disables every privilege except
`SeChangeNotifyPrivilege`; Schannel's `AcquireCredentialsHandle` then cannot
obtain a credential. The write boundary is unaffected by this — it is enforced
by the `WRITE_RESTRICTED` flag plus the restricting-SID allowlist, not by
privilege reduction.

**Workaround**: run HTTPS through a non-Schannel client inside the sandbox
(node `fetch`, `python urllib`), or disable the sandbox for that call. The
proper fix is to retain the LSA/Schannel-path privilege (`SeImpersonatePrivilege`)
instead of the blanket `DISABLE_MAX_PRIVILEGE`; see discussion #2184.
