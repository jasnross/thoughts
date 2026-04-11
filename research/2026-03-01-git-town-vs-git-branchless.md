---
date: 2026-03-01T19:22:00Z
topic: "git-town vs git-branchless: Stacked Branch & DAG Workflow Tools"
tags: [research, web, git, stacked-branches, git-town, git-branchless, workflow]
status: complete
last_updated: 2026-03-01
last_updated_by: assistant
---

# Research: git-town vs git-branchless

**Date**: 2026-03-01T19:22:00Z

## Research Question

Evaluating git-town and git-branchless for two Git workflow needs:
1. **Stacked branches** - Keeping a stack of branches on top of each other as parent branches get changed/merged to main
2. **DAG branches** - Two or more unstacked branches are required for another branch (multi-parent dependencies)

Also covering: full feature comparison, workflow differences, and project health/longevity.

## Summary

**git-town** and **git-branchless** take fundamentally different philosophical approaches to enhancing Git workflows. git-town is a branch-level workflow automation tool that wraps existing Git commands with opinionated, high-level operations. git-branchless is a commit-centric suite inspired by Mercurial and Google/Meta internal tooling that introduces a new mental model for working with commits.

**For stacked branches**: Both tools handle this well, but differently. git-town manages named branch hierarchies with explicit parent-child tracking. git-branchless tracks commit DAGs and can work without named branches entirely.

**For DAG/multi-parent branches**: **Neither tool fully supports this.** git-town explicitly rejected multi-parent support (GitHub issue #3232). git-branchless's commit DAG model is more naturally suited to this but doesn't have first-class DAG branch support either.

**Project health**: git-town is significantly healthier - active releases every 2-4 weeks, currently at v22.6.0, with a responsive maintainer. git-branchless is still in alpha (v0.10.0), with releases slowing significantly since 2023 and the sole maintainer's activity declining.

---

## Detailed Findings

### 1. Stacked Branches

#### git-town's Approach

git-town provides **first-class stacked branch support** with dedicated commands:

| Command | Purpose |
|---------|---------|
| `git town hack <name>` | Create a new feature branch off main (start a stack) |
| `git town append <name>` | Create a child branch on top of current (extend stack) |
| `git town prepend <name>` | Insert a branch between current and its parent |
| `git town detach` | Remove a branch from a stack, making it standalone |
| `git town swap` | Swap position of current branch with its parent |
| `git town set-parent` | Change the parent of a feature branch |
| `git town up` / `git town down` | Navigate the stack |
| `git town branch` | Display branch hierarchy |
| `git town switch` | Visual branch switcher with VIM motions |

**Syncing stacks when parents change:**

```bash
git town sync --all    # Sync all local branches
git town sync --stack  # Sync all branches in current stack
```

The sync command:
- Pulls updates from remote main
- Detects and removes locally merged branches
- Rebases each stack branch onto its parent (propagating through hierarchy)
- When a parent branch is merged to main, child branches are automatically rebased onto main

**Shipping stacks:**
- `git town ship` merges the completed branch into main
- Ships only direct children of main - must ship ancestors first (or use `--to-parent`)
- `git town propose --stack` creates PRs for all branches in a stack, each targeting its parent

**Phantom conflict handling:**
git-town auto-resolves "phantom conflicts" (false conflicts from squash-merging) for merge and compress sync strategies. Also recommends enabling `git rerere` for automatic conflict resolution reuse.

**Branch relationship storage:**
Stored in git config as `git-town-branch.<branch>.parent=<branch>`. Automatically maintained - users don't manage these entries directly.

#### git-branchless's Approach

git-branchless works at the **commit level** rather than the branch level:

| Command | Purpose |
|---------|---------|
| `git sl` (smartlog) | Visualize commit DAG with status icons |
| `git sync` | Rebase all local commit stacks onto main |
| `git move` | Relocate subtrees in the commit graph |
| `git restack` | Fix abandoned commits after rewrites |
| `git next` / `git prev` | Navigate commit stacks |
| `git amend` | Amend HEAD and auto-restack descendants |
| `git hide` / `git unhide` | Control commit visibility in smartlog |

**Syncing stacks:**

```bash
git sync              # Rebase all draft commits onto main (local only)
git sync --pull       # Fetch main from remote first, then rebase
git sync 'stack()'    # Sync specific commits via revset expression
```

By default, `git sync` **skips rebases that would cause conflicts**, providing a summary of successes and failures. Use `--merge` to force conflict resolution.

**Commit tracking:**
Uses an event log stored in SQLite that records all repository mutations. Implements Mercurial's Changeset Evolution to track commit rewrites (old OID -> new OID). This enables:
- Detecting "abandoned" commits after amends/rebases
- Automatic restacking of descendant commits
- Full undo capability at the repository-state level

**Key difference from git-town:**
git-branchless doesn't require named branches at all. You can work in detached HEAD mode, making commits directly, and use `git smartlog` to visualize your commit tree. Branches are optional metadata, not the primary organizational unit.

### 2. DAG / Multi-Parent Branch Workflows

#### git-town: Explicitly Not Supported

git-town models branch relationships as a **tree** (single parent per branch), not a DAG. Multi-parent support was requested in [GitHub issue #3232](https://github.com/git-town/git-town/issues/3232) but closed as "Not Planned" due to:

1. **Guaranteed merge conflicts** from multiple parents making conflicting changes
2. **UI complexity** requiring multi-choice lists instead of simple selectors
3. **Command restructuring** (`set-parent` would need `add-parent`/`remove-parent`)
4. **Visualization complexity** (network graph vs. tree)
5. **Rebase ambiguity** (which commits belong to which parent?)
6. **Cascading edge cases** in `diff-parent`, `compress`, etc.

**Workarounds suggested by maintainer:**
- Ship blocking "base" branches quickly
- Tackle dependent work independently after base merges
- Use sequential rather than parallel development
- Evaluate whether refactors truly need a shared base

#### git-branchless: More Natural Fit but Not First-Class

git-branchless's commit DAG model can naturally represent multi-parent relationships since it tracks commits, not branches. However:

- There's no dedicated "create a branch depending on two other branches" command
- The `git move` command works with subtrees, not arbitrary DAG merges
- `git sync` rebases stacks linearly

In practice, you could manually create a merge commit that combines two feature branches, and git-branchless's smartlog would display it. But the tooling isn't specifically designed to manage or sync these structures automatically.

### 3. Feature Comparison

#### Features in Both Tools

| Feature | git-town | git-branchless |
|---------|----------|----------------|
| Stacked branch management | Yes (branch-level) | Yes (commit-level) |
| Sync/rebase automation | `git town sync` | `git sync` |
| Undo last operation | `git town undo` | `git undo` |
| Conflict resolution workflow | `continue`/`skip`/`undo` | `--merge` flag / manual |
| Branch visualization | `git town branch` | `git smartlog` |
| Stack navigation | `git town up`/`down` | `git next`/`prev` |
| Interactive branch switching | `git town switch` (VIM motions) | `git sw --interactive` |

#### Unique to git-town

| Feature | Description |
|---------|-------------|
| **Forge integration** | GitHub, GitLab, Bitbucket, Gitea, Forgejo - create PRs, infer parents from PRs |
| **`git town propose`** | Create PRs from CLI, `--stack` for all stack branches |
| **`git town ship`** | Merge completed branches from CLI |
| **Branch type system** | 8 types (feature, prototype, parked, contribution, observed, perennial, main) with distinct sync behaviors |
| **`git town compress`** | Squash all commits on a branch (or `--stack` for whole stack) |
| **Phantom conflict auto-resolution** | Detects and resolves false conflicts from squash-merging |
| **Offline mode** | Omit all network operations for airplane-mode development |
| **GitHub Action** | Visualizes stack relationships in PR descriptions |
| **Configuration file** | `git-town.toml` with sections for branches, hosting, sync strategy |
| **`git town swap`** | Swap branch position with parent |
| **`git town detach`** | Remove branch from stack |
| **`git town prepend`** | Insert branch into middle of stack |
| **Shell completions** | Bash, Zsh, Fish, PowerShell |
| **Multiple sync strategies** | merge, rebase, compress - configurable per branch type |

#### Unique to git-branchless

| Feature | Description |
|---------|-------------|
| **`git undo` (repository-level)** | Can undo any operation: amends, merges, rebases, checkouts, branch operations. Operates on full repository state, not just last command |
| **`git smartlog`** | Rich commit DAG visualization with status icons (diamond=public, circle=draft, X=hidden) |
| **Revset query language** | Query commits with expressions like `stack()`, usable across multiple commands |
| **`git restack`** | Automatically fix abandoned commits after rewrites |
| **In-memory operations** | Rebases without modifying working copy (performance optimization) |
| **Speculative merges** | Preview potential merge conflicts before they occur |
| **Anonymous branching** | Work without named branches in detached HEAD mode |
| **Divergent development** | Make frequent speculative commits and backtrack freely |
| **`git move`** | Relocate subtrees within commit graph |
| **Event log** | SQLite-based audit trail of all repository mutations |
| **Changeset Evolution** | Track commit rewrite history (inspired by Mercurial) |
| **`git test`** | Run tests against specific commits/revsets |
| **Large repo optimizations** | Sparse index, segmented changelog, O(log n) merge-base |
| **`git hide`/`git unhide`** | Control commit visibility |

### 4. Workflow Philosophy Differences

#### git-town: "Git, but easier"

> "Git Town provides additional Git commands that automate the creation, synchronization, shipping, and cleanup of Git branches."

- **Branch-centric**: Named branches are the primary unit of work
- **Workflow-agnostic**: Compatible with Git Flow, GitHub Flow, GitLab Flow, trunk-based
- **Additive**: "Keep using Git the way you do now, but with extra commands"
- **Team-oriented**: Deep integration with PR-based review systems
- **Opinionated automation**: Automates common sequences of Git commands
- **Written in Go**, distributed as single binary

#### git-branchless: "Mercurial's best ideas, on Git"

> "When you're working on a commit, incremental changes are amended into the commit, instead of being added as a new commit."

- **Commit-centric**: Individual commits are the primary unit of work
- **Paradigm shift**: Requires adopting a new mental model (branchless/patch-stack)
- **Individual-focused**: Client-side only, no server requirements
- **Power-user oriented**: More powerful but steeper learning curve
- **Inspired by**: Mercurial Changeset Evolution, Google's CitC, Meta's Sapling
- **Written in Rust**, performance-optimized for monorepo scale

### 5. Project Health

#### git-town

| Metric | Value |
|--------|-------|
| **Current version** | v22.6.0 (released Feb 28, 2026) |
| **GitHub stars** | ~3,100 |
| **Language** | Go (46.9%), Gherkin tests (51.7%) |
| **Open issues** | 26 |
| **Total releases** | 119 |
| **Total commits** | 4,886 |
| **Release cadence** | Every 2-4 weeks (10 releases in last 6 months) |
| **License** | MIT |
| **Maintainer** | Kevin Goslar (Principal Software Engineer) |
| **Corporate backing** | None (community-driven) |
| **Testing** | Extensive Cucumber/Gherkin E2E test suite |

**Assessment: VERY HEALTHY**
- Consistent, frequent releases with no signs of slowdown
- Active issue triage (latest issue Mar 1, 2026)
- Mature versioning (v22.x) indicating stability
- Well-documented with professional-grade website
- Multiple CI/CD workflows (E2E, unit, lint, Windows)

**Recent releases (last 6 months):**
- v22.6.0 - Feb 28, 2026
- v22.5.0 - Jan 26, 2026
- v22.4.0 - Dec 24, 2025
- v22.3.0 - Dec 14, 2025
- v22.2.0 - Oct 31, 2025
- v22.1.0 - Oct 14, 2025
- v22.0.0 - Sep 23, 2025

#### git-branchless

| Metric | Value |
|--------|-------|
| **Current version** | v0.10.0 (released Oct 14, 2024) |
| **GitHub stars** | ~4,000 |
| **Language** | Rust (99.9%) |
| **Contributors** | 37 |
| **Open issues** | 76 |
| **Total commits** | 2,003 |
| **Release cadence** | ~5-6 months recently (significant slowdown) |
| **License** | Apache-2.0 / MIT dual |
| **Maintainer** | Waleed Khan (arxanas) |
| **Corporate backing** | None (individual project) |
| **Status** | Alpha - "Be prepared for breaking changes" |

**Assessment: MODERATELY ACTIVE with concerning slowdown**

Positive signs:
- Still accepting contributions (37 contributors)
- Comprehensive CI (Linux, Windows, macOS, Nix)
- Active Discord community
- Higher star count than git-town

Concerns:
- **17+ months since last release** (Oct 2024 -> Mar 2026)
- Release cadence degraded: monthly in 2022 -> 3-6 months in 2023 -> 2 releases in 2024 -> none in 2025-2026
- 76 open issues with "minimal recent engagement"
- Still in alpha (v0.x) after 4+ years
- Single maintainer (bus factor = 1); profile indicated "abroad until June 3, 2025"
- Python PyPI wrapper abandoned
- Users report performance issues (60+ second operations, 10+ minute `git next` in large repos)

**Release history showing slowdown:**
- v0.6.0 - Nov 2022
- v0.7.0 - Mar 2023 (4 months)
- v0.8.0 - Aug 2023 (5 months)
- v0.9.0 - May 2024 (9 months)
- v0.10.0 - Oct 2024 (5 months)
- No release since Oct 2024

### 6. The Broader Stacked-Branch Tool Landscape

For context, here are the other notable tools in this space:

| Tool | Type | Notes |
|------|------|-------|
| **Graphite** | Commercial SaaS | Most feature-rich, $100+/month, GitHub-only |
| **Jujutsu (jj)** | New VCS | Google-backed, replaces Git entirely |
| **Sapling (sl)** | New VCS | Meta, Git-compatible, built-in stacking |
| **ghstack** | OSS CLI | Meta/PyTorch, commit-oriented, creates multiple PRs |
| **spr** | OSS CLI | Modular, single commit per PR |
| **git-machete** | OSS CLI | Manually-managed branch relationships |
| **git --update-refs** | Native Git | Git 2.38+, native stacking support (still tedious) |

git-town and git-branchless occupy different niches:
- **git-town** = "enhance your existing workflow" (lowest barrier, most compatible)
- **git-branchless** = "better paradigm on Git infrastructure" (between native Git and VCS replacement)

## Sources

### git-town Documentation
- [Introduction - Git Town 22.6](https://www.git-town.com/)
- [Stacked Changes Guide](https://www.git-town.com/stacked-changes.html)
- [Commands Reference](https://www.git-town.com/all-commands.html)
- [Configuration](https://www.git-town.com/configuration.html)
- [Configuration File](https://www.git-town.com/configuration-file.html)
- [Branch Types](https://www.git-town.com/branch-types.html)
- [Feature Sync Strategy](https://www.git-town.com/preferences/sync-feature-strategy.html)
- [GitHub Repository](https://github.com/git-town/git-town)
- [GitHub Releases](https://github.com/git-town/git-town/releases)
- [Multi-Parent Issue #3232](https://github.com/git-town/git-town/issues/3232)
- [GitHub Action](https://github.com/git-town/action)
- [Ship Command](https://www.git-town.com/commands/ship.html)
- [Sync Command](https://www.git-town.com/commands/sync)
- [Parent Branch Tracking](https://www.git-town.com/preferences/parent)
- [Offline Mode](https://www.git-town.com/preferences/offline.html)

### git-branchless Documentation
- [GitHub Repository](https://github.com/arxanas/git-branchless)
- [Architecture Wiki](https://github.com/arxanas/git-branchless/wiki/Architecture)
- [Tutorial](https://github.com/arxanas/git-branchless/wiki/Tutorial)
- [git smartlog](https://github.com/arxanas/git-branchless/wiki/Command:-git-smartlog)
- [git sync](https://github.com/arxanas/git-branchless/wiki/Command:-git-sync)
- [git restack](https://github.com/arxanas/git-branchless/wiki/Command:-git-restack)
- [Divergent Development](https://github.com/arxanas/git-branchless/wiki/Workflow:-divergent-development)
- [GitHub Releases](https://github.com/arxanas/git-branchless/releases)
- [Installation](https://github.com/arxanas/git-branchless/wiki/Installation)

### Comparisons & Analysis
- [git-stack comparison document](https://github.com/gitext-rs/git-stack/blob/main/docs/comparison.md)
- [The Stacking Workflow (stacking.dev)](https://www.stacking.dev/)
- [Branchless Git Workflows - quanttype](https://quanttype.net/posts/2023-05-15-branchless-workflows.html)
- [Branchless Git - Ben Congdon](https://benjamincongdon.me/blog/2021/12/07/Branchless-Git/)
- [Stacked PRs - Awesome Code Reviews](https://www.awesomecodereviews.com/best-practices/stacked-prs/)
- [Stacked Diffs - Pragmatic Engineer](https://newsletter.pragmaticengineer.com/p/stacked-diffs)

### Community Discussions
- [HN: Branchless Workflow for Git](https://news.ycombinator.com/item?id=34306260)
- [HN: Git Town](https://news.ycombinator.com/item?id=14385813)
- [HN: Ask HN - Alternatives to Graphite](https://news.ycombinator.com/item?id=40375310)
- [Lobsters: git-branchless](https://lobste.rs/s/rqphcq/git_branchless_high_velocity_monorepo)

## Key Takeaways

1. **For stacked branches**: git-town is the stronger, more practical choice. It has mature, well-tested stacked branch support with forge integration (PRs, merge), and is under very active development. git-branchless handles stacking at the commit level which is powerful but requires a paradigm shift.

2. **For DAG/multi-parent branches**: Neither tool supports this well. git-town explicitly rejected it. git-branchless's architecture is more amenable to it conceptually, but it lacks dedicated tooling. For this workflow, you may need to stick with manual Git operations or look at tools like Jujutsu which has a more flexible DAG model.

3. **Project longevity**: git-town is a safer bet for long-term dependability. 119 releases, consistent 2-4 week cadence, mature v22.x, responsive maintainer. git-branchless's 17+ month release gap, alpha status, and single-maintainer risk factor are real concerns.

4. **Adoption path**: git-town is significantly easier to adopt - it enhances your existing workflow without requiring a mental model shift. git-branchless requires committing to the "branchless" paradigm which is powerful but disruptive.

5. **If you want both tools' strengths**: Consider git-town for daily workflow automation + Jujutsu or Sapling for the commit-centric paradigm (both are more actively maintained than git-branchless and offer similar capabilities).

## Open Questions

- **git-branchless maintenance**: Is the project effectively abandoned, or is the maintainer planning a comeback? The last release was Oct 2024.
- **Jujutsu as alternative**: Jujutsu (jj) is Google-backed, actively maintained, and offers many of git-branchless's commit-centric features. Worth investigating as an alternative to git-branchless.
- **git-town DAG workarounds**: How well does the "stack them in order" workaround actually work in practice for DAG branch dependencies?
- **Native Git stacking**: How far has `git rebase --update-refs` (Git 2.38+) come? Could native Git + a lightweight wrapper be sufficient?
