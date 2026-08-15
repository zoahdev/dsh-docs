# Security Policy (draft, ready for upstream adoption)

> Status: community draft by zoahdev · 2026-08-15 · Prepared for adoption
> into `deepseek-ai/deepseek-harness` when the PR channel opens. Mirrors how
> the community already reports issues (discussion #1932).

## Reporting a vulnerability

DeepSeek Harness is developed in the open and its issue/PR trackers are
currently discussion-first. Please report security issues through:

1. **GitHub Discussions** — create a post under the closest category with:
   - Environment: OS, dsh version, surface (web / TUI / headless)
   - Reproduction: minimal steps or a PoC
   - Impact: what an attacker gains and who is affected
   - Evidence: logs, source line, sanitized screenshots
2. **Private disclosure** — once a private channel exists (security email or
   GitHub private vulnerability reporting), use it for high-severity issues
   before public disclosure.

Do not include live credentials, tokens, or personal data in public reports.

## Scope

- In scope: DeepSeek Harness runtime, agent loop, tool execution and
  permission surfaces, plugin loading/install paths, session persistence,
  sandboxing/workspace-write enforcement, and bundled official adapters.
- Out of scope: third-party plugins (report to the plugin author), social
  engineering against users, and issues already covered by a plugin's own
  security model.

## Expectations

- Maintainers aim to acknowledge reports within 3 business days.
- Community members may verify and reproduce reports; the ecosystem bug
  radar (dsh-ecosystem) tracks them for visibility.
- Fixes ship as normal releases or staged patches on the public fork when
  the PR channel is closed.

## Known threat-model notes (community-verified)

- Approval dialogs are consent/routing UX, not OS security boundaries
  (#1863, #1923): execution delegated to a user-privileged external shell can
  bypass them. OS-level sandboxing (restricted tokens/containers) is the
  intended enforcement layer.
- Static scans are heuristics; `node:sqlite`'s `DatabaseSync.exec()` is SQL,
  not shell execution (#1928).
