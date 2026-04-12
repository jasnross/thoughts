---
date: 2026-04-12T03:51:23-0400
topic: "Multi-repo git status and management tools"
tags: [research, git, multi-repo, cli, tui, developer-tools]
research_methods: [web]
status: complete
last_updated: 2026-04-12
last_updated_by: assistant
---

# Research: Multi-repo Git Status and Management Tools

**Date**: 2026-04-12T03:51:23-0400
**Research Methods**: web

## Research Question

What tools exist for showing the git status of all local repositories at a glance — working tree state (clean/dirty, changed file counts), current branch, ahead/behind main — with optional bulk management? Preference for CLI/TUI over desktop apps.

## Summary

The CLI space has two standout tools that complement each other: **gita** (registered repo list with rich status + bulk operations) and **multi-git-status** (zero-config directory scanner). For bulk task automation, **mani** fills the gap as a YAML-driven task runner. On the desktop side, **GitKraken Workspaces** is the only client with a genuine multi-repo dashboard; every other GUI treats multi-repo as a bookmark list.

## Detailed Findings

### CLI / TUI Tools

#### gita — Best All-Around CLI Tool

- **URL**: [github.com/nosarthur/gita](https://github.com/nosarthur/gita)
- **Type**: CLI (Python)
- **Stars**: ~1.8k | **Active**: yes (495+ commits)
- **Install**: `pip3 install -U gita` / `pipx install gita` / `uv tool install gita`

| Requirement | Support |
|---|---|
| Working tree state | Yes — symbols: `+` staged, `*` unstaged, `?` untracked, `$` stash |
| Current branch | Yes — shown by `gita ll` |
| Ahead/behind | Yes — color-coded (green=synced, purple=ahead, yellow=behind, red=diverged) |
| Bulk operations | Yes — `gita fetch`, `gita pull`, `gita super <any git command>`, async parallel execution, repo grouping |

**How it works**: You register repos with `gita add <path>` (or `gita add -a` to scan a directory). `gita ll` prints a compact status table. `gita super` proxies any git command to all (or grouped) repos in parallel.

**Limitations**: Status uses color-coding rather than numeric ahead/behind counts (customizable via `gita info`). Requires explicit repo registration.

#### multi-git-status (mgitstatus) — Best Zero-Config Scanner

- **URL**: [github.com/fboender/multi-git-status](https://github.com/fboender/multi-git-status)
- **Type**: CLI (shell script)
- **Stars**: ~528 | **Last release**: v2.3, October 2024
- **Install**: `brew install multi-git-status`

| Requirement | Support |
|---|---|
| Working tree state | Yes — "Uncommitted changes", "Untracked files", stash count |
| Current branch | Yes |
| Ahead/behind | Partial — "Needs push/pull" flags, no numeric counts |
| Bulk operations | No (status-only; `-f` flag runs `git fetch` before scan) |

**How it works**: Point it at a parent directory (`mgitstatus ~/code`) and it scans up to 2 levels deep for git repos. No registration or config needed.

**Limitations**: No numeric ahead/behind. Status-only — no bulk operations. Simple but limited information density.

#### mani — Best Task Runner for Repos

- **URL**: [github.com/alajmo/mani](https://github.com/alajmo/mani) | [manicli.com](https://manicli.com/)
- **Type**: CLI with built-in TUI
- **Stars**: ~682 | **Last release**: v0.32.0, February 2026
- **Install**: `brew install mani`

| Requirement | Support |
|---|---|
| Working tree state | Not built-in (achievable via `mani exec --all git status`) |
| Current branch | Not built-in |
| Ahead/behind | Not built-in |
| Bulk operations | Yes — YAML-configured tasks, parallel execution, tagging, filtering |

**How it works**: Define repos and custom tasks in `mani.yaml`. Run any command across all or tagged repos. Has a built-in TUI for interactive use.

**Limitations**: Not a status dashboard — it's a task runner. Requires YAML config setup. Best paired with gita or mgitstatus for the status view.

#### Other CLI Tools Evaluated

| Tool | URL | Notes |
|---|---|---|
| **gr (git-run)** | [github.com/mixu/gr](https://github.com/mixu/gr) | Solid features but stale maintenance, requires Node.js |
| **gitup** | [github.com/earwig/git-repo-updater](https://github.com/earwig/git-repo-updater) | Primarily a batch updater, not a status viewer |
| **myrepos (mr)** | [myrepos.branchable.com](https://myrepos.branchable.com/) | Battle-tested Perl tool, dated interface, supports SVN/Hg too |
| **Google repo** | [github.com/GerritCodeReview/git-repo](https://github.com/GerritCodeReview/git-repo) | Manifest-driven, heavyweight — for Android-scale monorepo setups |
| **lazygit** | [github.com/jesseduffield/lazygit](https://github.com/jesseduffield/lazygit) | Excellent single-repo TUI (~76k stars), no multi-repo support |

### Desktop / GUI Tools

#### GitKraken — Only Real Multi-Repo Desktop Client

- **URL**: [gitkraken.com/features/workspaces](https://www.gitkraken.com/features/workspaces)
- **Platform**: macOS, Windows, Linux
- **Pricing**: Free (public repos only) / Pro $8/mo / Advanced $12/mo / Business $16/mo

The only desktop client with a genuine multi-repo dashboard. Workspaces group repos with bulk fetch/pull, cross-repo PR visibility, and team sharing. Integrates GitHub, GitLab, Bitbucket, Azure DevOps.

#### Other Desktop Clients

| Tool | Multi-repo Support | Platform | Price |
|---|---|---|---|
| **Tower** | Repo grouping/filtering, no dashboard | macOS, Windows | $69-99/yr |
| **Fork** | Repo list with stats, no bulk ops | macOS, Windows | $59.99 one-time |
| **Sublime Merge** | Tabs per repo, no cross-repo view | macOS, Windows, Linux | $99 perpetual |
| **SourceTree** | Bookmarks with status indicators | macOS, Windows | Free |
| **GitHub Desktop** | Repo list, one-at-a-time view | macOS, Windows | Free |
| **SmartGit** | Deep per-repo tools, no workspace view | macOS, Windows, Linux | ~$6-8/mo |
| **GitButler** | No multi-repo (novel virtual branches) | macOS, Windows, Linux | Free/OSS |

None of these have a true multi-repo status dashboard — they treat multi-repo as a navigation/bookmark feature.

## Sources

- [GitHub - nosarthur/gita](https://github.com/nosarthur/gita)
- [GitHub - fboender/multi-git-status](https://github.com/fboender/multi-git-status)
- [GitHub - alajmo/mani](https://github.com/alajmo/mani)
- [mani CLI documentation](https://manicli.com/)
- [GitHub - mixu/gr](https://github.com/mixu/gr)
- [GitHub - earwig/git-repo-updater](https://github.com/earwig/git-repo-updater)
- [myrepos](https://myrepos.branchable.com/)
- [GitHub - GerritCodeReview/git-repo](https://github.com/GerritCodeReview/git-repo)
- [GitHub - jesseduffield/lazygit](https://github.com/jesseduffield/lazygit)
- [GitKraken Workspaces](https://www.gitkraken.com/features/workspaces)
- [GitKraken Pricing](https://www.gitkraken.com/pricing)
- [Tower Features](https://www.git-tower.com/features/all-features)
- [Fork Git Client](https://fork.dev)
- [Sublime Merge](https://www.sublimemerge.com/)
- [SourceTree](https://www.sourcetreeapp.com)
- [SmartGit](https://www.smartgit.dev/)
- [GitButler](https://gitbutler.com/)
- [multi-git-status on Homebrew](https://formulae.brew.sh/formula/multi-git-status)
- [gita on Terminal Trove](https://terminaltrove.com/gita/)
- [Hacker News: CLI tool to check git status of multiple projects](https://news.ycombinator.com/item?id=45925109)
- [rothgar/awesome-tuis](https://github.com/rothgar/awesome-tuis)

## Key Takeaways

1. **gita** is the strongest single tool for this use case — it covers all primary requirements (status, branch, ahead/behind) and adds bulk operations with parallel execution.
2. **multi-git-status** complements gita for discovery — zero-config scan of a directory tree when you don't want to register repos individually.
3. **mani** fills the gap if you need structured, repeatable bulk operations beyond what `gita super` offers.
4. No TUI application exists that provides a multi-repo dashboard. lazygit is single-repo only.
5. On the desktop side, **GitKraken Workspaces** is the only client with real multi-repo management; everything else is a bookmark list.

## Open Questions

- gita's ahead/behind is color-coded, not numeric. Worth checking if `gita info` customization can add exact counts — the docs suggest it may be possible via custom formatters.
- mani's built-in TUI mode may provide some of the dashboard experience — worth trying `mani exec --all git status` in TUI mode to see the output format.
- There may be newer Rust-based tools emerging in this space (e.g., tools referenced in Hacker News threads) that weren't well-established enough to surface in this research.
