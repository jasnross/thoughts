# gityard: Multi-Repository Git Management Tool — Implementation Plan

## Overview

Build `gityard`, a Rust CLI that provides rich summarized status across many git
repositories and safe bulk git operations with pre-flight validation. The tool
combines repo registration/scanning, a `git2`-powered inspector, compact
color-coded table output, and an all-or-nothing executor for fetch/pull/push/checkout.

Repository location: `/Users/jasonr/Workspace/jasnross/gityard`

## Current State Analysis

No code exists yet — this is a greenfield project. The design is fully specified
in the idea document (`thoughts/ideas/2026-04-12-gityard-multi-repo-manager.md`)
with all major architectural decisions resolved: hybrid git2/CLI approach, rayon
parallelism for reads / `std::thread::scope` for network I/O, XDG+TOML
configuration, and a defined safety model.

### Key Discoveries:

- The `agentspec` project at `/Users/jasonr/Workspace/jasnross/agentspec/`
  establishes the engineering patterns to follow: edition 2024, strict clippy
  lints (5 denied groups + 4 restriction lints), modern module layout, `anyhow`
  + `thiserror` error handling, SHA-pinned CI actions, and integration tests
  exercising the real binary
- `git2` v0.20.x provides all required read APIs: `repo.head()`,
  `repo.statuses()`, `repo.graph_ahead_behind()`, `repo.stash_foreach()`
- No existing Rust tool covers this exact feature set — the closest is `gita`
  (Python) which lacks pre-flight safety and numeric ahead/behind counts

## Desired End State

A working `gityard` binary (installable via `cargo install --path .`) that:

1. Manages a registry of git repositories via `add`, `scan`, `list`, `remove`
2. Shows a compact, color-coded status table across all registered repos
3. Supports filtering by status (`--dirty`, `--clean`, `--ahead`, `--behind`)
4. Executes `fetch`, `pull`, `push`, `checkout` with pre-flight validation and
   all-or-nothing semantics
5. Organizes repos into named groups with group-targeted operations
6. Shows a cross-repo branch overview via `branches`
7. Passes `cargo fmt --check`, `cargo clippy --all-targets`, `cargo test`, and
   `cargo deny check licenses` in CI

### Verification:

```
# Build and install
cargo install --path .

# Register repos and see status
gityard scan ~/Workspace/jasnross
gityard status
gityard status --dirty

# Safe operations
gityard fetch
gityard pull --group rust

# Branch overview
gityard branches
```

## What We're NOT Doing

- **Distribution**: Homebrew formula, release-please, cross-platform release
  workflows — deferred to a follow-up plan after v0.1.0 is working
- **TUI / interactive dashboard**: CLI with rich formatted output only
- **Machine-readable output**: `--json` / `--porcelain` deferred
- **Stash/merge/rebase operations**: Only fetch, pull, push, checkout in v1
- **Remote API integration**: No GitHub/GitLab API calls
- **Windows support**: macOS and Linux only for v1

## Implementation Approach

Five phases, each producing a working increment:

1. **Scaffold & Registry** — project setup + repo management commands
2. **Inspector** — git2-based repo state reading into typed structs
3. **Status & Presenter** — the main `gityard status` view with filtering
4. **Safe Executor** — pre-flight validated bulk operations
5. **Groups & Branches** — repo grouping and cross-repo branch overview

Phases are ordered by dependency: each builds on the previous. Phase 3 is the
first "useful" milestone (you can see status across repos). Phase 4 adds the
safety-differentiating feature. Phase 5 rounds out the v0.1.0 feature set.

---

## Phase 1: Project Scaffold & Registry

### Overview

Set up the Rust project with agentspec-quality tooling and implement the repo
registry: `add`, `scan`, `list`, `remove` subcommands that persist to
`~/.config/gityard/repos.toml`.

### Changes Required:

#### 1. Project Configuration

**File**: `Cargo.toml`

- [x] Create with `edition = "2024"`, package metadata (`name`, `version = "0.1.0"`,
      `description`, `license = "MIT"`)
- [x] Add dependencies: `anyhow = "1"`, `clap = { version = "4", features = ["derive"] }`,
      `dirs = "6"`, `serde = { version = "1", features = ["derive"] }`,
      `thiserror = "2"`, `toml = "0.8"`, `walkdir = "2"`
- [x] Add dev-dependencies: `tempfile = "3"`, `assert_cmd = "2"`, `predicates = "3"`
- [x] Add clippy lint configuration matching agentspec: deny `complexity`,
      `pedantic`, `perf`, `style`, `suspicious` at `priority = -1`; deny
      `expect_used`, `panic`, `unwrap_used`, `wildcard_enum_match_arm`; allow
      `implicit_hasher`, `missing_errors_doc`, `must_use_candidate`,
      `similar_names`, `struct_field_names`, `module_name_repetitions`

```toml
[lints.clippy]
complexity = { level = "deny", priority = -1 }
pedantic = { level = "deny", priority = -1 }
perf = { level = "deny", priority = -1 }
style = { level = "deny", priority = -1 }
suspicious = { level = "deny", priority = -1 }
expect_used = "deny"
panic = "deny"
unwrap_used = "deny"
wildcard_enum_match_arm = "deny"
implicit_hasher = "allow"      # requires generics over BuildHasher — unnecessary for internal APIs
missing_errors_doc = "allow"   # pub fns are cross-module, not a documented public API yet
must_use_candidate = "allow"   # flags every pure fn — too noisy until the API stabilizes
similar_names = "allow"        # flags unambiguous pairs like `dst`/`dest`
struct_field_names = "allow"   # flags structs where all fields share a suffix (e.g. `ahead_*`, `behind_*`)
module_name_repetitions = "allow"  # gityard-specific: e.g. `registry::RepoEntry` flagged as repetitive
```

**File**: `clippy.toml`

- [x] Create with test ergonomics: `allow-expect-in-tests = true`,
      `allow-panic-in-tests = true`

**File**: `rustfmt.toml`

- [x] Create matching agentspec: `imports_granularity = "Module"`,
      `group_imports = "StdExternalCrate"`

**File**: `deny.toml`

- [x] Create with license allowlist starting from agentspec's list (Apache-2.0,
      BSD-2-Clause, BSD-3-Clause, ISC, MIT, MIT-0, MPL-2.0, Unicode-3.0,
      Unicode-DFS-2016, Zlib)
- [x] After adding `git2` in Phase 2, run `cargo deny check licenses` to
      identify any additional licenses required by `git2`/`libgit2-sys` and
      their transitive dependencies — add only what actually fails

**File**: `.github/workflows/ci.yml`

- [x] Create CI workflow: fmt check, clippy, test, cargo-deny license check
- [x] Pin all action SHAs matching agentspec pattern (with human-readable tag comments)

**File**: `.gitignore`

- [x] Create with `/target` entry

**File**: `CLAUDE.md`

- [x] Create project-level instructions: build commands, commit conventions
      (conventional commits), clippy rules, module layout convention, design
      principles carried from agentspec

#### 2. CLI Structure

**File**: `src/cli.rs`

- [x] Define `Cli` struct with `#[derive(Parser)]` and `#[command(subcommand)]`
- [x] Define `Command` enum with initial subcommands:
  - `Add { paths: Vec<PathBuf> }` — register repos
  - `Scan { dir: PathBuf, #[arg(long, default_value_t = 3)] depth: usize }` — discover repos
  - `List` — show registered repos
  - `Remove { repos: Vec<String> }` — unregister repos (by path or alias)
  - Additional subcommands added in later phases (Status in Phase 3, executor
    commands in Phase 4, Group/Branches in Phase 5)

```rust
#[derive(Debug, Parser)]
#[command(name = "gityard", about = "Multi-repository git management")]
#[command(version)]
pub struct Cli {
    #[command(subcommand)]
    pub command: Command,
}

#[derive(Debug, Subcommand)]
pub enum Command {
    /// Register one or more repositories
    Add {
        /// Paths to git repositories
        paths: Vec<PathBuf>,
    },
    /// Discover and register repositories under a directory
    Scan {
        /// Directory to scan for git repositories
        dir: PathBuf,
        /// Maximum directory levels below the scan dir to search (default: 3)
        #[arg(long, default_value_t = 3)]
        depth: usize,
    },
    /// List all registered repositories
    List,
    /// Unregister repositories by path or alias
    Remove {
        /// Repository paths or aliases to remove
        repos: Vec<String>,
    },
}
```

#### 3. Configuration & Registry

**File**: `src/config.rs`

- [x] Define `Config` struct (deserialized from `config.toml`):
  - `status: StatusConfig` (default columns, color preferences)
  - `scan: ScanConfig` (default depth)
  - `defaults: DefaultsConfig` (main branch names: `["main", "master"]`)
- [x] Implement `Config::load()` that reads from `config_dir()/config.toml`,
      falling back to defaults if file doesn't exist
- [x] Use `dirs::config_dir()` with `gityard` subdirectory; support
      `GITYARD_CONFIG_DIR` env var override
- [x] Add `#[serde(default, deny_unknown_fields)]` on all config structs

```rust
/// Resolves the gityard configuration directory.
///
/// Precedence: `GITYARD_CONFIG_DIR` env var > `dirs::config_dir()/gityard`.
pub fn config_dir() -> Result<PathBuf> {
    if let Ok(dir) = std::env::var("GITYARD_CONFIG_DIR") {
        return Ok(PathBuf::from(dir));
    }
    dirs::config_dir()
        .map(|d| d.join("gityard"))
        .ok_or_else(|| anyhow!("could not determine config directory"))
}
```

**File**: `src/registry.rs`

- [x] Define `RepoEntry` struct: `path: PathBuf`, `alias: Option<String>`,
      `groups: Vec<String>`
- [x] Define `Registry` struct wrapping `Vec<RepoEntry>` with TOML
      serialization to `repos.toml`
- [x] Implement `Registry::load(config_dir)` — reads from `repos.toml`, returns
      empty registry if file doesn't exist
- [x] Implement `Registry::save(config_dir)` — writes to `repos.toml`, creating
      the config directory if needed
- [x] Implement `Registry::add(path)` — canonicalizes path, validates it's a git
      repo (check for `.git` directory in Phase 1; upgrade to
      `git2::Repository::open` validation in Phase 2 when `git2` is added),
      auto-generates alias from directory name, deduplicates by canonical path
- [x] Implement `Registry::remove(identifier)` — remove by path or alias
- [x] Implement `Registry::list()` — returns slice of entries
- [x] Add `#[serde(deny_unknown_fields)]` on `RepoEntry`

```toml
# ~/.config/gityard/repos.toml
[[repos]]
path = "/Users/jasonr/Workspace/jasnross/agentspec"
alias = "agentspec"
groups = []

[[repos]]
path = "/Users/jasonr/Workspace/jasnross/gityard"
alias = "gityard"
groups = []
```

#### 4. Scanner

**File**: `src/scanner.rs`

- [x] Implement `scan(dir, max_depth) -> Vec<PathBuf>` that walks a directory
      tree using `walkdir` and returns paths containing `.git` directories
- [x] Skip directories that are themselves inside `.git`
- [x] Return canonical paths for consistent deduplication with registry

#### 5. Binary Entry Point

**File**: `src/main.rs`

- [x] Parse CLI with `Cli::parse()`
- [x] Load config and registry
- [x] Dispatch to command handlers: `add`, `scan`, `list`, `remove`
- [x] `add`: validate paths are git repos, add to registry, save, print confirmation
- [x] `scan`: run scanner, show discovered repos, add all to registry, save
- [x] `list`: print registered repos (path + alias)
- [x] `remove`: remove from registry, save, print confirmation
- [x] Return `anyhow::Result<()>` from `main()`

**File**: `src/lib.rs`

- [x] Declare public modules: `config`, `registry`, `scanner`

#### Tests for This Phase

**File**: `src/registry.rs` (unit tests module)

- [x] Test `Registry::add` deduplicates by canonical path
- [x] Test `Registry::add` rejects non-git directories
- [x] Test `Registry::remove` by path and by alias
- [x] Test `Registry::load` returns empty registry when file doesn't exist
- [x] Test round-trip: save then load preserves entries

**File**: `src/scanner.rs` (unit tests module)

- [x] Test `scan` discovers git repos in nested directories (create temp dirs
      with `git init`)
- [x] Test `scan` respects depth limit
- [x] Test `scan` skips non-repo directories

**File**: `src/config.rs` (unit tests module)

- [x] Test `Config::load` returns defaults when no config file exists
- [x] Test `Config::load` parses valid TOML
- [x] Test `Config::load` rejects unknown fields

**File**: `tests/cli.rs` (integration tests)

- [x] Test `gityard add <path>` with a real temp git repo
- [x] Test `gityard list` shows registered repos
- [x] Test `gityard remove` removes a repo
- [x] Test `gityard scan <dir>` discovers repos
- [x] Use `assert_cmd` crate to run the binary and check stdout/stderr/exit code

### Success Criteria:

#### Automated Verification:

- [x] `cargo fmt --check` passes
- [x] `cargo clippy --all-targets` passes with no warnings
- [x] `cargo test` passes (unit + integration tests)
- [x] `cargo deny check licenses` passes
- [x] `gityard add <path-to-git-repo>` registers the repo
- [x] `gityard scan <dir>` discovers git repos recursively
- [x] `gityard list` displays registered repos
- [x] `gityard remove <repo>` removes a repo from the registry
- [x] Registry persists across invocations (data in `~/.config/gityard/repos.toml`)

---

## Phase 2: Inspector

### Overview

Implement the git2-based inspector that reads repository state into typed
structs: current branch, working tree status (dirty/clean with file counts),
ahead/behind counts relative to the main branch and upstream remote, and stash
count. This is the data layer that powers the status view.

### Changes Required:

#### 1. Dependencies

**File**: `Cargo.toml`

- [x] Add `git2 = "0.20"` to dependencies
- [x] Add `rayon = "1"` to dependencies

#### 2. Inspector Module

**File**: `src/inspector.rs`

- [x] Define `RepoState` struct with all fields needed for the status view:

```rust
/// Complete state of a single repository, read via git2.
pub struct RepoState {
    /// Canonical path to the repository root.
    pub path: PathBuf,
    /// Alias from registry (for display).
    pub alias: String,
    /// Current branch name (or HEAD detached description).
    pub branch: String,
    /// Whether the working tree has any changes.
    pub is_dirty: bool,
    /// Count of files with staged changes.
    pub staged_count: usize,
    /// Count of files with unstaged modifications.
    pub modified_count: usize,
    /// Count of untracked files.
    pub untracked_count: usize,
    /// Commits ahead of the main branch (main/master).
    pub ahead_main: Option<usize>,
    /// Commits behind the main branch.
    pub behind_main: Option<usize>,
    /// Commits ahead of the upstream tracking branch.
    pub ahead_remote: Option<usize>,
    /// Commits behind the upstream tracking branch.
    pub behind_remote: Option<usize>,
    /// Number of stash entries.
    pub stash_count: usize,
}
```

- [x] Define `InspectError` enum with `thiserror::Error`:
  - `OpenFailed { path: PathBuf, source: git2::Error }`
- [x] Implement `inspect(entry: &RepoEntry, main_branches: &[String]) -> Result<RepoState>`:
  1. Open repo with `git2::Repository::open(&entry.path)`
  2. Read current branch from `repo.head()` → `reference.shorthand()`;
     if HEAD is detached, use `repo.head().peel_to_commit()` to produce a
     description like `"HEAD detached at abc1234"` — this is a normal state,
     not an error, so the `branch` field holds the description and
     `ahead_main`/`behind_main` are still computed from the HEAD OID
  3. Read working tree status from `repo.statuses(None)` → iterate entries,
     count by status flags (`INDEX_*` for staged, `WT_*` for modified, `NEW` for
     untracked)
  4. Find the main branch: try each name in `main_branches` config, look for
     `refs/heads/{name}` — first match wins
  5. Compute ahead/behind main: `repo.graph_ahead_behind(head_oid, main_oid)`
  6. Find upstream tracking branch: `repo.find_branch(branch_name, BranchType::Local)`
     → `branch.upstream()` → get its target OID
  7. Compute ahead/behind remote: `repo.graph_ahead_behind(head_oid, upstream_oid)`
  8. Count stashes: `repo.stash_foreach(|_, _, _| { count += 1; true })`
- [x] Implement `inspect_all` using `rayon::par_iter()` for parallel inspection,
      returning `Vec<(RepoEntry, Result<RepoState, InspectError>)>` (each result
      paired with its entry for error reporting)

```rust
pub fn inspect_all(
    entries: &[RepoEntry],
    main_branches: &[String],
) -> Vec<(RepoEntry, Result<RepoState, InspectError>)> {
    entries
        .par_iter()
        .map(|entry| {
            let result = inspect(entry, main_branches);
            (entry.clone(), result)
        })
        .collect()
}
```

#### 3. Wire Into Library

**File**: `src/lib.rs`

- [x] Add `pub mod inspector;`

#### Tests for This Phase

**File**: `src/inspector.rs` (unit tests module)

All tests create real git repos in temporary directories using `git2::Repository::init`:

- [x] Test clean repo: `is_dirty` is false, all counts are 0
- [x] Test dirty repo: create a file, verify `untracked_count = 1`, `is_dirty = true`
- [x] Test staged changes: `git add` a file, verify `staged_count = 1`
- [x] Test modified file: modify a tracked file, verify `modified_count = 1`
- [x] Test branch detection: create and checkout a branch, verify `branch` field
- [x] Test ahead/behind main: create commits on a branch, verify `ahead_main` count
- [x] Test stash count: create a stash entry, verify `stash_count = 1`
- [x] Test missing main branch: repo with no `main` or `master`, verify
      `ahead_main` and `behind_main` are `None`
- [x] Test detached HEAD: checkout a commit directly, verify `branch` field
      contains a description like `"HEAD detached at abc1234"` and ahead/behind
      main is still computed
- [x] Test upstream tracking: set up a local remote + tracking branch, verify
      `ahead_remote`/`behind_remote`

### Success Criteria:

#### Automated Verification:

- [x] `cargo fmt --check` passes
- [x] `cargo clippy --all-targets` passes
- [x] `cargo test` passes — all inspector unit tests green
- [x] `inspect()` correctly reads branch, dirty status, file counts, ahead/behind,
      and stash count from real git repos in temp directories

---

## Phase 3: Status Command & Presenter

### Overview

Implement the primary `gityard status` command: load the registry, inspect all
repos in parallel, format results into a compact color-coded table with summary
footer, and support status-based filtering flags.

### Changes Required:

#### 1. Dependencies

**File**: `Cargo.toml`

- [x] Add `colored = "3"` to dependencies (for terminal colors)

#### 2. Filter Module

**File**: `src/filter.rs`

- [x] Define `StatusFilter` struct with boolean flags:

```rust
pub struct StatusFilter {
    pub dirty: bool,
    pub clean: bool,
    pub ahead: bool,
    pub behind: bool,
    pub diverged: bool,
}
```

- [x] Implement `StatusFilter::matches(&self, state: &RepoState) -> bool`
  - If no flags set, match all repos
  - If any flag set, repo must match at least one active flag
  - `dirty`: `state.is_dirty`
  - `clean`: `!state.is_dirty`
  - `ahead`: `matches!(ahead_main, Some(n) if n > 0)` or same for `ahead_remote`
  - `behind`: `matches!(behind_main, Some(n) if n > 0)` or same for `behind_remote`
  - `diverged`: both ahead and behind some reference

#### 3. Presenter Module

**File**: `src/presenter.rs`

- [x] Implement `format_status_table(states: &[&RepoState]) -> String` that produces
      the compact table format from the idea doc:

```
 REPO              BRANCH        STATUS   ↑↓ MAIN   ↑↓ REMOTE
 ✓ api-server       main          clean    ─          ─
 ✗ web-frontend     feat/login    3M 1?    +5 -2      +3
 ✓ shared-lib       main          clean    ─          ─
 ✗ deploy-scripts   fix/rollback  1M       +1        +1

 4 repos · 2 dirty · 2 ahead of main
```

- [x] Column formatting:
  - REPO: alias, left-aligned, prefixed with `✓` (green) or `✗` (red)
  - BRANCH: current branch name, left-aligned
  - STATUS: `clean` (green) or `{N}M {N}S {N}?` compact format
    (`M` = unstaged modifications from `modified_count`,
     `S` = staged changes from `staged_count`,
     `?` = untracked from `untracked_count`)
  - ↑↓ MAIN: `+N` (cyan) ahead, `-N` (yellow) behind, `─` if synced/N/A
  - ↑↓ REMOTE: same format as main column
- [x] Compute column widths dynamically from data
- [x] Render summary footer: total repos, dirty count, ahead-of-main count
- [x] Handle inspection errors gracefully: show repo with `error` status in red

- [x] Implement `format_repo_errors(errors: &[(RepoEntry, InspectError)]) -> String`
      for repos that failed inspection (e.g., deleted repo still in registry)

#### 4. Status CLI Arguments

**File**: `src/cli.rs`

- [x] Add `Status(StatusArgs)` variant to `Command` enum
- [x] Define `StatusArgs`:

```rust
#[derive(Debug, Parser)]
pub struct StatusArgs {
    /// Show only dirty repositories
    #[arg(long)]
    pub dirty: bool,
    /// Show only clean repositories
    #[arg(long)]
    pub clean: bool,
    /// Show only repositories ahead of main or remote
    #[arg(long)]
    pub ahead: bool,
    /// Show only repositories behind main or remote
    #[arg(long)]
    pub behind: bool,
    /// Show only repositories that have diverged
    #[arg(long)]
    pub diverged: bool,
}
```

#### 5. Wire Status Command

**File**: `src/main.rs`

- [x] Add `Command::Status` handler:
  1. Load registry
  2. Run `inspect_all` across all entries (parallel via rayon)
  3. Separate successes from errors
  4. Apply `StatusFilter` from CLI args
  5. Format with presenter and print to stdout
  6. If any inspection errors, print error summary to stderr

**File**: `src/lib.rs`

- [x] Add `pub mod filter;` and `pub mod presenter;`

#### Tests for This Phase

**File**: `src/filter.rs` (unit tests)

- [x] Test no flags set matches all repos
- [x] Test `--dirty` matches only dirty repos
- [x] Test `--clean` matches only clean repos
- [x] Test `--ahead` matches repos ahead of main or remote
- [x] Test `--behind` matches repos behind
- [x] Test `--diverged` matches repos both ahead and behind
- [x] Test multiple flags: `--dirty --ahead` matches repos matching either

**File**: `src/presenter.rs` (unit tests)

- [x] Test table output formatting with known `RepoState` values
- [x] Test column alignment with varying alias/branch lengths
- [x] Test summary footer counts
- [x] Test error display for failed inspections
- [x] Test `clean` status renders as green, dirty as red — presenter functions
      should accept a `colors_enabled: bool` parameter (or the presenter struct
      holds this config) so tests pass `false` without relying on the global
      `colored::control::set_override`, which is not thread-safe across parallel
      tests

**File**: `tests/cli.rs` (integration tests — extend)

- [x] Test `gityard status` with registered repos shows table output
- [x] Test `gityard status --dirty` filters to dirty repos only
- [x] Test `gityard status` with no registered repos shows informative message

### Success Criteria:

#### Automated Verification:

- [x] `cargo fmt --check` passes
- [x] `cargo clippy --all-targets` passes
- [x] `cargo test` passes — filter and presenter unit tests green
- [x] `gityard status` produces compact table with correct status for all
      registered repos
- [x] `gityard status --dirty` shows only dirty repos
- [x] `gityard status` with no repos prints a helpful message, not an empty table

#### Manual Verification:

- [ ] Visual inspection of `gityard status` output against real repositories
      confirms readability — colors render correctly, columns align, summary
      footer is accurate (manual-only: subjective visual quality)

---

## Phase 4: Safe Executor

### Overview

Implement the safety-differentiating feature: pre-flight validated bulk git
operations with all-or-nothing semantics. Covers `fetch`, `pull`, `push`, and
`checkout`. Each operation has defined pre-flight rules; if any repo fails
pre-flight, no repo is modified (unless `--allow-partial` overrides).

### Changes Required:

#### 1. Operation Definitions

**File**: `src/operation.rs`

- [x] Define `Operation` enum:

```rust
pub enum Operation {
    Fetch,
    Pull,
    Push { force: bool },
    Checkout { branch: String },
}
```

- [x] Define `PreflightResult` for each repo:

```rust
pub enum PreflightResult {
    /// Repo passes pre-flight — operation can proceed.
    Ready,
    /// Repo is blocked — operation cannot proceed.
    Blocked { reason: String },
    /// Repo will be skipped (e.g., branch doesn't exist for checkout).
    Skipped { reason: String },
}
```

- [x] Implement pre-flight rules per operation:
  - `Fetch`: always `Ready` (no local state changes)
  - `Pull`: `Blocked` if working tree is dirty
  - `Push`: `Blocked` if behind upstream (`behind_remote` is `Some(n)` where
    `n > 0`) unless `force` is true — covers both diverged and strictly-behind
    cases. If `behind_remote` is `None` (no upstream tracking branch), the push
    is `Ready` (user may be pushing a new branch)
  - `Checkout`: `Blocked` if working tree is dirty; `Skipped` if branch doesn't
    exist in the repo (with warning)

- [x] Implement `preflight(op: &Operation, state: &RepoState) -> PreflightResult`

#### 2. Executor Module

**File**: `src/executor.rs`

- [x] Define `ExecArgs` CLI struct:

```rust
#[derive(Debug, Parser)]
pub struct ExecArgs {
    /// Proceed even if some repos fail pre-flight (skip blocked repos)
    #[arg(long)]
    pub allow_partial: bool,
    /// Target a specific group
    #[arg(long)]
    pub group: Option<String>,
}
```

- [x] Define `CheckoutArgs` extending `ExecArgs` with `branch: String`

- [x] Implement `execute(op: Operation, entries: &[RepoEntry], states: &[RepoState], allow_partial: bool) -> Result<ExecutionReport>`:
  1. **Pre-flight phase**: run `preflight()` for each `(entry, state)` pair
  2. **Decision**: if any repo is `Blocked` and `!allow_partial`, abort with
     report showing which repos blocked and why
  3. **Dry-run display**: print what will happen per repo (proceed/skip/blocked)
  4. **Execution phase**: for each `Ready` repo, shell out to `git` CLI:
     - `git fetch` / `git pull` / `git push` / `git checkout <branch>`
     - Run in parallel using `std::thread::scope` (not rayon — network
       operations are I/O-bound and can block for seconds; rayon's fixed-size
       thread pool would starve). `thread::scope` spawns one thread per repo
       with no pool size constraint.
     - Capture stdout/stderr and exit code per repo
  5. **Report**: return structured `ExecutionReport` with per-repo results

- [x] Define `ExecutionReport`:

```rust
pub struct ExecutionReport {
    pub operation: Operation,
    pub results: Vec<RepoExecResult>,
}

pub struct RepoExecResult {
    pub entry: RepoEntry,
    pub outcome: ExecOutcome,
}

pub enum ExecOutcome {
    Success,
    Failed { stderr: String },
    Skipped { reason: String },
    Blocked { reason: String },
}
```

- [x] Implement `format_preflight_report()` for the pre-flight dry-run display
- [x] Implement `format_execution_report()` for the post-execution summary

#### 3. CLI Subcommands

**File**: `src/cli.rs`

- [x] Add `Fetch(ExecArgs)`, `Pull(ExecArgs)`, `Push(PushArgs)`,
      `Checkout(CheckoutArgs)` variants to `Command`
- [x] `PushArgs` extends `ExecArgs` with `#[arg(long)] force: bool`
- [x] `CheckoutArgs` extends `ExecArgs` with `branch: String` positional arg

#### 4. Wire Commands

**File**: `src/main.rs`

- [x] Add handlers for `Fetch`, `Pull`, `Push`, `Checkout`:
  1. Load registry (if `--group` is passed before Phase 5, return an error:
     "group filtering requires the `group` subcommand — see Phase 5")
  2. Inspect all targeted repos
  3. Run executor with appropriate `Operation`
  4. Print execution report

**File**: `src/lib.rs`

- [x] Add `pub mod operation;` and `pub mod executor;`

#### Tests for This Phase

**File**: `src/operation.rs` (unit tests)

- [x] Test `Fetch` pre-flight is always `Ready`
- [x] Test `Pull` pre-flight: `Blocked` when dirty, `Ready` when clean
- [x] Test `Push` pre-flight: `Blocked` when behind upstream, `Ready` when only ahead
- [x] Test `Push` pre-flight: `Blocked` when diverged (both ahead and behind)
- [x] Test `Push` with `force`: `Ready` even when behind upstream
- [x] Test `Checkout` pre-flight: `Blocked` when dirty, `Skipped` when branch
      missing, `Ready` when clean and branch exists

**File**: `src/executor.rs` (unit tests)

- [x] Test all-or-nothing: if any repo blocked, no execution happens
- [x] Test `--allow-partial`: blocked repos skipped, ready repos executed
- [x] Test execution report format

**File**: `tests/cli.rs` (integration tests — extend)

- [x] Test `gityard fetch` succeeds on clean repos (create temp repos with a
      local remote)
- [x] Test `gityard pull` blocked when dirty repo exists (exit code != 0, stderr
      shows which repo blocked)
- [x] Test `gityard pull --allow-partial` skips dirty repos and pulls clean ones
- [x] Test `gityard checkout main` switches branches on clean repos

### Success Criteria:

#### Automated Verification:

- [x] `cargo fmt --check` passes
- [x] `cargo clippy --all-targets` passes
- [x] `cargo test` passes — operation and executor tests green
- [x] `gityard fetch` fetches all repos
- [x] `gityard pull` is blocked when any repo is dirty (all-or-nothing)
- [x] `gityard pull --allow-partial` skips dirty repos and pulls clean ones
- [x] Pre-flight report clearly shows which repos are blocked and why

---

## Phase 5: Groups & Branches

### Overview

Add repo grouping for organizational targeting and the `branches` command for
cross-repo branch overview. This completes the v0.1.0 feature set.

### Changes Required:

#### 1. Group Management

**File**: `src/cli.rs`

- [x] Add `Group(GroupCommand)` variant to `Command`
- [x] Define `GroupCommand` subcommand enum:

```rust
#[derive(Debug, Subcommand)]
pub enum GroupCommand {
    /// Add repos to a group
    Add {
        /// Group name
        name: String,
        /// Repo aliases or paths to add
        repos: Vec<String>,
    },
    /// List all groups
    List,
    /// Remove a group (does not remove repos)
    Remove {
        /// Group name to remove
        name: String,
    },
}
```

**File**: `src/registry.rs`

- [x] Implement `Registry::add_to_group(group_name, repo_identifiers)` — adds
      group name to matching entries' `groups` field
- [x] Implement `Registry::remove_group(group_name)` — removes group name from
      all entries
- [x] Implement `Registry::list_groups() -> BTreeMap<String, Vec<&RepoEntry>>` —
      returns repos organized by group
- [x] Implement `Registry::entries_for_group(name) -> Vec<&RepoEntry>` — returns
      entries matching a group name

**File**: `src/main.rs`

- [x] Add `Group` command handlers (add, list, remove)
- [x] Update `Status`, `Fetch`, `Pull`, `Push`, `Checkout` handlers to filter
      by `--group` when present (using `Registry::entries_for_group`)

#### 2. Branches Command

**File**: `src/cli.rs`

- [x] Add `Branches(BranchesArgs)` variant to `Command`
- [x] Define `BranchesArgs`:

```rust
#[derive(Debug, Parser)]
pub struct BranchesArgs {
    /// Target a specific group
    #[arg(long)]
    pub group: Option<String>,
}
```

**File**: `src/branches.rs`

- [x] Define `BranchInfo` struct:

```rust
pub struct BranchInfo {
    pub name: String,
    pub is_head: bool,
    pub ahead_main: Option<usize>,
    pub behind_main: Option<usize>,
    pub upstream: Option<String>,
    /// Seconds since epoch from the branch tip commit's committer timestamp.
    /// Stored as i64 for comparison/staleness detection; formatted to
    /// YYYY-MM-DD for display by the presenter.
    pub last_commit_epoch: i64,
    /// Whether the branch tip is an ancestor of the main branch
    /// (i.e., `repo.graph_descendant_of(main_oid, branch_oid)` returns true).
    pub is_merged: bool,
}
```

- [x] Implement `list_branches(entry: &RepoEntry, main_branches: &[String]) -> Result<Vec<BranchInfo>>`:
  1. Open repo with `git2::Repository::open`
  2. Iterate local branches with `repo.branches(Some(BranchType::Local))`
  3. For each branch: compute ahead/behind main, check if merged, get last
     commit date, check upstream tracking
- [x] Implement `list_branches_all(entries, config) -> Vec<(RepoEntry, Vec<BranchInfo>)>`
      using `rayon::par_iter()` (same pattern as `inspect_all` — git2 read
      operations are CPU-bound, so rayon is the right fit here)

**File**: `src/presenter.rs`

- [x] Add `format_branches_table(data: &[(String, Vec<BranchInfo>)]) -> String`:
  - Group by repo
  - Show branch name, ahead/behind main, last commit date, merged status
  - Highlight current branch (HEAD)
  - Color stale branches (>30 days) differently

```
 REPO              BRANCH          ↑↓ MAIN    LAST COMMIT   MERGED
 agentspec
                   * main           ─          2026-04-12    ─
                     feat/presets   +3         2026-04-10    no
 gityard
                   * main           ─          2026-04-12    ─
```

#### 3. Wire Into Library

**File**: `src/lib.rs`

- [x] Add `pub mod branches;`

**File**: `src/main.rs`

- [x] Add `Branches` command handler: load registry, filter by group, list
      branches across repos, format and print

#### Tests for This Phase

**File**: `src/registry.rs` (unit tests — extend)

- [x] Test `add_to_group` adds group to matching entries
- [x] Test `remove_group` removes group from all entries
- [x] Test `list_groups` returns correct grouping
- [x] Test `entries_for_group` filters correctly

**File**: `src/branches.rs` (unit tests)

- [x] Test branch listing on a repo with multiple branches
- [x] Test ahead/behind main computation per branch
- [x] Test merged detection
- [x] Test `last_commit_epoch` is populated and presenter formats it as YYYY-MM-DD

**File**: `tests/cli.rs` (integration tests — extend)

- [x] Test `gityard group add rust agentspec gityard` creates group
- [x] Test `gityard group list` shows groups
- [x] Test `gityard status --group rust` filters to group members
- [x] Test `gityard branches` shows branch overview

### Success Criteria:

#### Automated Verification:

- [x] `cargo fmt --check` passes
- [x] `cargo clippy --all-targets` passes
- [x] `cargo test` passes — group and branches tests green
- [x] `gityard group add <name> <repos>` creates a group
- [x] `gityard group list` displays all groups and their members
- [x] `gityard status --group <name>` shows only repos in that group
- [x] `gityard fetch --group <name>` targets only group members
- [x] `gityard branches` shows cross-repo branch overview with correct
      ahead/behind and merge status

---

## Testing Strategy

### Cross-Phase Testing:

- [ ] End-to-end workflow test: `scan` → `status` → `fetch` → `pull` → `status`
      verifying state changes are reflected
- [ ] Test with 20+ repos to verify output remains scannable and performance is
      acceptable (rayon parallelism)
- [ ] Test with repos in various states: clean, dirty, ahead, behind, diverged,
      detached HEAD, missing remote

### Manual Testing Steps:

1. Register real workspace repos with `gityard scan ~/Workspace/jasnross`
2. Run `gityard status` and verify output matches actual repo states
3. Make changes in one repo, verify `gityard status --dirty` surfaces it
4. Run `gityard pull` with a dirty repo — verify it blocks all repos
5. Run `gityard pull --allow-partial` — verify it skips the dirty repo
6. Create a group and verify `--group` filtering works across all commands

## Performance Considerations

- **Dual parallelism model**: rayon `.par_iter()` for CPU-bound `git2` read
  operations (inspector, branches) — work-stealing thread pool matches CPU cores.
  `std::thread::scope` for I/O-bound network operations (executor: fetch, pull,
  push) — spawns one thread per repo with no pool size constraint, avoiding
  rayon thread pool starvation on slow network calls.
- **git2 vs. subprocess overhead**: git2 in-process reads (~1ms/repo) are
  significantly faster than subprocess `git status` (~5-30ms/repo). For 50 repos,
  this is the difference between ~50ms and ~250ms-1.5s.
- **Lazy config loading**: Config and registry are loaded once at startup, not
  per-repo.

## References

- Idea document: `thoughts/ideas/2026-04-12-gityard-multi-repo-manager.md`
- Research: `thoughts/research/2026-04-12-multi-repo-git-status-tools.md`
- Architecture reference: `/Users/jasonr/Workspace/jasnross/agentspec/` — Rust
  project patterns (Cargo.toml lints, clippy.toml, rustfmt.toml, deny.toml, CI,
  module layout, testing)
- [git2 crate docs](https://docs.rs/git2)
- [rayon crate docs](https://docs.rs/rayon)
- [clap derive docs](https://docs.rs/clap/latest/clap/_derive/index.html)
- [dirs crate docs](https://docs.rs/dirs)
