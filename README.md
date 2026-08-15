# dsh-docs — PR-ready documentation proposals for DeepSeek Harness

Official PRs are currently closed in `deepseek-ai/deepseek-harness`; Discussions are the active channel. These documents mirror the official docs structure and are ready to submit the moment the channel opens:

```
docs/
  user/develop/basic/publish.md      # plugin publishing guide (preflight + allowBuilds + dist-tags)
  cookbook/adding-a-package.md       # adding a package to a profile
  troubleshooting.md                 # crash families, install failures, Windows-specific issues
  specs/community-market-registry.md # shared marketplace registry contract v2
  specs/plugin-first-class-commands.md # dsh plugin check + dsh doctor as first-class commands
```

Every command in these documents was actually run on 2026-08-15 (Windows, Node 24, pnpm 11, dsh 0.1.0-rc.6). Sources:

- [dsh-plugin-doctor](https://github.com/zoahdev/dsh-plugin-doctor) (preflight, `--profile`, `--env`)
- [dsh-plugin-template](https://github.com/zoahdev/dsh-plugin-template) (runtime peer guard)
- Discussions #1629, #1697, #1719, #1734, #1763, #1774, #1782, #1789

## License

CC0 — free to adopt into the official docs.
