# gityard trim: Implementation Plan

## Overview

Implement `gityard trim` for bulk branch cleanup and enhance `gityard branches`
with remote-tracking visibility. The work builds on existing branch inspection
(`branches.rs`), the parallel execution pattern (`executor.rs`), and the
OR-semantics filter pattern (`filter.rs`).

## Current State Analysis

### Branch inspection (`src/branches.rs:78-82`)

The upstream detection collapses all failure modes into `None`:

```rust
let upstream = branch
    .upstream()
    .ok()
    .and_then(|u| u.name().ok().flatten().map(String::from));
```

There is no distinction between "no upstream configured" and "upstream configured
but the remote-tracking ref was pruned." Both produce `upstream: None`.

### Branches table (`src/presenter.rs:165-241`)

The current columns are: REPO, BRANCH, ↑↓ MAIN, LAST COMMIT, MERGED. The
`upstream` field from `BranchInfo` is not displayed anywhere — it's collected
but unused.

### Key Discoveries:

- `BranchInfo.upstream` is not referenced by the presenter, so changing it to a
  richer type only requires updating `branches.rs` and test fixtures that
  construct `BranchInfo` directly (`presenter.rs:422-429`)
- The git2 crate provides `repo.branch_upstream_name(refname)` which reads
  branch config without resolving the ref, enabling clean "gone" detection by
  separating config lookup from ref resolution
- `Branch::delete()` fails with a generic error if the branch is HEAD; checking
  `branch.is_head()` beforehand provides a better error message
- The `StatusFilter` pattern (`filter.rs:8-45`) — OR semantics with an
  `is_empty()` gate — is the model for the branch filter

## Desired End State

After implementation:

1. `gityard branches` shows a REMOTE column: upstream name for tracked branches,
   `gone` (colored) for branches with pruned upstreams, `─` for untracked branches
2. `gityard branches --gone` and `--no-remote` filter branches by tracking state
3. `gityard trim` deletes gone branches across registered repos
4. `gityard trim --no-remote` also deletes branches with no upstream
5. `gityard trim --dry-run` previews what would be deleted
6. HEAD and main branches are never deleted
7. Exit code is 1 if any branch deletion fails

### Verification:

- `just check` passes (format + lint + build + test + license check)
- All new unit tests pass for tracking state detection, branch filtering,
  trimmer safety logic, and output formatting
- All new integration tests pass for `trim`, `trim --dry-run`, `trim --no-remote`,
  and `branches` filter flags

## What We're NOT Doing

- No auto-fetch before trim — users run `gityard fetch` first
- No interactive confirmation — users preview with `--dry-run`
- No "merged into main" as a trim criterion
- No remote branch deletion — local branches only
- No changes to the `status` command or `inspector.rs`

## Implementation Approach

Three phases, each building on the previous and independently testable:

1. **Data model**: Introduce `TrackingState` enum, update `list_branches()` with
   two-step detection, update presenter to display the new REMOTE column
2. **Branch filtering**: Add `--gone`/`--no-remote` flags to `branches` command
   with a `BranchFilter` following the `StatusFilter` pattern
3. **Trim command**: New CLI command, `trimmer.rs` module, parallel deletion with
   safety checks, output formatting, exit code

---

## Phase 1: Branch Tracking State and REMOTE Column

### Overview

Replace `BranchInfo.upstream: Option<String>` with a `TrackingState` enum that
distinguishes three states: tracking (healthy upstream), gone (upstream config
exists but ref is pruned), and untracked (no upstream config). Update the
presenter to display this as a new REMOTE column in the branches table.

### Changes Required:

#### 1. Add `TrackingState` enum and update `BranchInfo`

**File**: `src/branches.rs`

- [x] Add `TrackingState` enum before `BranchInfo` (near line 7):

```rust
/// Remote-tracking state of a local branch.
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum TrackingState {
    /// Branch tracks a live remote-tracking ref (e.g., "origin/main").
    Tracking { name: String },
    /// Upstream config exists but the remote-tracking ref was pruned.
    /// `name` is the configured upstream ref that no longer resolves.
    Gone { name: String },
    /// No upstream tracking configured.
    Untracked,
}
```

- [x] Replace the `upstream` field in `BranchInfo` (line 18-19):

```rust
// Before:
pub upstream: Option<String>,

// After:
/// Remote-tracking state: live upstream, gone (pruned), or untracked.
pub tracking: TrackingState,
```

#### 2. Update `list_branches()` with two-step detection

**File**: `src/branches.rs`

- [x] Extract a shared `detect_tracking_state()` helper function (public, since
  `trimmer.rs` also calls it in phase 3):
  > **Deviation:** Used `let...else` syntax instead of `match` for early returns,
  > as required by clippy's `manual_let_else` pedantic lint.

```rust
/// Determine the remote-tracking state of a local branch.
///
/// Non-UTF-8 ref names (where `Reference::name()` returns `None`) fall through
/// to `Untracked` since `branch_upstream_name("")` returns `Err`. This is an
/// accepted limitation — non-UTF-8 branch names are extremely rare in practice.
pub fn detect_tracking_state(repo: &git2::Repository, branch: &git2::Branch) -> TrackingState {
    let refname = match branch.get().name() {
        Some(name) => name,
        None => return TrackingState::Untracked,
    };
    match repo.branch_upstream_name(refname) {
        Ok(buf) => {
            let upstream_ref = match buf.as_str() {
                Some(s) => s,
                None => return TrackingState::Untracked,
            };
            let short_name = upstream_ref
                .strip_prefix("refs/remotes/")
                .unwrap_or(upstream_ref)
                .to_string();
            if repo.find_reference(upstream_ref).is_ok() {
                TrackingState::Tracking { name: short_name }
            } else {
                TrackingState::Gone { name: short_name }
            }
        }
        Err(_) => TrackingState::Untracked,
    }
}
```

- [x] Replace the upstream detection block (lines 78-82) to call the new helper:

```rust
let tracking = detect_tracking_state(&repo, &branch);
```

- [x] Update the `BranchInfo` construction (line 97-105) to use `tracking`
  instead of `upstream`

#### 3. Update unit tests in `branches.rs`

**File**: `src/branches.rs`

- [x] Update `test_list_branches_single_branch` to assert on `tracking` field
  (it should be `TrackingState::Untracked` since the test repo has no remote)
  > **Deviation:** Instead of modifying existing tests (which don't assert on the
  > upstream/tracking field), added a dedicated `test_tracking_state_untracked` test.
- [x] Update `test_list_branches_multiple` similarly
  > **Deviation:** Same as above — covered by `test_tracking_state_untracked`.

#### 4. Update presenter to display REMOTE column

**File**: `src/presenter.rs`

- [x] Add a `use crate::branches::TrackingState;` import
- [x] In `format_branches_table()`, compute `max_remote` column width by
  iterating branch tracking states (similar to `max_alias` / `max_branch`
  computation at lines 176-188). Minimum width should be 6 ("REMOTE" header)
- [x] Add "REMOTE" to the header line (line 194-197)
- [x] For each branch row (lines 206-236), format the tracking state:
  - `TrackingState::Tracking { name }` → the name string
  - `TrackingState::Gone { name }` → `"gone"` colored red (not yellow — red
    matches the "attention" semantics used elsewhere in the codebase)
  - `TrackingState::Untracked` → `"─"`
- [x] Append the formatted remote value to each row

#### 5. Update presenter test fixtures

**File**: `src/presenter.rs`

- [x] Update `make_branch` helper (lines 418-430) to use `tracking:
  TrackingState::Untracked` instead of `upstream: None`
- [x] Add a test for the REMOTE column display with all three tracking states

#### Tests for This Phase

**Unit tests in `src/branches.rs`:**

- [x] Test that a branch with no remote produces `TrackingState::Untracked`
  (covered by `test_tracking_state_untracked`)
- [x] Test "gone" detection: create a repo with a remote, set up tracking, then
  delete the remote-tracking ref and verify `TrackingState::Gone`
  (covered by `test_tracking_state_gone`)
- [x] Test healthy tracking: create a repo with remote and tracking branch,
  verify `TrackingState::Tracking` with correct name
  (covered by `test_tracking_state_tracking`)

**Unit tests in `src/presenter.rs`:**

- [x] Test REMOTE column shows upstream name for `Tracking` state
- [x] Test REMOTE column shows `"gone"` for `Gone` state
- [x] Test REMOTE column shows `"─"` for `Untracked` state
  (all three covered by `test_branches_table_remote_column_all_states`)
- [x] Test column alignment with varying remote name lengths
  (covered by updated `test_branches_table_dynamic_alignment` which now checks
  REMOTE column alignment)

**Integration test in `tests/cli.rs`:**

- [x] Update `test_branches_shows_overview` to also check for "REMOTE" header

### Success Criteria:

#### Automated Verification:

- [x] `just check` passes
  > **Deviation:** `just check` requires committed code for `cargo fix`. Verified
  > via `cargo clippy --all-targets` + `cargo +nightly fmt` + `cargo test` instead.
- [x] All existing tests still pass (no regressions from the field rename)
- [x] New unit tests for tracking state detection pass
- [x] New presenter tests for REMOTE column pass

---

## Phase 2: Branch Filter Flags

### Overview

Add `--gone` and `--no-remote` filter flags to `gityard branches`, following
the `StatusFilter` OR-semantics pattern. When a flag is active, only branches
matching that state are shown.

### Changes Required:

#### 1. Add filter flags to `BranchesArgs`

**File**: `src/cli.rs`

- [x] Add two boolean flags to `BranchesArgs` (lines 132-138):

```rust
#[derive(Debug, Parser)]
pub struct BranchesArgs {
    /// Show only branches whose upstream remote has been deleted
    #[arg(long)]
    pub gone: bool,
    /// Show only branches with no upstream tracking branch
    #[arg(long)]
    pub no_remote: bool,
    /// Target a specific group
    #[arg(long)]
    pub group: Option<String>,
}
```

#### 2. Add `BranchFilter` struct

**File**: `src/filter.rs`

- [x] Add a `use crate::branches::{BranchInfo, TrackingState};` import
- [x] Add `BranchFilter` struct after `StatusFilter`, following the same pattern:

```rust
/// Tracking-state filter for branch selection.
///
/// When no flags are set, all branches match. When one or more flags are set,
/// a branch matches if it satisfies at least one active flag (OR semantics).
pub struct BranchFilter {
    pub gone: bool,
    pub no_remote: bool,
}

impl BranchFilter {
    pub fn is_empty(&self) -> bool {
        !self.gone && !self.no_remote
    }

    pub fn matches(&self, branch: &BranchInfo) -> bool {
        if self.is_empty() {
            return true;
        }

        if self.gone && matches!(branch.tracking, TrackingState::Gone { .. }) {
            return true;
        }
        if self.no_remote && matches!(branch.tracking, TrackingState::Untracked) {
            return true;
        }

        false
    }
}
```

#### 3. Apply filter in `cmd_branches()`

**File**: `src/main.rs`

- [x] Add `use gityard::filter::BranchFilter;` import (near line 12)
- [x] In `cmd_branches()` (lines 313-342), construct the filter from args and
  apply it when collecting branch refs:

```rust
let filter = BranchFilter {
    gone: args.gone,
    no_remote: args.no_remote,
};

// In the Ok(branch_list) arm, filter branches and skip repos with no matches:
let refs: Vec<&_> = branch_list
    .iter()
    .filter(|b| filter.matches(b))
    .collect();
if !refs.is_empty() {
    table_data.push((alias.clone(), refs));
}
```

- [x] Track whether any errors occurred (add a `has_errors` bool) and update the
  empty-after-filter message to account for it:

```rust
if table_data.is_empty() {
    if has_errors {
        // Errors were already printed; don't add a misleading "no data" message
    } else if filter.is_empty() {
        eprintln!("no branch data available");
    } else {
        eprintln!("no branches match the given filters");
    }
    return Ok(());
}
```

#### Tests for This Phase

**Unit tests in `src/filter.rs`:**

- [x] Test `BranchFilter` with no flags matches all tracking states
- [x] Test `--gone` matches only `TrackingState::Gone`, not `Tracking` or
  `Untracked`
- [x] Test `--no-remote` matches only `TrackingState::Untracked`
- [x] Test both flags with OR semantics: matches `Gone` and `Untracked` but
  not `Tracking`

**Integration test in `tests/cli.rs`:**

- [x] Test `branches --gone` on a repo with a gone upstream shows the branch
- [x] Test `branches --no-remote` on a repo with an untracked branch shows it
- [x] Test `branches --gone` on a repo with no gone branches shows the
  "no branches match" message

### Success Criteria:

#### Automated Verification:

- [x] `just check` passes
  > **Deviation:** Same as Phase 1 — verified via clippy + fmt + test individually.
- [x] New `BranchFilter` unit tests pass
- [x] New integration tests for `--gone` and `--no-remote` pass

---

## Phase 3: Trim Command

### Overview

Implement `gityard trim` — a new command that deletes local branches based on
their tracking state. Default: delete "gone" branches. With `--no-remote`: also
delete untracked branches. With `--dry-run`: show what would be deleted without
deleting. HEAD and main branches are always protected. Exit code is 1 if any
deletion fails.

### Changes Required:

#### 1. Add `Trim` command to CLI

**File**: `src/cli.rs`

- [x] Add `TrimArgs` struct:

```rust
/// Arguments for the `trim` subcommand.
#[derive(Debug, Parser)]
pub struct TrimArgs {
    /// Also delete branches with no upstream tracking branch
    #[arg(long)]
    pub no_remote: bool,
    /// Show what would be deleted without deleting
    #[arg(long)]
    pub dry_run: bool,
    /// Target a specific group
    #[arg(long)]
    pub group: Option<String>,
}
```

- [x] Add `Trim(TrimArgs)` variant to `Command` enum (after `Branches`, before
  `Completions`), with doc comment:

```rust
/// Delete local branches whose upstream remote has been deleted
Trim(TrimArgs),
```

#### 2. Create trimmer module

**File**: `src/trimmer.rs` (new file)

- [x] Define `TrimOutcome` enum:

```rust
/// Outcome for a single branch trim operation.
#[derive(Debug)]
pub enum TrimOutcome {
    /// Branch was deleted.
    Deleted,
    /// Branch would be deleted (dry-run mode).
    WouldDelete,
    /// Branch was protected (HEAD or main) and skipped.
    Protected { reason: String },
    /// Branch deletion failed.
    Failed { reason: String },
}
```

- [x] Define `BranchTrimResult` struct for per-branch results:

```rust
/// Result of trimming a single branch.
#[derive(Debug)]
pub struct BranchTrimResult {
    pub name: String,
    pub tracking: TrackingState,
    pub outcome: TrimOutcome,
}
```

- [x] Define `RepoTrimResult` struct:

```rust
/// Result of trimming branches in a single repository.
#[derive(Debug)]
pub struct RepoTrimResult {
    pub alias: String,
    pub branches: Vec<BranchTrimResult>,
}
```

- [x] Implement `trim_repo()` function — the core single-repo logic:

```rust
pub fn trim_repo(
    entry: &RepoEntry,
    main_branches: &[String],
    include_no_remote: bool,
    dry_run: bool,
) -> Result<RepoTrimResult, String>
```

  Logic:
  1. Open the repo with `git2::Repository::open`
  2. Iterate local branches with `repo.branches(Some(BranchType::Local))`,
     binding each as `let (mut branch, _) = ...` (`mut` needed for `delete()`)
  3. For each branch, determine tracking state by calling
     `branches::detect_tracking_state(&repo, &branch)` (the shared helper
     from phase 1)
  4. Skip branches that don't match the criteria (gone always, untracked only
     if `include_no_remote`)
  5. Check safety: skip if `branch.is_head()` or if name is in `main_branches`
     → `TrimOutcome::Protected`
  6. If `dry_run`, emit `TrimOutcome::WouldDelete`
  7. Otherwise call `branch.delete()` → `Deleted` or `Failed`

- [x] Implement `trim_all()` for parallel execution across repos:

```rust
pub fn trim_all(
    entries: &[&RepoEntry],
    main_branches: &[String],
    include_no_remote: bool,
    dry_run: bool,
) -> Vec<(String, Result<RepoTrimResult, String>)>
```

  Uses `rayon::par_iter()` matching the pattern in `branches::list_branches_all()`.
  Parallelism is across repos (each `trim_repo` call operates on a different
  repository), not across branches within a single repo — so concurrent
  `Branch::delete()` calls never target the same repo.
  Returns per-repo results with errors for repos that fail to open, letting
  the caller (cmd_trim) report them separately.

#### 3. Register the trimmer module

**File**: `src/lib.rs`

- [x] Add `pub mod trimmer;` to module declarations

#### 4. Add command dispatch

**File**: `src/main.rs`

- [x] Add imports: `use cli::TrimArgs;`, `use gityard::trimmer;`, and
  `use gityard::trimmer::TrimOutcome;`
- [x] Add match arm in the `match cli.command` block (after `Branches`):

```rust
Command::Trim(args) => cmd_trim(&config_dir, &config, &args),
```

- [x] Implement `cmd_trim()`, following the `cmd_branches()` pattern of
  separating successes from errors:

```rust
fn cmd_trim(config_dir: &Path, config: &Config, args: &TrimArgs) -> Result<()> {
    let registry = Registry::load(config_dir)?;
    let entries = resolve_entries(&registry, args.group.as_deref())?;
    if entries.is_empty() {
        return Ok(());
    }

    let results = trimmer::trim_all(
        &entries,
        &config.defaults.main_branches,
        args.no_remote,
        args.dry_run,
    );

    let mut reports = Vec::new();
    for (alias, result) in &results {
        match result {
            Ok(report) => reports.push(report),
            Err(e) => eprintln!("error inspecting {alias}: {e}"),
        }
    }

    let output = presenter::format_trim_report(&reports, true);
    if !output.is_empty() {
        println!("{output}");
    }

    let has_failures = reports
        .iter()
        .flat_map(|r| &r.branches)
        .any(|b| matches!(b.outcome, TrimOutcome::Failed { .. }));

    if has_failures {
        anyhow::bail!("some branches could not be deleted");
    }

    Ok(())
}
```

#### 5. Add trim report formatting

**File**: `src/presenter.rs`

- [x] Add `use crate::trimmer::{RepoTrimResult, TrimOutcome};` import
- [x] Implement `format_trim_report()`:

```rust
pub fn format_trim_report(reports: &[&RepoTrimResult], colors: bool) -> String
```

  Format per-repo, per-branch results following the executor report style:
  - `✓ alias: deleted branch-name (gone)` — green checkmark
  - `✓ alias: would delete branch-name (gone)` — cyan for dry-run
  - `─ alias: protected branch-name (HEAD)` — yellow dash
  - `✗ alias: failed branch-name: reason` — red cross

  Summary footer: `N deleted · N protected · N failed` (or `N would delete`
  in dry-run mode)

  When a repo has no matching branches, omit it from the output entirely.

#### Tests for This Phase

**Unit tests in `src/trimmer.rs`:**

- [x] Test that HEAD branch is protected (returns `Protected`)
- [x] Test that main branches are protected
- [x] Test that gone branches are deleted
- [x] Test that untracked branches are skipped when `include_no_remote` is false
- [x] Test that untracked branches are deleted when `include_no_remote` is true
- [x] Test dry-run mode returns `WouldDelete` without actually deleting

**Unit tests in `src/presenter.rs`:**

- [x] Test `format_trim_report` output for a mix of deleted/protected/failed
- [x] Test summary footer counts
- [x] Test dry-run output shows "would delete"

**Integration tests in `tests/cli.rs`:**

- [x] Test `trim --dry-run` on a repo with a gone branch shows "would delete"
  and does not actually delete
- [x] Test `trim` on a repo with a gone branch deletes the branch
- [x] Test `trim` protects HEAD branch (does not delete the current branch)
- [x] Test `trim --no-remote` deletes branches with no upstream
- [x] Test `trim` with no matching branches produces clean output
- [x] Test `trim --group` scopes to the specified group
- [ ] Test exit code is non-zero when a deletion fails
  > **Skipped:** This test requires forcing `Branch::delete()` to fail, which is
  > difficult to trigger reliably via the CLI (would need a read-only filesystem
  > or locked ref). The unit test for `TrimOutcome::Failed` in the presenter
  > covers the formatting path. The exit-code logic in `cmd_trim` is a simple
  > `anyhow::bail!` conditional on `has_failures`, matching the proven pattern
  > from `cmd_exec`.

### Success Criteria:

#### Automated Verification:

- [x] `just check` passes
  > **Deviation:** Same as prior phases — verified via clippy + fmt + test individually.
- [x] All new trimmer unit tests pass
- [x] All new presenter tests for trim report pass
- [x] All new integration tests for `trim` pass
- [x] Existing tests are unaffected

---

## Testing Strategy

### Cross-Phase Testing:

- [ ] End-to-end: `gityard fetch` (to prune remotes) → `gityard branches --gone`
  (to see gone branches) → `gityard trim --dry-run` (to preview) →
  `gityard trim` (to delete) — verify the full user workflow
- [ ] Multi-repo: test trim across multiple repos where some have gone branches,
  some have untracked branches, and some have neither

### Test Helper Additions:

A new test helper will be needed in both `tests/cli.rs` and `src/branches.rs`
unit tests to create the "gone" state:

1. Create a repo with a remote and push a branch
2. Set up local tracking
3. Delete the remote-tracking ref (simulating `git fetch --prune` after the
   remote branch was deleted)

This can be done with git2 by calling `repo.find_reference("refs/remotes/origin/branch-name")`
and then `.delete()` on the reference.

## References

- Idea document: `thoughts/ideas/2026-04-12-gityard-trim-command.md`
- Branch inspection: `src/branches.rs` — `BranchInfo` struct, `list_branches()`
- Presenter: `src/presenter.rs` — `format_branches_table()` (line 165)
- CLI args: `src/cli.rs` — `BranchesArgs` (line 132), `StatusArgs` (line 89)
- Filter pattern: `src/filter.rs` — `StatusFilter` (line 8)
- Executor pattern: `src/executor.rs` — `ExecOutcome` (line 12),
  `format_report()` (line 139)
- git2 "gone" detection: `repo.branch_upstream_name()` + `repo.find_reference()`
- git2 branch deletion: `Branch::delete()`, guarded by `Branch::is_head()`
