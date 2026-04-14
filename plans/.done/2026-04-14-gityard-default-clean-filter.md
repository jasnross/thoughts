# Default "Hide Clean" Filtering Implementation Plan

## Overview

Add default clean-hiding to `gityard status` and `gityard branches` so that
"totally clean" items are hidden by default. Add `--all` to show everything,
redefine `--clean` to show only totally-clean items, and add `--topic` to show
only non-main items. When any explicit filter flag is passed, default hiding is
disabled and the flag applies to the full set.

Reference: `thoughts/ideas/2026-04-14-gityard-default-clean-filter.md`

## Current State Analysis

Both commands collect data in parallel via rayon, apply post-collection filters
with OR semantics, then pass filtered results to the presenter. The presenter
has no knowledge of what was filtered — it receives only the post-filter slice.

### Key Discoveries:

- `StatusFilter` (`filter.rs:10-48`) has 5 boolean flags; `is_empty()` means
  "show all." The existing `--clean` flag at `filter.rs:33` checks only
  `!state.is_dirty`, not the full "totally clean" definition.
- `BranchFilter` (`filter.rs:54-79`) has 2 boolean flags (`gone`, `no_remote`).
- Neither `RepoState` nor `BranchInfo` currently tracks whether the item is on/is
  a main branch — this must be computed at inspection time.
- The presenter functions receive `&[&RepoState]` or `&[(String, Vec<&BranchInfo>)]`
  with no additional metadata. The hidden-items footer requires passing extra
  information (either a count or having `cmd_*` print it directly).
- `format_branches_table` (`presenter.rs:209-278`) has no summary footer, unlike
  `format_status_table` and `format_worktrees_table` which do.

## Desired End State

Running `gityard status` with no flags hides repos that are on main, not dirty,
and synced with both main and remote. Running `gityard branches` with no flags
hides branches that are merged into main and have active upstream tracking
(including the main branch itself). Both commands print a footer like
`(3 clean repos hidden — use --all to show)` when items are hidden.

### Verification:

```sh
just check                     # full suite passes
gityard status                 # hides totally-clean repos, shows footer
gityard status --all           # shows everything (current behavior)
gityard status --dirty         # shows only dirty (unchanged behavior)
gityard status --clean         # shows only totally-clean repos
gityard status --topic         # shows repos on non-main branches
gityard branches               # hides merged+tracked branches (including main)
gityard branches --all         # shows everything (current behavior)
gityard branches --clean       # shows only merged+tracked branches
gityard branches --topic       # shows non-main branches
```

## What We're NOT Doing

- Changing what data is collected per repo/branch
- Modifying the `worktrees` subcommand
- Adding a `--quiet` flag to suppress the hidden-items footer
- Persisting filter preferences in config
- Including stash count in the "clean" definition

## Implementation Approach

The change is structured bottom-up: data layer additions, then filter logic, then
orchestration wiring, then presenter output. The key architectural decision is to
**not change existing filter semantics**. Instead, default hiding is a separate
concern handled in `cmd_status`/`cmd_branches` before delegating to the existing
filter pipeline.

The logic in each command function becomes:

```
if --all:
    apply explicit filter to full set (is_empty → show all, same as today)
else if any explicit flag is set:
    apply explicit filter to full set (same as today)
else:
    hide totally-clean items (new default)
```

This means the existing `StatusFilter`/`BranchFilter` OR-semantics code is
untouched for the explicit-flag path. The `--all` flag is not a filter field — it
controls which code path runs, similar to how `worktrees --all` controls data
collection rather than filtering.

## Implementation

### Overview

Add `is_on_main_branch` to `RepoState` and `is_main` to `BranchInfo` during
data collection. Add `is_totally_clean` functions to `filter.rs`. Redefine
`--clean` and add `--all`/`--topic` flags. Wire up three-way dispatch in
command functions. Add hidden-items footer to presenter output.

### Changes Required:

#### 1. Data Layer — `RepoState` and `BranchInfo`

**File**: `src/inspector.rs`

- [x] Add `is_on_main_branch: bool` field to `RepoState` (after `branch` at
  line 16). Doc comment: "Whether the current branch is one of the configured
  main branches."
- [x] Compute the field in `inspect()` after `read_branch()` returns
  (`inspector.rs:75`):
  ```rust
  let is_on_main_branch = main_branches.iter().any(|m| m == &branch_name);
  ```
  Note: for detached HEAD (`"HEAD detached at abc1234"`) and unborn branches
  (`"(unborn)"`), the comparison correctly returns `false` — these are never
  considered "on main."
- [x] Include `is_on_main_branch` in the `RepoState` construction at line 99.

**File**: `src/branches.rs`

- [x] Add `is_main: bool` field to `BranchInfo` (after `name` at line 23). Doc
  comment: "Whether this branch is one of the configured main branches."
- [x] Compute the field in `list_branches()` after extracting `name`
  (`branches.rs:75`):
  ```rust
  let is_main = main_branches.iter().any(|m| m == &name);
  ```
- [x] Include `is_main` in the `BranchInfo` construction at line 111.

#### 2. Filter Layer — Clean Detection and `--topic`

**File**: `src/filter.rs`

- [x] Add public function `is_totally_clean_status(state: &RepoState) -> bool`
  that returns true when ALL of:
  - `state.is_on_main_branch`
  - `!state.is_dirty`
  - `matches!(state.ahead_main, Some(0)) && matches!(state.behind_main, Some(0))`
    (use explicit `Some(0)` rather than `unwrap_or(0)` — when `is_on_main_branch`
    is true these are always `Some`, but explicit matching is more robust and
    correctly rejects repos where no main branch was found)
  - `state.ahead_remote.unwrap_or(0) == 0 && state.behind_remote.unwrap_or(0) == 0`
    (`None` means no upstream configured — treat as "nothing to be out of sync
    with," so local-only repos without remotes are not forced to always show up)

- [x] Add public function `is_totally_clean_branch(branch: &BranchInfo) -> bool`
  that returns true when BOTH of:
  - `branch.is_merged`
  - `matches!(branch.tracking, TrackingState::Tracking { .. })`

- [x] Add `topic: bool` field to `StatusFilter` (line 15). Add the match arm
  in `matches()`: `if self.topic && !state.is_on_main_branch { return true; }`

- [x] Redefine the `clean` match arm in `StatusFilter::matches()` (line 33):
  change from `!state.is_dirty` to `is_totally_clean_status(state)`.
  **Breaking change**: existing `--clean` users who expected "not dirty" will
  now get the stricter "totally clean" definition. This is intentional — see
  the idea document's resolved questions.

- [x] Update `StatusFilter::is_empty()` (line 21) to include `&& !self.topic`.

- [x] Add `topic: bool` and `clean: bool` fields to `BranchFilter` (line 54-57).

- [x] Add match arms in `BranchFilter::matches()`:
  - `topic`: `if self.topic && !branch.is_main { return true; }`
  - `clean`: `if self.clean && is_totally_clean_branch(branch) { return true; }`

- [x] Update `BranchFilter::is_empty()` (line 62) to include
  `&& !self.topic && !self.clean`.

#### 3. CLI Layer — New Flags

**File**: `src/cli.rs`

- [x] Add to `StatusArgs` (after `diverged` at line 106):
  ```rust
  /// Show only repositories on a topic (non-main) branch
  #[arg(long)]
  pub topic: bool,
  /// Show all repositories, including totally clean ones
  #[arg(long)]
  pub all: bool,
  ```

- [x] Update `StatusArgs` doc comment for `clean` (line 96) from "Show only
  clean repositories" to "Show only totally clean repositories (on main, not
  dirty, synced with main and remote)".

- [x] Add to `BranchesArgs` (after `no_remote` at line 156):
  ```rust
  /// Show only branches that are not a main branch
  #[arg(long)]
  pub topic: bool,
  /// Show only totally clean branches (merged with active upstream tracking)
  #[arg(long)]
  pub clean: bool,
  /// Show all branches, including totally clean ones
  #[arg(long)]
  pub all: bool,
  ```

- [x] Add `conflicts_with` attributes to enforce mutual exclusivity (clap's
  `conflicts_with` is symmetric — declaring on one side is sufficient):
  - `StatusArgs::all`: `#[arg(long, conflicts_with = "clean")]`
  - `BranchesArgs::all`: `#[arg(long, conflicts_with = "clean")]`
  - `StatusArgs::clean`: `#[arg(long, conflicts_with_all = ["dirty", "all"])]`
  - `BranchesArgs::clean`: `#[arg(long, conflicts_with = "all")]`

#### 4. Orchestration Layer — Command Functions

**File**: `src/main.rs`

- [x] Update `cmd_status` (line 172-226). Replace the current filter
  construction and application (lines 194-202) with three-way dispatch:

  ```rust
  let filter = StatusFilter {
      dirty: args.dirty,
      clean: args.clean,
      ahead: args.ahead,
      behind: args.behind,
      diverged: args.diverged,
      topic: args.topic,
  };

  let (filtered, hidden_count) = if args.all || !filter.is_empty() {
      // Explicit flags or --all: apply filter to full set (is_empty → show all)
      let result: Vec<&_> = states.into_iter().filter(|s| filter.matches(s)).collect();
      (result, 0)
  } else {
      // Default: hide totally-clean items
      let mut shown = Vec::new();
      let mut hidden = 0usize;
      for s in states {
          if gityard::filter::is_totally_clean_status(s) {
              hidden += 1;
          } else {
              shown.push(s);
          }
      }
      (shown, hidden)
  };
  ```

- [x] Update the empty-result messaging (lines 204-212). When `filtered` is
  empty and `hidden_count > 0`, print the hidden-items message to stderr rather
  than "all repositories are clean and synced":
  ```rust
  if filtered.is_empty() && errors.is_empty() {
      if hidden_count > 0 {
          let label = if hidden_count == 1 { "repo" } else { "repos" };
          eprintln!("all repos are clean · {hidden_count} {label} hidden — use --all to show");
      } else if !filter.is_empty() {
          eprintln!("no repositories match the given filters");
      } else {
          // --all with no repos, or genuinely no repos registered
          eprintln!("all repositories are clean and synced");
      }
      return Ok(());
  }
  ```

- [x] After printing the status table (line 216), print the hidden-items footer
  to stdout if `hidden_count > 0`:
  ```rust
  if hidden_count > 0 {
      let label = if hidden_count == 1 { "repo" } else { "repos" };
      println!("\n ({hidden_count} clean {label} hidden — use --all to show)");
  }
  ```

- [x] Update `cmd_branches` (line 379-423) with the same three-way dispatch
  pattern. The filter construction (lines 385-388) becomes:
  ```rust
  let filter = BranchFilter {
      gone: args.gone,
      no_remote: args.no_remote,
      topic: args.topic,
      clean: args.clean,
  };
  ```

- [x] Apply default hiding in the per-repo branch filtering loop
  (lines 392-400):
  ```rust
  let apply_default_hiding = !args.all && filter.is_empty();

  let mut total_hidden = 0usize;
  for (alias, _path, result) in &results {
      match result {
          Ok(branch_list) => {
              let refs: Vec<&_> = if apply_default_hiding {
                  branch_list
                      .iter()
                      .filter(|b| {
                          if gityard::filter::is_totally_clean_branch(b) {
                              total_hidden += 1;
                              false
                          } else {
                              true
                          }
                      })
                      .collect()
              } else {
                  branch_list.iter().filter(|b| filter.matches(b)).collect()
              };
              if !refs.is_empty() {
                  table_data.push((alias.clone(), refs));
              }
          }
          Err(e) => { /* unchanged */ }
      }
  }
  ```

- [x] Add hidden-items footer after `format_branches_table` output
  (after line 420):
  ```rust
  if total_hidden > 0 {
      let label = if total_hidden == 1 { "branch" } else { "branches" };
      println!("\n ({total_hidden} clean {label} hidden — use --all to show)");
  }
  ```

- [x] Update the empty-result messaging for branches (lines 408-417) to handle
  the case where everything was hidden by default, matching the status pattern.

#### 5. Test Fixture Updates

**File**: `src/filter.rs` (test module, starting at line 133)

- [x] Update `make_state` helper (line 139) to include
  `is_on_main_branch: true` in the default `RepoState`.

- [x] Update `make_branch_info` helper (line 247) to include `is_main: false`
  in the default `BranchInfo`.

- [x] Update `filter` helper (line 158) to include `topic: false` in the
  default `StatusFilter`.

- [x] Update `branch_filter` helper (line 259) to include `topic: false` and
  `clean: false` in the default `BranchFilter`.

#### Tests

**File**: `src/filter.rs` (unit tests)

- [x] Test `is_totally_clean_status`: verify all four conditions must be true.
  Test each condition in isolation being false while others are true → not clean.
  Test all true → clean. Test edge cases: `ahead_main: None` (no main branch
  found), `ahead_remote: None` (no upstream).

- [x] Test `is_totally_clean_branch`: verify both conditions must be true.
  Merged + Tracking → clean. Merged + Gone → not clean. Merged + Untracked →
  not clean. Not merged + Tracking → not clean.

- [x] Test redefined `--clean` on `StatusFilter`: should now match only
  totally-clean repos (not just `!is_dirty`). A repo that is not dirty but on a
  feature branch should NOT match `--clean`.

- [x] Test `--topic` on `StatusFilter`: matches repos where
  `!is_on_main_branch`. Does not match repos on main.

- [x] Test `--topic` on `BranchFilter`: matches branches where `!is_main`. Does
  not match main branches.

- [x] Test `--clean` on `BranchFilter`: matches only
  `is_totally_clean_branch()` branches.

- [x] Test combined flags with `--topic`: `--topic --dirty` should match repos
  that are on a topic branch OR are dirty (OR semantics preserved).

**File**: `tests/cli.rs` (integration tests)

- [x] Test default status hiding: register a clean repo (on main, no changes,
  synced) and a dirty repo. `gityard status` should show only the dirty repo
  and contain "hidden" in output.

- [x] Test `gityard status --all`: both repos appear.

- [x] Test `gityard status --topic`: register a repo on a feature branch and
  one on main. Only the feature-branch repo should appear.

- [x] Test existing `--dirty` flag still works unchanged (backward compat):
  `gityard status --dirty` shows dirty repos from the full set, including those
  that would be hidden by default.

- [x] Update `test_status_dirty_filter` (line 160): the existing test asserts
  `clean-repo` is NOT in `--dirty` output. With default hiding, `gityard status`
  (no flags) would also hide the clean repo. Verify the existing test still
  passes as-is (it should — `--dirty` is an explicit flag that overrides default
  hiding).
  > **Deviation:** Two additional existing tests required updates:
  > `test_status_with_registered_repos` now uses `--all` because its clean repo
  > is hidden by default. `test_status_with_group_filter` similarly now uses
  > `--all`. Both tests were about verifying table output format, not filtering
  > behavior, so adding `--all` preserves their intent.

### Success Criteria:

#### Automated Verification:

- [x] `just check` passes (format + lint + build + test + license check)
- [x] All existing tests in `src/filter.rs` pass without modification to test
  logic (only fixture updates for new fields)
- [x] All existing tests in `tests/cli.rs` pass
- [x] New unit tests for `is_totally_clean_status` and `is_totally_clean_branch`
  pass
- [x] New unit tests for `--topic` and redefined `--clean` pass
- [x] New integration tests for default hiding and `--all` pass
- [x] `--all` and `--clean` are enforced as mutually exclusive by clap
  (verify with `gityard status --all --clean` producing an error)
- [x] `--clean` and `--dirty` are enforced as mutually exclusive on status

#### Manual Verification:

- [x] Run `gityard status` against real registered repos and confirm clean repos
  are hidden with a footer showing the count
- [x] Run `gityard branches` against real repos and confirm merged+tracked
  branches (including main) are hidden
- [x] Run `gityard status --all` and `gityard branches --all` and confirm output
  matches pre-change behavior
