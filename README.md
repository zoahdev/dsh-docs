# dsh-docs — PR-ready documentation proposals for DeepSeek Harness

Official PRs are currently closed in `deepseek-ai/deepseek-harness`; Discussions are the active channel. These documents mirror the official docs structure and are ready to submit the moment the channel opens:

```
docs/
  user/develop/basic/publish.md      # plugin publishing guide (preflight + allowBuilds + dist-tags)
  cookbook/adding-a-package.md       # adding a package to a profile
  troubleshooting.md                 # crash families, install failures, Windows-specific issues
  specs/community-market-registry.md # shared marketplace registry contract v2
  specs/plugin-first-class-commands.md # dsh plugin check + dsh doctor as first-class commands
  rfc/community-plugin-registry.md  # RFC #1846 (archived copy, maintainer feedback welcome)
  specs/upstream-patches.md         # cherry-pick-ready patch queue (#1697/#1842/#1856/#1861)
  security-policy.md                # responsible-disclosure draft (discussion #1932)
```

Every command in these documents was actually run on 2026-08-15 (Windows, Node 24, pnpm 11, dsh 0.1.0-rc.6). Sources:

- [dsh-plugin-doctor](https://github.com/zoahdev/dsh-plugin-doctor) (preflight, `--profile`, `--env`)
- [dsh-plugin-template](https://github.com/zoahdev/dsh-plugin-template) (runtime peer guard)
- Discussions #1629, #1697, #1719, #1734, #1763, #1774, #1782, #1789

## License

CC0 — free to adopt into the official docs.
---

# 中文说明

**面向 DeepSeek Harness 的、随时可提 PR 的文档提案包。**

官方现在关着 PR 通道，Discussions 是活跃渠道。这些文档完全对齐官方目录结构，通道一开就能直接提交：

```
docs/
  user/develop/basic/publish.md      # 插件发布指南（preflight + allowBuilds + dist-tags）
  cookbook/adding-a-package.md       # 往 profile 里加包
  troubleshooting.md                 # 崩溃族、安装失败、Windows 特有问题
  specs/community-market-registry.md # 共享市场注册表契约 v2
  specs/plugin-first-class-commands.md # dsh plugin check + dsh doctor 一等命令
  rfc/community-plugin-registry.md  # RFC #1846（存档副本，欢迎维护者反馈）
  specs/upstream-patches.md         # 可 cherry-pick 的补丁队列（#1697/#1842/#1856/#1861）
  security-policy.md                # 负责任披露草案（讨论 #1932）
```

文档里的每一条命令都在 2026-08-15 真实跑过（Windows、Node 24、pnpm 11、dsh 0.1.0-rc.6）。来源：

- [dsh-plugin-doctor](https://github.com/zoahdev/dsh-plugin-doctor)（preflight、`--profile`、`--env`）
- [dsh-plugin-template](https://github.com/zoahdev/dsh-plugin-template)（运行时 peer 守卫）
- Discussions #1629、#1697、#1719、#1734、#1763、#1774、#1782、#1789

## 许可

CC0——可自由并入官方文档。
