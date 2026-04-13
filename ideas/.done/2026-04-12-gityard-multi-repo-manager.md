# gityard: Multi-Repository Git Management Tool

## Problem Statement

Developers who work across many repositories lack a single tool that provides
both a clear, summarized view of repository state and the ability to safely
execute operations across multiple repos. Existing tools each solve part of the
problem — gita has rich status and bulk operations but requires Python, mgitstatus
does zero-config scanning but is status-only, mani runs tasks but has no built-in
status awareness — but nothing combines all three capabilities into a single,
dependency-free native binary.

The output usability problem is equally important: tools like gitup can run
`git status` across repositories, but the raw output is long, noisy, and
difficult to scan. What's needed is not just multi-repo execution but
**multi-repo summarization** — condensed, scannable output designed for humans
managing many repositories at once.

## Motivation

- **No single tool covers the full workflow.** The current best approach requires
  combining gita (status + operations) with mgitstatus (scanning) — two tools in
  two languages with different interfaces and mental models.
- **Python dependency is a friction point.** gita is the strongest contender but
  requiring a Python runtime (pip/pipx/uv) adds installation complexity and
  version management overhead that a native binary avoids entirely.
- **Raw multi-repo output is unusable at scale.** Running the same git command
  across 20+ repos produces a wall of text. Developers need summarized,
  structured output that surfaces what matters (which repos need attention) and
  hides what doesn't (repos that are clean and synced).
- **Unsafe bulk operations are risky.** Tools like `gita super` will happily
  `git pull` into a repo with uncommitted changes, or succeed in 19 repos and
  fail in the 20th, leaving the set in a partially-updated state. Safe
  multi-repo operations require pre-flight validation and transactional
  semantics.

## Context

### Existing Landscape

Research completed 2026-04-12 (see `research/2026-04-12-multi-repo-git-status-tools.md`):

| Tool | Language | Status View | Bulk Ops | Safety | Notes |
|------|----------|-------------|----------|--------|-------|
| **gita** | Python | Rich (symbols, color-coded ahead/behind) | Yes (parallel) | No pre-flight checks | Closest match overall |
| **mgitstatus** | Shell | Basic (flags, no numeric counts) | No (status-only) | N/A | Zero-config scanning |
| **mani** | Go | None built-in | Yes (YAML tasks) | No | Task runner, not status tool |
| **gr** | Node.js | Basic | Yes | No | Stale maintenance |
| **gitup** | Python | Minimal | Batch update only | No | Updater, not manager |
| **lazygit** | Go | Excellent | Yes | Yes | Single-repo only |

No existing tool provides: native binary + rich summarized status + safe bulk
operations + both registration and scanning.

### Engineering Approach

gityard will follow the engineering standards established in the
[agentspec](https://github.com/jasnross/agentspec) project:

- **Solid architecture from day one** — clean abstractions and composable
  building blocks that make the code easy to extend.
- **Type safety everywhere** — structs and strong types for repository state,
  command results, and configuration rather than raw strings.
- **Structured over fragile** — parse git output into typed structures rather
  than manipulating strings directly. Prefer serialization (serde) over manual
  formatting.
- **Pragmatic dependency choices** — skeptical about dependencies but willing to
  use reliable, well-maintained crates when they're the right tool (e.g., git2
  for repository operations, clap for CLI parsing, tabled/comfy-table for output
  formatting).
- **Tests from the start** — comprehensive test coverage to enable confident
  refactoring and feature development.

## Goals / Success Criteria

- [ ] Single native binary with zero runtime dependencies (installable via
      `cargo install`, Homebrew, or direct download)
- [ ] Rich, summarized status view across all managed repositories: working tree
      state (clean/dirty, file counts), current branch, ahead/behind counts
      relative to main/master and upstream
- [ ] Both repo registration (`gityard add <path>`) and directory scanning
      (`gityard scan <dir>`) for discovering repositories
- [ ] Safe execution of common git operations across multiple repos with
      pre-flight validation (e.g., block `pull` when working tree is dirty) and
      all-or-nothing semantics (if pre-flight fails for any repo, no repo is
      modified)
- [ ] Output designed for scannability: compact tables, color-coded status,
      grouping, filtering — surface repos that need attention, hide those that
      don't
- [ ] Cross-repo branch overview: list all branches and their status across all
      repositories, like a super `git branch --list` with richer information
- [ ] Open-source quality: clear error messages, documentation, reasonable
      defaults, configurable behavior

## Non-Goals (Out of Scope)

- **TUI / interactive dashboard** — gityard is a CLI with rich formatted output,
  not a persistent terminal UI. A TUI could be explored later but is not part of
  the initial scope.
- **Supporting every git operation** — the safe execution model will cover a
  curated subset of common operations (fetch, pull, push, checkout, stash)
  rather than proxying arbitrary git commands.
- **Remote repository management** — gityard manages local clones. It does not
  clone, fork, or interact with GitHub/GitLab APIs (at least initially).
- **Non-git VCS** — git only. No SVN, Mercurial, or other VCS support.
- **GUI / desktop application** — CLI only.

## Proposed Solution Sketch

### Core Concepts

- **Registry**: persistent store of known repositories (paths, aliases, groups/tags)
- **Scanner**: discovers git repositories under a given directory tree
- **Inspector**: reads repository state (branch, dirty status, ahead/behind, stashes)
  into typed structs
- **Executor**: runs git operations with pre-flight checks and all-or-nothing semantics
- **Presenter**: formats repository state into summarized, scannable CLI output

### Key Subcommands (Illustrative)

```
gityard add <path>          # Register a repo
gityard scan <dir>          # Discover and register repos under a directory
gityard status              # Summarized status of all repos (the main view)
gityard branches            # Cross-repo branch overview
gityard fetch [--all]       # Safe fetch across repos
gityard pull [--all]        # Safe pull (blocked if dirty)
gityard group <name> <repos...>  # Organize repos into named groups
gityard <cmd> --group <name>     # Target a specific group
```

### Safety Model

The "safe execution" model is a distinguishing feature:

1. **Pre-flight phase**: Before executing any operation, validate preconditions
   across ALL targeted repos (e.g., clean working tree for pull, branch exists
   for checkout).
2. **Dry-run reporting**: Show what would happen in each repo, highlighting any
   repos that would be skipped or blocked.
3. **All-or-nothing**: If pre-flight fails for ANY repo, the operation is aborted
   for ALL repos. The user sees which repos blocked the operation and why.
4. **Explicit override**: Users can opt into partial execution with a flag
   (e.g., `--allow-partial`) but the default is safe.

## Constraints

- **Rust** — the implementation language, chosen for native binary distribution,
  performance, and type safety. No runtime dependencies.
- **Cross-platform** — must work on macOS and Linux at minimum. Windows support
  is desirable but not a launch requirement.
- **Hybrid git access** — git2 crate (libgit2 bindings) for all read operations;
  shell out to the `git` CLI for network write operations (fetch, pull, push).
  See "Resolved Questions" for rationale.
- **Configurability as a design principle** — features should be configurable via
  both config file and CLI flags where possible, enabling users to create aliases
  for different workflows. Start with sensible defaults but design for
  customization.

## Resolved Questions

These were identified as open questions during initial ideation and have since
been researched and resolved.

### git2 vs. git CLI → Hybrid Approach

**Decision**: Use git2 for reads, shell out to `git` for writes.

**Rationale**: git2 (v0.20.4, actively maintained under `rust-lang` org) natively
supports all required read operations — `repo.head()` for branch,
`repo.graph_ahead_behind()` for ahead/behind counts, `repo.statuses()` for
dirty/clean + file counts, `repo.stash_foreach()` for stash count — with
in-process performance (~1ms vs. 5-30ms subprocess overhead per repo). However,
authentication is git2's major pain point for network operations: libgit2 doesn't
use OS keychain, git credential helpers, or ssh-agent automatically. Even
GitButler (the most prominent Rust git client) has a "use git executable" fallback
for auth. Shelling out to `git` for fetch/pull/push avoids this complexity
entirely since the system git handles auth natively.

### Parallelism Model → rayon

**Decision**: Use rayon for parallel repo operations.

**Rationale**: `.par_iter()` over the repo list with automatic work-stealing is
the ideal fit. Each thread opens its own `Repository` instance (git2 is safe with
separate instances on separate threads). No async machinery needed — this is
fundamentally CPU-bound/blocking work. tokio is wrong abstraction (designed for
async I/O), std::thread requires manual pool management. Used by cargo and fd for
similar patterns.

### Branch Status Definition → All Signals, Configurable

**Decision**: Show ahead/behind main, last commit date, merge status, and upstream
tracking state. Configurable via config file and CLI flags.

**Rationale**: All four signals serve different workflows. Ahead/behind shows
work-in-progress state, last commit date identifies stale branches, merge status
shows cleanup candidates, and upstream tracking shows push/pull state. Rather than
picking a subset, make all available and let users configure which columns appear
in output — either globally in config or per-invocation via flags. This supports
creating shell aliases for different views.

### Filtering and Targeting → Status Filters First

**Decision**: Start with status-based filters (`--dirty`, `--clean`, `--ahead`,
`--behind`, `--diverged`). Architecture should be extensible for tags, path globs,
and other filter types in the future.

**Rationale**: Status filters address the primary use case: "show me what needs
attention." Groups handle organizational targeting. Together these cover the
minimum viable filtering surface. Tags and path globs can be added later without
architectural changes if the filter system is designed as a composable trait/enum.

### Configuration → XDG + TOML

**Decision**: Store config at `~/.config/gityard/config.toml` and registry at
`~/.config/gityard/repos.toml`. Use the `dirs` crate for platform-appropriate
XDG path resolution. Support `GITYARD_CONFIG_DIR` env var override.

**Rationale**: XDG Base Directory spec is the modern standard for CLI tool
configuration. TOML is the natural format for the Rust ecosystem. Splitting config
and registry into separate files keeps concerns separate — settings don't get
interleaved with a potentially large repo list.

### Crate Name → Available

**Confirmed**: `gityard` is not taken on crates.io, and no conflicting
GitHub organizations or well-known projects exist under that name.

### Safe Operation Scope → Core 4

**Decision**: v1 includes fetch, pull, push, and checkout. Stash and merge
deferred to later versions.

**Rationale**: These four operations cover the daily multi-repo workflow: fetch to
update remote state, pull to sync, push to publish, and checkout for "switch all
repos to main" workflows. Each has clear pre-flight rules:
- **fetch**: always safe (no local state changes)
- **pull**: blocked if working tree is dirty
- **push**: blocked if upstream has diverged (require `--force` for override)
- **checkout**: blocked if working tree is dirty; warns if branch doesn't exist
  in some repos

Stash is a natural future addition (enables "stash, pull, unstash" workflows) but
adds conflict-handling complexity for unstash that warrants separate design.

### Output Format → Compact Table, Human-Readable

**Decision**: Compact aligned table (one repo per line) with symbols and color.
Human-readable only for v1; machine-readable output (`--json`) deferred.

**Design reference** for `gityard status`:

```
 REPO              BRANCH        STATUS   ↑↓ MAIN   ↑↓ REMOTE
 ✓ api-server       main          clean    ─          ─
 ✗ web-frontend     feat/login    3M 1?    +5 -2     +3
 ✓ shared-lib       main          clean    ─          ─
 ✗ deploy-scripts   fix/rollback  1M       +1        +1

 4 repos • 2 dirty • 2 ahead of main
```

Key conventions:
- ✓/✗ prefix for quick clean/dirty scanning
- `M` = modified, `?` = untracked, `A` = staged (following git's own conventions)
- `+N` = ahead, `-N` = behind for main and remote columns
- `─` for "nothing to report" (synced/not applicable)
- Summary footer with aggregate counts
- Columns are configurable via config file and CLI flags

**Rationale**: Compact table maximizes scannability at scale (20-50 repos). One
line per repo means the full picture fits on one screen. Machine-readable output
is a clean extension point — the data collection and presentation layers should be
separated in the architecture to make `--json` easy to add later.

## Future Considerations

- **gix as git2 alternative**: The pure-Rust gitoxide (`gix`) crate is maturing
  rapidly and may eventually replace git2 for read operations. Worth monitoring
  but not ready to depend on for all needed operations today.
- **Machine-readable output**: `--json` and/or `--porcelain` output modes for
  scripting integration. Architecture should separate data collection from
  presentation to make this straightforward.
- **TUI mode**: An optional interactive dashboard could be explored after the CLI
  is stable. The data layer built for CLI output would serve a TUI equally well.
- **Additional safe operations**: stash/unstash, merge, rebase — each with their
  own pre-flight rules. Candidates for v1.x releases.
- **Tags/labels on repos**: More flexible than groups alone — a repo can have
  multiple tags. Useful for filtering by language, team, project, etc.

## References

- Research: `thoughts/research/2026-04-12-multi-repo-git-status-tools.md`
- Architecture reference: [agentspec](https://github.com/jasnross/agentspec) — Rust project demonstrating preferred engineering patterns
- [gita](https://github.com/nosarthur/gita) — closest existing tool (Python)
- [multi-git-status](https://github.com/fboender/multi-git-status) — zero-config scanner (shell)
- [mani](https://github.com/alajmo/mani) — YAML task runner with repo management (Go)
- [git2 crate](https://crates.io/crates/git2) — Rust bindings for libgit2 (v0.20.4, `rust-lang` org)
- [auth-git2 crate](https://crates.io/crates/auth-git2) — SSH agent + credential helper wiring for git2
- [gitoxide/gix](https://github.com/GitoxideLabs/gitoxide) — pure-Rust git implementation (future alternative to git2)
- [rayon](https://github.com/rayon-rs/rayon) — data parallelism library for Rust
- [clap](https://crates.io/crates/clap) — Rust CLI argument parser
- [dirs](https://crates.io/crates/dirs) — platform-appropriate XDG directory paths
- [GitButler](https://github.com/gitbutlerapp/gitbutler) — real-world example of hybrid git2 + git CLI approach
