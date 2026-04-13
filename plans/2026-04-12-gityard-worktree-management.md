# gityard: Worktree Management — Implementation Plan

## Overview

Add worktree visibility and cleanup to gityard: a new `gityard worktrees` command that lists worktrees across all registered repos, and an enhancement to `gityard trim` that removes worktrees associated with gone branches before deleting those branches.

## Current State Analysis

gityard has no worktree awareness today. The registry accepts worktree checkout paths (validates `.git` as file or directory at `registry.rs:66-78`), but no command inspects or manages worktrees.

### Key Discoveries:

- The `git2` crate provides `Repository::worktrees()` (returns names of linked worktrees), `Repository::find_worktree(name)` (returns a `Worktree`), and `Worktree::path()` (returns the absolute working directory path). `Worktree::validate()` checks structural integrity (path exists, metadata consistent). `Worktree::prune()` removes worktree metadata — by default it only prunes *invalid* worktrees (directory missing). To prune a valid (still-existing) worktree, `WorktreePruneOptions::valid(true)` is required; adding `working_tree(true)` also deletes the directory on disk.
- `git2`'s prune API has **no dirty-checking** — the three flags (`valid`, `locked`, `working_tree`) control structural pruning only. Dirty detection requires opening a `Repository` at the worktree path and calling `repo.statuses(None)`, which is the same pattern used by `inspector::count_statuses()` (`inspector.rs:149-187`).
- The `_all` parallel pattern (`branches::list_branches_all`, `trimmer::trim_all`, `inspector::inspect_all`) is consistent: `entries.par_iter().map(|entry| (alias, result)).collect()`. The new worktree module follows this same shape.
- The `branches::TrackingState` enum and `detect_tracking_state()` function (`branches.rs:144-165`) are already `pub` — the worktree module can reuse them directly.
- Presenter functions take pre-filtered data, use `colored::control::set_override(colors)`, compute dynamic column widths, and return `String`. Tests use a `COLOR_LOCK` mutex and golden-output assertions (`presenter.rs:430-956`).
- Filter structs (`filter.rs:4-79`) have `is_empty()` and `matches(&T) -> bool` with OR semantics.

## Desired End State

After implementation:

1. `gityard worktrees` displays a table of all linked worktrees across registered repos, showing path, branch, dirty/clean status, remote tracking state, and last commit date. Repos with no linked worktrees are omitted by default.
2. `gityard worktrees --gone` filters to worktrees whose branch has a gone upstream. `--dirty` filters to worktrees with uncommitted changes. `--all` includes repos with no linked worktrees (showing the main working tree). `--group` filters by repo group.
3. `gityard trim` detects when a gone branch has a linked worktree. Clean worktrees are removed before the branch is deleted. Dirty worktrees cause both the worktree removal and branch deletion to be skipped, with a warning. `--dry-run` reports what would happen.
4. All new code passes `just check` (format + clippy + build + test + licenses).

### Verification:

- `just check` passes (covers fmt, lint, build, test, licenses)
- Manual test with a repo containing worktrees confirms correct output for `gityard worktrees`
- Manual test with a gone-branch worktree confirms `gityard trim` removes it (clean) or skips it (dirty)

## Expected Output Examples

### `gityard worktrees` (default — only repos with linked worktrees)

```
 REPO       WORKTREE                    BRANCH           STATUS    REMOTE             LAST COMMIT
 frontend   .worktrees/feat-auth        feat/auth        2M        origin/feat/auth   2026-04-10
            .worktrees/fix-nav          fix/nav          clean     gone               2026-03-28
 backend    ../backend-experiment       experiment       1M 3?     ─                  2026-04-11
            .worktrees/perf-tuning      perf/db-indexes  clean     origin/perf/db     2026-04-08

 4 worktrees across 2 repos · 2 dirty · 1 gone
```

Note: main trees in `--all` mode show full status (dirty/clean) — the examples happen to show clean main trees, but dirty main trees would display the same compact format as linked worktrees (e.g., "2M 1?").

### `gityard worktrees --gone`

```
 REPO       WORKTREE                BRANCH    STATUS    REMOTE    LAST COMMIT
 frontend   .worktrees/fix-nav      fix/nav   clean     gone      2026-03-28

 1 worktree across 1 repo · 1 gone
```

### `gityard worktrees --all` (includes repos with no linked worktrees)

```
 REPO       WORKTREE                    BRANCH           STATUS    REMOTE             LAST COMMIT
 frontend   /Users/me/repos/frontend    main             clean     origin/main        2026-04-12
            .worktrees/feat-auth        feat/auth        2M        origin/feat/auth   2026-04-10
            .worktrees/fix-nav          fix/nav          clean     gone               2026-03-28
 backend    /Users/me/repos/backend     main             clean     origin/main        2026-04-12
            ../backend-experiment       experiment       1M 3?     ─                  2026-04-11
            .worktrees/perf-tuning      perf/db-indexes  clean     origin/perf/db     2026-04-08
 docs       /Users/me/repos/docs        main             clean     origin/main        2026-04-05

 6 worktrees across 3 repos · 2 dirty · 1 gone
```

### `gityard worktrees` with a missing (prunable) worktree

```
 REPO       WORKTREE                    BRANCH           STATUS     REMOTE    LAST COMMIT
 frontend   .worktrees/feat-auth        feat/auth        2M         gone      2026-04-10
            .worktrees/old-experiment   old-experiment   missing    ─         ─

 2 worktrees across 1 repo · 1 dirty · 1 gone · 1 missing
```

The branch name for missing worktrees is recovered from `.git/worktrees/<name>/HEAD` metadata, not from the (deleted) working directory.

### `gityard worktrees` with a locked worktree

```
 REPO       WORKTREE                    BRANCH           STATUS     REMOTE             LAST COMMIT
 backend    .worktrees/long-migration   db/migration-v2  locked     origin/db/migr..   2026-04-01
            .worktrees/perf-tuning      perf/db-indexes  clean      origin/perf/db     2026-04-08

 2 worktrees across 1 repo · 1 locked
```

A locked worktree shows "locked" (yellow) in the STATUS column. Dirty state is not checked for locked worktrees — the lock itself is the relevant signal.

### `gityard trim` with a locked worktree

```
 ─ backend: protected db/migration-v2 (locked worktree at .worktrees/long-migration)
 ✓ backend: deleted feat/old-api (gone)

 1 deleted · 1 protected
```

### `gityard trim` with gone-branch worktrees

```
 ✓ frontend: deleted fix/nav (gone, removed worktree at .worktrees/fix-nav)
 ─ backend: protected hotfix/broken (dirty worktree at .worktrees/hotfix-broken)
 ✓ backend: deleted feat/old-api (gone)

 2 deleted · 1 protected
```

### `gityard trim --dry-run` with gone-branch worktrees

```
 ✓ frontend: would delete fix/nav (gone, would remove worktree at .worktrees/fix-nav)
 ─ backend: protected hotfix/broken (dirty worktree at .worktrees/hotfix-broken)
 ✓ backend: would delete feat/old-api (gone)

 2 would delete · 1 protected
```

## What We're NOT Doing

- Worktree creation (`git worktree add`) — gityard is an observer/cleanup tool, not a lifecycle manager
- Worktree-targeted bulk operations (fetch/pull/push per worktree)
- Cross-repo worktree coordination (relating worktrees across repos)
- Interactive worktree selection/removal UI
- JSON output mode (future extension, not in scope)

## Implementation Approach

Three phases, each producing a compilable, testable unit:

1. **Worktree inspection module** — data collection layer (`worktrees.rs`)
2. **`gityard worktrees` command** — CLI, filter, presenter, handler
3. **Trim worktree integration** — extend `trimmer.rs` to handle worktrees on gone branches

---

## Phase 1: Worktree Inspection Module

### Overview

Create `src/worktrees.rs` — the data collection layer for worktree information. Follows the same pattern as `branches.rs`: a struct holding per-worktree data, a per-repo function, and a parallel `_all` wrapper.

### Changes Required:

#### 1. New module: `src/worktrees.rs`

**File**: `src/worktrees.rs` (new)

- [x] Define `WorktreeKind` enum to distinguish linked worktrees from the main working tree:

```rust
/// Distinguishes linked worktrees from the main working tree.
#[derive(Clone, Debug, Eq, PartialEq)]
pub enum WorktreeKind {
    /// The repo's primary working tree (not a linked worktree).
    Main,
    /// A linked worktree created via `git worktree add`.
    Linked { name: String },
}
```

- [x] Define unified `WorktreeInfo` struct:

```rust
/// Information about a single working tree (main or linked).
#[derive(Debug)]
pub struct WorktreeInfo {
    /// Whether this is the main working tree or a linked worktree.
    pub kind: WorktreeKind,
    /// Absolute path to the working directory.
    pub path: PathBuf,
    /// Branch checked out (or detached HEAD description).
    pub branch: String,
    /// Whether the worktree path still exists on disk (always true for Main).
    pub is_valid: bool,
    /// Whether the worktree has uncommitted changes (only meaningful when `is_valid` is true).
    pub is_dirty: bool,
    pub staged_count: usize,
    pub modified_count: usize,
    pub untracked_count: usize,
    /// Remote-tracking state of the checked-out branch.
    pub tracking: TrackingState,
    /// Committer timestamp (seconds since epoch) of the branch tip.
    pub last_commit_epoch: i64,
    /// Whether the worktree is locked (`git worktree lock`). Always false for Main.
    pub is_locked: bool,
}
```

- [x] Define `RepoWorktrees` struct to hold per-repo results:

```rust
/// All worktree information for a single repository.
#[derive(Debug)]
pub struct RepoWorktrees {
    pub alias: String,
    /// All working trees for this repo. When `include_main` is true,
    /// the first entry is the main tree (kind: Main). Linked worktrees follow.
    pub worktrees: Vec<WorktreeInfo>,
}
```

- [x] Implement `list_worktrees(entry: &RepoEntry, include_main: bool) -> Result<RepoWorktrees, String>`:
  - Open repo via `git2::Repository::open(&entry.path)`
  - Call `repo.worktrees()` to get linked worktree names
  - For each name: `repo.find_worktree(name)`, then `worktree.validate()` to check validity
  - For valid worktrees: open a `Repository` at `worktree.path()` for HEAD and dirty-checking (this worktree-opened repo correctly returns the worktree's own HEAD and status). Call `inspector::read_branch(&worktree_repo)` for the branch name (returns `(String, Option<Oid>)` — only the name string is needed). Call `inspector::count_statuses(&worktree_repo)` for dirty detection. For tracking state, call `branches::detect_tracking_state` on the **parent** repo (the one opened at `entry.path`), not the worktree-opened repo — branch upstream config lives in the parent's `.git/config`, so looking it up from the worktree repo may return `Untracked` incorrectly. Obtain the `Branch` reference via `parent_repo.find_branch(&branch_name, BranchType::Local)` before passing it to `detect_tracking_state`. Read tip commit timestamp from the worktree repo's HEAD commit. For `is_locked`: `worktree.is_locked()` returns `Result<WorktreeLockStatus, Error>`, not `bool` — match on `WorktreeLockStatus::Locked(_)` to derive the boolean.
  - For invalid worktrees: `repo.find_worktree(name)` still succeeds (metadata in `.git/worktrees/<name>` is intact), so the branch name can be recovered from the worktree's HEAD ref in the metadata. Read the branch name from the metadata if possible; fall back to `"(unknown)"` only if the ref cannot be parsed. Populate `is_valid: false`, `is_locked: false`, zero out status fields.
  - If `include_main`: prepend a `WorktreeInfo` with `kind: WorktreeKind::Main`, `is_valid: true`, `is_locked: false`, populated with the parent repo's HEAD, statuses, and tracking state
  - Note: `WorktreeInfo.path` stores the absolute path (as returned by `Worktree::path()`). Display path computation (relative to repo when inside/adjacent, absolute otherwise) is the presenter's responsibility, consistent with the inspector/presenter separation principle. Use `pathdiff::diff_paths` (add `pathdiff` crate as a dependency) or manual prefix-stripping for the relative path computation in the presenter. The entry's path is passed to the presenter alongside the worktree data for this purpose.

- [x] Implement `list_worktrees_all(entries: &[&RepoEntry], include_main: bool) -> Vec<(String, PathBuf, Result<RepoWorktrees, String>)>`:
  - `entries.par_iter().map(...)` — same shape as `branches::list_branches_all`
  - Note: unlike `list_branches_all` and `trim_all`, no `main_branches` parameter — the worktree listing has no need to find or protect main branches

- [x] Extract `count_statuses` from `inspector.rs` into a shared helper or make it `pub(crate)`. Currently it's a private function at `inspector.rs:149-187`. The worktrees module needs the same logic. Options:
  - Make `count_statuses` `pub(crate)` in `inspector.rs` (simplest, follows "colocate with consumer" only when there's one consumer — with two consumers it's fine to share)
  - The worktree module also needs `read_branch` logic (`inspector.rs:117-146`) — make that `pub(crate)` too

#### 2. Add `pathdiff` dependency

**File**: `Cargo.toml`

- [x] Add `pathdiff = "0.2"` to `[dependencies]` — used by the presenter to compute relative display paths for worktrees. Verify license compatibility with `just licenses` (`cargo deny check licenses`).

#### 3. Register module in `src/lib.rs`

**File**: `src/lib.rs`

- [x] Add `pub mod worktrees;` to module list

#### 4. Visibility changes in `src/inspector.rs`

**File**: `src/inspector.rs`

- [x] Change `fn count_statuses` (line 149) from private to `pub(crate) fn count_statuses`
- [x] Change `fn read_branch` (line 117) from private to `pub(crate) fn read_branch`

#### Tests for This Phase

**File**: `src/worktrees.rs` (inline `#[cfg(test)]` module)

- [x] `test_list_worktrees_no_worktrees` — repo with no linked worktrees returns empty `worktrees` vec
- [x] `test_list_worktrees_with_linked_worktree` — create a repo, add a worktree via `git2`, verify `WorktreeInfo` fields are populated correctly (name, branch, is_valid, path)
- [x] `test_list_worktrees_invalid_worktree` — create a worktree, delete its directory, verify `is_valid: false` and status fields zeroed
- [x] `test_list_worktrees_dirty_worktree` — create a worktree, modify a file in it, verify `is_dirty: true` and correct counts
- [x] `test_list_worktrees_include_main` — verify a `WorktreeKind::Main` entry is present when `include_main: true` and absent when `false`
- [x] `test_list_worktrees_locked_worktree` — lock a worktree via git2, verify `is_locked: true` in the result
- [x] `test_list_worktrees_all_parallel` — two repos, verify `list_worktrees_all` returns results for both

Test helper: create a helper `init_repo_with_worktree(tmp: &Path) -> (PathBuf, String)` that sets up a repo with a linked worktree (commit, branch, `git2::Repository::worktree()`). Follow the pattern of `trimmer::tests::init_repo_with_gone_branch`.

### Success Criteria:

#### Automated Verification:

- [x] `just check` passes (fmt, lint, build, test, licenses)
- [x] All new unit tests in `worktrees.rs` pass
- [x] Existing tests in `inspector.rs` still pass after visibility changes

---

## Phase 2: `gityard worktrees` Command

### Overview

Wire up the CLI command, filter, presenter table, and handler function so `gityard worktrees` produces output.

### Changes Required:

#### 1. CLI definition

**File**: `src/cli.rs`

- [x] Add `WorktreesArgs` struct:

```rust
/// Arguments for the `worktrees` subcommand.
#[derive(Debug, Parser)]
pub struct WorktreesArgs {
    /// Show only worktrees whose branch upstream has been deleted
    #[arg(long)]
    pub gone: bool,
    /// Show only worktrees with uncommitted changes
    #[arg(long)]
    pub dirty: bool,
    /// Include repos with no linked worktrees (shows main working tree)
    #[arg(long)]
    pub all: bool,
    /// Target a specific group
    #[arg(long)]
    pub group: Option<String>,
}
```

- [x] Add `Worktrees(WorktreesArgs)` variant to the `Command` enum, with doc comment: `/// List git worktrees across repositories`

#### 2. Filter struct

**File**: `src/filter.rs`

- [x] Add `WorktreeFilter` struct following the existing pattern:

```rust
pub struct WorktreeFilter {
    pub gone: bool,
    pub dirty: bool,
}

impl WorktreeFilter {
    pub fn is_empty(&self) -> bool {
        !self.gone && !self.dirty
    }

    pub fn matches(&self, worktree: &WorktreeInfo) -> bool {
        if self.is_empty() {
            return true;
        }
        if self.gone && matches!(worktree.tracking, TrackingState::Gone { .. }) {
            return true;
        }
        if self.dirty && worktree.is_dirty {
            return true;
        }
        false
    }
}
```

Note: `WorktreeFilter::matches` operates on any `WorktreeInfo`, but filtering is only applied to linked worktrees (`kind: Linked`) in the handler. The main tree (`kind: Main`) is always included for repos that have at least one matching linked worktree — it provides context. If a filter is active (e.g., `--gone`) and a repo has no matching linked worktrees, the entire repo (including its main tree) is omitted.

#### 3. Presenter function

**File**: `src/presenter.rs`

- [x] Add `format_worktrees_table` function. Input type: `&[(String, PathBuf, Vec<&WorktreeInfo>)]` — alias, repo path (for relative display path computation), and pre-ordered worktree refs (main tree first if present, then linked). Follow the `format_branches_table` pattern:
  - Dynamic column widths for REPO, WORKTREE (path), BRANCH, STATUS, REMOTE, LAST COMMIT
  - First worktree of a repo shows alias, subsequent rows show `""`
  - STATUS column: "clean", "missing" (red, for `is_valid: false`), "locked" (yellow, for `is_locked: true`), or compact dirty format ("3M 1S 2?" — same as `format_status_table`). If both locked and dirty, show "locked 2M" or similar.
  - REMOTE column: upstream name, "gone" (red), or "─" — same as `format_branches_table`
  - LAST COMMIT: YYYY-MM-DD, yellow if >30 days stale — same as `format_branches_table`
  - WORKTREE column: compute display path from the absolute `WorktreeInfo.path` relative to the repo path using `pathdiff::diff_paths`. If `diff_paths` returns `None` (can happen on Windows with different drive roots), fall back to the absolute path. Show absolute also when the relative path is not inside or adjacent to the repo (i.e., the relative path doesn't start with `.` or `..`).
  - Summary footer: "N worktrees across N repos" with optional " · N dirty · N gone · N locked · N missing" counts. The worktree count includes main trees when `--all` is active (e.g., 3 main + 3 linked = "6 worktrees across 3 repos"). Without `--all`, only linked worktrees are counted.

#### 4. Handler function

**File**: `src/main.rs`

- [x] Add `Command::Worktrees(args) => cmd_worktrees(&config_dir, &args)` match arm
  > **Deviation:** `cmd_worktrees` does not take `&Config` — worktree listing has no need for
  > `main_branches` or other config. Simpler signature than `cmd_branches`.
- [x] Implement `cmd_worktrees` following the `cmd_branches` pattern:
  - `Registry::load` + `resolve_entries` (group filtering)
  - Call `worktrees::list_worktrees_all(&entries, args.all)`
  - Construct `WorktreeFilter` from args
  - For each repo result: apply filter to linked worktrees only (skip `kind: Main` entries during filtering). If no linked worktrees match the filter, omit the entire repo. If matches exist and `--all` was specified, include the main tree entry as the first item.
  - Collect into presenter-ready data: `Vec<(alias, repo_path, Vec<&WorktreeInfo>)>`
  - Call `presenter::format_worktrees_table` and print

#### Tests for This Phase

**File**: `src/filter.rs` (inline tests)

- [x] `test_worktree_filter_empty_matches_all` — no flags set matches any worktree
- [x] `test_worktree_filter_gone` — `gone: true` matches only gone-tracking worktrees
- [x] `test_worktree_filter_dirty` — `dirty: true` matches only dirty worktrees
- [x] `test_worktree_filter_combined_or` — `gone: true, dirty: true` matches either

**File**: `src/presenter.rs` (inline tests)

- [x] `test_worktrees_table_basic` — single repo with one worktree, verify output contains repo alias, worktree path, branch
- [x] `test_worktrees_table_missing_worktree` — worktree with `is_valid: false`, verify "missing" appears in STATUS
- [x] `test_worktrees_table_with_main_tree` — `--all` mode, verify `WorktreeKind::Main` entry renders as first row with repo path
- [x] `test_worktrees_table_locked_worktree` — worktree with `is_locked: true`, verify "locked" appears in STATUS
- [x] `test_worktrees_table_full_output` — golden-output test with multiple repos and worktree states (clean, dirty, gone, locked, missing), following the pattern at `presenter.rs:793-933`

**File**: `tests/cli.rs` (integration test)

- [x] `test_worktrees_command_no_repos` — verify `gityard worktrees` handles empty registry gracefully
- [x] `test_worktrees_command_help` — verify `--help` output includes the command description

### Success Criteria:

#### Automated Verification:

- [x] `just check` passes
- [x] All new filter tests pass
- [x] All new presenter tests pass (including golden-output)
- [x] Integration tests pass

#### Manual Verification:

- [ ] Run `gityard worktrees` against a real repo with worktrees and verify the table looks correct (manual-only: visual layout judgment)
- [ ] Run `gityard worktrees --gone` and `--dirty` and verify filtering works as expected

---

## Phase 3: Trim Worktree Integration

### Overview

Extend `gityard trim` to detect and handle worktrees on gone branches. When a gone branch has a linked worktree, remove the worktree (if clean) before deleting the branch. Skip both if the worktree is dirty.

### Changes Required:

#### 1. New outcome variants

**File**: `src/trimmer.rs`

- [x] Add new `TrimOutcome` variants for worktree-related results:

```rust
pub enum TrimOutcome {
    // ... existing variants ...
    /// Branch was deleted after its worktree was removed.
    DeletedAfterWorktreeRemoval { worktree_path: PathBuf },
    /// Branch would be deleted after worktree removal (dry-run).
    WouldDeleteAfterWorktreeRemoval { worktree_path: PathBuf },
    /// Branch was protected because its worktree has uncommitted changes.
    ProtectedDirtyWorktree { worktree_path: PathBuf },
    /// Branch was protected because its worktree is locked.
    ProtectedLockedWorktree { worktree_path: PathBuf },
}
```

#### 2. New `TrimAction` variant and worktree resolution step

**File**: `src/trimmer.rs`

- [x] Add new `TrimAction` variants and a worktree state struct:

```rust
/// Pre-assessed state of a worktree linked to a trim candidate.
struct WorktreeState {
    name: String,
    path: PathBuf,
    is_dirty: bool,
    is_locked: bool,
}

enum TrimAction {
    Delete,
    /// Branch should be deleted after removing its clean, unlocked worktree.
    DeleteWithWorktree { worktree_name: String, worktree_path: PathBuf },
    Protect { reason: String },
    /// Branch is protected because its worktree is dirty.
    ProtectDirtyWorktree { worktree_path: PathBuf },
    /// Branch is protected because its worktree is locked.
    ProtectLockedWorktree { worktree_path: PathBuf },
}
```

Using structured `TrimAction` variants (not overloaded `Protect { reason: String }`) gives the action loop a direct 1:1 mapping to `TrimOutcome` variants without string matching:
- `Delete` → `Deleted` / `WouldDelete` (unchanged)
- `DeleteWithWorktree` → `DeletedAfterWorktreeRemoval` / `WouldDeleteAfterWorktreeRemoval`
- `Protect` → `Protected` (unchanged — HEAD, main branch)
- `ProtectDirtyWorktree` → `ProtectedDirtyWorktree`
- `ProtectLockedWorktree` → `ProtectedLockedWorktree`

- [x] Add a helper function `build_worktree_branch_map(repo: &Repository) -> HashMap<String, WorktreeState>` that:
  - Calls `repo.worktrees()` to list linked worktree names
  - For each name: `repo.find_worktree(name)`, validate the worktree, check `worktree.is_locked()`, then open a `Repository` at `worktree.path()`, read HEAD branch name, check dirty state via `inspector::count_statuses`
  - If `Repository::open` fails (e.g., worktree path was deleted but `.git/worktrees/<name>` metadata still exists), skip the entry — stale worktree metadata does not block branch deletion
  - Detached HEAD worktrees are skipped (they cannot be on a "gone branch")
  - Returns a `HashMap<branch_name, WorktreeState>` with all assessed state

- [x] Add a `resolve_worktrees` function — a post-assessment step mirroring the `resolve_head` pattern:

```rust
fn resolve_worktrees(
    repo: &Repository,
    candidates: &mut [TrimCandidate],
) {
```

  - Called after `resolve_head`, before the action loop (same position in the pipeline)
  - Calls `build_worktree_branch_map(repo)` once
  - For each candidate with `TrimAction::Delete`:
    - Look up `candidate.name` in the map
    - If found and **locked**: change action to `TrimAction::ProtectLockedWorktree { worktree_path }`
    - If found and **dirty**: change action to `TrimAction::ProtectDirtyWorktree { worktree_path }`
    - If found and **clean**: change action to `TrimAction::DeleteWithWorktree { worktree_name, worktree_path }`
    - If not found: leave action as `TrimAction::Delete` (no worktree)

- [x] Add a helper function `remove_worktree(repo: &Repository, worktree_name: &str) -> Result<(), String>` that:
  - Calls `repo.find_worktree(worktree_name)`
  - Prunes with a mutable options binding (builder methods return `&mut Self`, so chaining into `prune()` doesn't compile):
    ```rust
    let mut opts = WorktreePruneOptions::new();
    opts.valid(true);        // allow pruning a valid (still-existing) worktree
    opts.working_tree(true); // also delete the working directory on disk
    worktree.prune(Some(&mut opts))
    ```
  - Maps errors to `String`

- [x] Update the action loop to handle the new `TrimAction` variants. Each variant maps directly to a `TrimOutcome` — no string matching needed:
  - The action loop remains purely about execution — all assessment happened in `resolve_worktrees`
  - `DeleteWithWorktree { worktree_name, worktree_path }`: if `dry_run`, produce `WouldDeleteAfterWorktreeRemoval { worktree_path }`; otherwise call `remove_worktree(&repo, &worktree_name)`, then `delete_branch`, produce `DeletedAfterWorktreeRemoval { worktree_path }`
  - `ProtectDirtyWorktree { worktree_path }`: produce `ProtectedDirtyWorktree { worktree_path }` (increment `protected` counter)
  - `ProtectLockedWorktree { worktree_path }`: produce `ProtectedLockedWorktree { worktree_path }` (increment `protected` counter)
  - `Delete`: unchanged existing logic
  - `Protect { reason }`: unchanged existing logic — only covers original reasons (HEAD, main branch)
  - Note: `wildcard_enum_match_arm` is denied in this project, so all five `TrimAction` variants require explicit match arms (no `_` catch-all). This is the desired behavior — adding a variant forces all match sites to be updated.

#### 3. Presenter updates for new outcomes

**File**: `src/presenter.rs`

- [x] Extend `format_trim_report` to handle the four new `TrimOutcome` variants. Worktree paths in trim output use the same `pathdiff` display-path computation as the worktrees table (relative to repo path when possible, absolute otherwise). The repo path is available from the `RepoTrimResult` (may need to add a `repo_path: PathBuf` field, or pass it alongside the report):
  - `DeletedAfterWorktreeRemoval { worktree_path }`: green `✓`, message like `"deleted {branch} (gone, removed worktree at {path})"`
  - `WouldDeleteAfterWorktreeRemoval { worktree_path }`: cyan `✓`, message like `"would delete {branch} (gone, would remove worktree at {path})"`
  - `ProtectedDirtyWorktree { worktree_path }`: yellow `─`, message like `"protected {branch} (dirty worktree at {path})"`
  - `ProtectedLockedWorktree { worktree_path }`: yellow `─`, message like `"protected {branch} (locked worktree at {path})"`
  - Update summary counters accordingly (deleted/would_delete/protected)

#### Tests for This Phase

**File**: `src/trimmer.rs` (inline tests)

- [x] `test_trim_gone_branch_with_clean_worktree` — create a repo with a gone branch that has a linked worktree (clean). Verify trim deletes the worktree and the branch, outcome is `DeletedAfterWorktreeRemoval`.
- [x] `test_trim_gone_branch_with_dirty_worktree` — same setup but modify a file in the worktree. Verify trim skips both, outcome is `ProtectedDirtyWorktree`, worktree and branch still exist.
- [x] `test_trim_gone_branch_with_worktree_dry_run` — clean worktree, `dry_run: true`. Verify outcome is `WouldDeleteAfterWorktreeRemoval`, worktree and branch still exist.
- [x] `test_trim_gone_branch_no_worktree_unchanged` — verify existing behavior is unchanged for gone branches without worktrees (regression test).
- [x] `test_trim_gone_branch_with_locked_worktree` — lock the worktree via git2, verify trim skips both worktree and branch, outcome is `ProtectedLockedWorktree`.
- [x] `test_trim_gone_branch_with_invalid_worktree` — worktree directory deleted manually. The `build_worktree_branch_map` helper skips worktrees where `Repository::open` fails (directory missing), so the branch is not in the map. Verify trim deletes the branch normally with a standard `Deleted` outcome (not `DeletedAfterWorktreeRemoval`). Stale worktree metadata is left for `git worktree prune` — trimmer does not call `prune()` on worktrees it couldn't open.

Test helper: extend `init_repo_with_gone_branch` or create a new `init_repo_with_gone_branch_and_worktree(tmp: &Path) -> (PathBuf, String, String, PathBuf)` that also creates a linked worktree checked out on the gone branch.

**File**: `src/presenter.rs` (inline tests)

- [x] `test_trim_report_worktree_outcomes` — golden-output test covering all four new outcome variants (deleted after worktree removal, would delete after worktree removal, protected dirty worktree, protected locked worktree) alongside existing ones

### Success Criteria:

#### Automated Verification:

- [x] `just check` passes
- [x] All new trimmer tests pass
- [x] All existing trimmer tests still pass (regression)
- [x] New presenter golden-output test passes

#### Manual Verification:

- [ ] Create a real repo with a gone-branch worktree (clean), run `gityard trim`, verify worktree is removed and branch is deleted
- [ ] Same with a dirty worktree, verify both are skipped with a warning
- [ ] Run `gityard trim --dry-run`, verify it reports what would happen without acting

---

## Performance Considerations

- Worktree inspection opens a `Repository` at each worktree path (for dirty-checking and HEAD reading). For repos with many worktrees, this is multiple `Repository::open` calls per repo. This is the same pattern as `inspector::count_stashes` (which re-opens for `&mut Repository`), so the cost is expected and acceptable.
- The `repo.statuses(None)` call is the most expensive operation per worktree (walks the working tree). For large worktrees this could be slow, but it's inherent to dirty-checking and unavoidable.
- The `build_worktree_branch_map` helper in trim opens a `Repository` at each worktree path once (for HEAD reading and dirty-checking), building a HashMap for O(1) per-candidate lookup. Total cost is O(worktrees + candidates) per repo. The dirty check and lock check happen during the `resolve_worktrees` assessment step, keeping the action loop allocation-free.

## References

- Idea document: `thoughts/ideas/2026-04-12-gityard-worktree-management.md`
- git2 `Worktree` API: https://docs.rs/git2/latest/git2/struct.Worktree.html
- git2 `WorktreePruneOptions`: https://docs.rs/git2/latest/git2/struct.WorktreePruneOptions.html
- Existing parallel inspection pattern: `src/branches.rs:121-133`
- Existing trimmer: `src/trimmer.rs`
- Existing filter pattern: `src/filter.rs:53-79`
- Existing presenter pattern: `src/presenter.rs:167-268`
