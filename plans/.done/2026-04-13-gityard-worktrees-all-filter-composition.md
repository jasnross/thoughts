# Fix `--all` + Filter Composition in `worktrees` Command

## Overview

Fix the `worktrees` command so that `--all` and filter flags (`--dirty`,
`--gone`) compose correctly. Currently `--all --dirty` shows unfiltered main
tree entries alongside filtered linked worktrees. The fix applies filters
uniformly to all entries — main and linked — so `--all` expands the candidate
repos while filters narrow the visible entries.

## Current State Analysis

### The bug

`cmd_worktrees()` at `src/main.rs:332-351` separates main tree entries from
linked worktrees and only applies the filter to linked worktrees:

```rust
let main_tree: Option<&_> = repo_wts.worktrees.iter()
    .find(|w| w.kind == WorktreeKind::Main);
let linked: Vec<&_> = repo_wts.worktrees.iter()
    .filter(|w| !matches!(w.kind, WorktreeKind::Main))
    .filter(|w| filter.matches(w))
    .collect();
```

The skip logic at lines 346–351 keeps repos with `--all` even when no linked
worktrees match, so the unfiltered main tree appears alone.

### The `--all` flag

`--all` is described in `cli.rs:145` as "Include repos with no linked worktrees
(shows main working tree)." Its current effect is twofold:

1. Passes `include_main: true` to `list_worktrees()`, which constructs a
   `WorktreeInfo` with `kind: WorktreeKind::Main` for each repo
2. Overrides the skip logic so repos appear even with zero matching linked
   worktrees

### `WorktreeFilter` doc comment

`filter.rs:87-88` says: "Only applied to linked worktrees in the handler; the
main tree is always included for repos with at least one matching linked
worktree." This doc comment describes the current (buggy) behavior and needs
updating.

### Key Discoveries

- `WorktreeFilter::matches()` at `filter.rs:101-114` already works correctly on
  any `WorktreeInfo` regardless of kind — it checks `is_dirty` and `tracking`
  fields, which are populated for main tree entries too (`worktrees.rs:140-159`)
- The main tree's `is_dirty` and `tracking` fields are set from the same
  inspection logic as linked worktrees, so filtering the main tree is
  semantically correct
- No other command has an `--all`-style flag (`cli.rs` — checked all `*Args`
  structs)

## Desired End State

After this change:

1. `gityard worktrees --dirty` shows only dirty linked worktrees (unchanged)
2. `gityard worktrees --all` shows all repos including those with no linked
   worktrees, displaying the main tree (unchanged)
3. `gityard worktrees --all --dirty` shows only dirty entries across all repos —
   main tree included only if it is dirty, linked worktrees included only if
   dirty. Repos with zero dirty entries (main or linked) are omitted
4. Same composable behavior for `--all --gone` and `--all --dirty --gone`
5. `WorktreeFilter` doc comment reflects the new behavior

### Verification

- `just check` passes
- New and updated CLI tests verify the composition behavior

## What We're NOT Doing

- Adding visual distinction (dim/grey) for context rows — Option 3 from the
  original TODO. The uniform filtering approach is simpler and matches user
  expectations
- Changing the `--all` flag's help text beyond a minor clarification — the
  current description is accurate for the new behavior
- Auditing other commands for `--all`-style flags — no others exist

## Implementation

### Overview

Apply the filter uniformly to all worktree entries (main and linked), simplify
the skip logic, update the `WorktreeFilter` doc comment, update existing tests,
and add a new test for `--all` + filter composition.

### Changes Required

#### 1. Apply filter to all entries uniformly

**File**: `src/main.rs`

- [x] Replace the separate main-tree extraction and linked-only filtering
  (lines 332–358) with a single pass that filters all entries:

Replace:

```rust
// Separate main tree entries from linked worktrees.
let main_tree: Option<&_> = repo_wts
    .worktrees
    .iter()
    .find(|w| w.kind == WorktreeKind::Main);
let linked: Vec<&_> = repo_wts
    .worktrees
    .iter()
    .filter(|w| !matches!(w.kind, WorktreeKind::Main))
    .filter(|w| filter.matches(w))
    .collect();

// Skip repos with no content to display:
// - When filtering, skip if no linked worktrees match
// - When --all, keep repos even with no linked worktrees (show main tree)
if linked.is_empty() && main_tree.is_none() {
    continue;
}
if linked.is_empty() && !args.all {
    continue;
}

// Build the final list: main tree first (if --all), then linked.
let mut wts = Vec::new();
if let Some(main) = main_tree {
    wts.push(main);
}
wts.extend(linked);
```

With:

```rust
let wts: Vec<&_> = repo_wts
    .worktrees
    .iter()
    .filter(|w| filter.matches(w))
    .collect();

if wts.is_empty() {
    continue;
}
```

This works because:
- When `--all` is false, `list_worktrees()` returns only linked worktrees —
  the filter runs on linked worktrees only (same as before)
- When `--all` is true, `list_worktrees()` returns main + linked — the
  filter runs on all entries (the fix)
- When no filter flags are set, `filter.matches()` returns `true` for
  everything (unchanged behavior)
- Ordering is preserved: `list_worktrees()` already puts the main tree first
  in the vector (`worktrees.rs:157-158`)

#### 2. Update `WorktreeFilter` doc comment

**File**: `src/filter.rs`

- [x] Update the doc comment on `WorktreeFilter` (lines 82–88) to reflect that
  the filter is now applied to all worktree entries uniformly:

Replace:

```rust
/// Worktree filter for the `worktrees` command.
///
/// When no flags are set, all worktrees match. When one or more flags are set,
/// a worktree matches if it satisfies at least one active flag (OR semantics).
///
/// Only applied to linked worktrees in the handler; the main tree is always
/// included for repos with at least one matching linked worktree.
```

With:

```rust
/// Worktree filter for the `worktrees` command.
///
/// When no flags are set, all worktrees match. When one or more flags are set,
/// a worktree matches if it satisfies at least one active flag (OR semantics).
///
/// Applied uniformly to all worktree entries (main tree and linked worktrees).
```

#### 3. Update `--all` help text

**File**: `src/cli.rs`

- [x] Update the help text for `--all` (line 145) to clarify its interaction
  with filters:

Replace:

```rust
/// Include repos with no linked worktrees (shows main working tree)
```

With:

```rust
/// Include the main working tree for each repo
```

The current text focuses on the "repos with no linked worktrees" use case,
but `--all` now has a broader meaning: it includes the main tree as a
filterable entry for every repo. The new text is accurate for both
`--all` alone and `--all` combined with filters.

#### 4. Update existing test: `test_worktrees_dirty_filter`

**File**: `tests/cli.rs`

- [x] The existing `test_worktrees_dirty_filter` (line 873) creates two linked
  worktrees, dirties one, and asserts `--dirty` shows only the dirty one.
  This test continues to pass as-is because `--dirty` without `--all` only
  sees linked worktrees. No change needed — verify it still passes.

#### 5. Update existing test: `test_worktrees_all_includes_main_tree`

**File**: `tests/cli.rs`

- [x] The existing `test_worktrees_all_includes_main_tree` (line 947) asserts
  that `--all` shows both the main branch and the linked worktree branch.
  This test continues to pass because no filter is active, so all entries
  match. No change needed — verify it still passes.

#### 6. Add new test: `test_worktrees_all_dirty_filters_main_tree`

**File**: `tests/cli.rs`

- [x] Add a test that verifies `--all --dirty` filters out a clean main tree.
  Setup: use `init_repo_with_worktree`, dirty the linked worktree, register
  the repo. Run `gityard worktrees --all --dirty`. Assert:
  - The dirty linked worktree branch (`feature-wt`) appears in output
  - The clean main branch does NOT appear in output

```rust
#[test]
#[allow(clippy::expect_used)]
fn test_worktrees_all_dirty_filters_main_tree() {
    let tmp = TempDir::new().expect("tempdir");
    let config_dir = tmp.path().join("config");
    let (repo_path, wt_branch, wt_path) = init_repo_with_worktree(tmp.path());

    // Dirty the linked worktree
    std::fs::write(wt_path.join("file.txt"), "modified content").expect("write");

    // Detect the main branch name
    let main_branch = {
        let repo = git2::Repository::open(&repo_path).expect("open");
        let head = repo.head().expect("head");
        head.shorthand().expect("shorthand").to_string()
    };

    gityard_with_config(&config_dir)
        .args(["add", repo_path.to_str().expect("utf8")])
        .assert()
        .success();

    gityard_with_config(&config_dir)
        .args(["worktrees", "--all", "--dirty"])
        .assert()
        .success()
        .stdout(
            predicate::str::contains(&wt_branch)
                .and(predicate::str::contains(&main_branch).not()),
        );
}
```

#### 7. Add new test: `test_worktrees_all_dirty_no_matches`

**File**: `tests/cli.rs`

- [x] Add a test that verifies `--all --dirty` omits repos entirely when
  nothing is dirty. Setup: use `init_repo_with_worktree` (all worktrees are
  clean by default), register the repo. Run
  `gityard worktrees --all --dirty`. Assert no table output (no `WORKTREE`
  header) and stderr contains "no worktrees match the given filters".

```rust
#[test]
#[allow(clippy::expect_used)]
fn test_worktrees_all_dirty_no_matches() {
    let tmp = TempDir::new().expect("tempdir");
    let config_dir = tmp.path().join("config");
    let (repo_path, _wt_branch, _wt_path) = init_repo_with_worktree(tmp.path());

    gityard_with_config(&config_dir)
        .args(["add", repo_path.to_str().expect("utf8")])
        .assert()
        .success();

    gityard_with_config(&config_dir)
        .args(["worktrees", "--all", "--dirty"])
        .assert()
        .success()
        .stdout(predicate::str::contains("WORKTREE").not())
        .stderr(predicate::str::contains("no worktrees match"));
}
```

#### 8. Remove TODO entry

**File**: `TODO.md`

- [x] Remove the "Filter flags don't compose with `--all`" section from
  `TODO.md`. After this change, `TODO.md` should be empty (only the
  `# TODO` heading remains). If the file is empty, delete it entirely.

### Success Criteria

#### Automated Verification

- [x] `just check` passes (format + lint + build + test + license check)
- [x] All existing worktree CLI tests pass: `cargo test --test cli test_worktrees`
- [x] New tests pass: `cargo test --test cli test_worktrees_all_dirty`
- [x] `cargo clippy --all-targets` passes with no new warnings

## References

- Original TODO: `TODO.md:3` (to be removed)
- Original design plan: `thoughts/plans/2026-04-12-gityard-worktree-management.md`
- Prior test coverage plan: `thoughts/plans/2026-04-13-gityard-worktree-test-coverage-and-symlink-fix.md`
- `src/main.rs:314-384` — `cmd_worktrees()` handler
- `src/filter.rs:82-115` — `WorktreeFilter` struct and matching
- `src/worktrees.rs:132-236` — `list_worktrees()` data collection
- `src/cli.rs:136-151` — `WorktreesArgs` definition
- `tests/cli.rs:854-992` — existing worktree CLI tests
