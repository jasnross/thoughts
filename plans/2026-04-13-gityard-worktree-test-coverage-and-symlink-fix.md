# Worktree CLI Test Coverage and display_path Symlink Fix

## Overview

Add CLI integration tests for `gityard worktrees` and `gityard trim` commands
that exercise real linked worktrees end-to-end, and fix a symlink-related bug
in `display_path` by canonicalizing `WorktreeInfo.path` at construction time.
Separately, add edge-case tests for submodule and bare repo behavior.

## Current State Analysis

### CLI test coverage gaps

`tests/cli.rs` has 25 tests covering most commands. The `worktrees` command
only has two tests: `test_worktrees_command_no_repos` and
`test_worktrees_command_help`. There are no CLI tests exercising `gityard
worktrees` with actual linked worktrees or `gityard trim` on repos that have
worktrees. The unit tests in `worktrees.rs` (7 tests) and `trimmer.rs` (15
tests) thoroughly cover the logic, but CLI-level tests are needed to catch
argument parsing, output formatting, and cross-module wiring regressions.

### The symlink bug

`WorktreeInfo.path` is assigned from `git2::Worktree::path()` at `worktrees.rs`
line 175 without canonicalization (lines 201 and 220 are the `path: wt_path`
field assignments in the valid/invalid arms that consume this value). On
macOS, `/var` is a symlink to `/private/var`, so `Worktree::path()` may return a
differently-resolved path than `entry.path` (which is canonicalized at
`registry.rs:74-76`). This causes `display_path` in `presenter.rs:284-293` to:
1. Fail the `==` comparison for the main tree check (line 286)
2. Produce incorrect relative paths from `pathdiff::diff_paths` (line 289)

The same issue exists in `trimmer.rs:256` where `build_worktree_branch_map`
assigns `wt.path().to_path_buf()` — the path appears in `TrimOutcome` display
fields.

### Key Discoveries

- `registry.rs:74-76` — `entry.path` is canonicalized eagerly; this is the
  project convention
- `worktrees.rs:357-361` — The existing unit test
  `test_list_worktrees_with_linked_worktree` works around the symlink issue by
  comparing `.canonicalize().ok()` on both sides
- `tests/cli.rs` has helpers `init_real_git_repo`, `init_repo_with_remote`,
  `init_repo_with_gone_branch` — new tests can follow these patterns
- `gityard worktrees` output includes columns: REPO, WORKTREE, BRANCH, STATUS,
  REMOTE, LAST COMMIT
- `gityard trim` output renders worktree paths via `.display()` directly on the
  `PathBuf` in `TrimOutcome` variants (in `presenter::format_trim_report`) —
  it does NOT use the `display_path` helper. Canonicalizing at construction time
  in the trimmer ensures these `.display()` calls produce canonical paths

## Desired End State

After this plan is complete:

1. `WorktreeInfo.path` and trimmer's internal `WorktreeState.path` are
   canonicalized at construction time, consistent with `entry.path`
2. `display_path` produces correct relative paths regardless of symlink
   resolution differences
3. The existing workaround in `test_list_worktrees_with_linked_worktree` is
   simplified to a direct `==` comparison
4. CLI integration tests exercise `gityard worktrees` and `gityard trim` with
   actual linked worktrees
5. Edge-case behavior for submodules and bare repos is codified in tests
6. All existing tests continue to pass

### Verification

- `just check` passes (format + lint + build + test + license check)
- New CLI tests exercise the full command pipeline (argument parsing → module
  logic → presenter output)

## What We're NOT Doing

- Full first-class support for submodules or bare repos — current behavior is
  acceptable; tests codify it
- Expanding the existing presenter golden tests (unless the symlink fix changes
  output)
- Fixing the `--all`/filter composition issue (TODO item #1) — separate design
  decision
- Fixing the `worktrees --all` phantom entry for bare repos — noted in tests,
  but the registration gate already prevents bare repos from being registered
  normally

## Implementation Approach

The changes are atomic: the symlink fix is 4 lines across 2 files, the test
simplification is 1 assertion change, and the new tests are additive. A single
implementation section keeps things simple.

## Implementation

### Overview

Fix path canonicalization at `WorktreeInfo` and `WorktreeState` construction
time, simplify the existing workaround test, and add CLI integration tests.

### Changes Required

#### 1. Canonicalize `WorktreeInfo.path` at construction time

**File**: `src/worktrees.rs`

- [x] Line 175: Add canonicalization after `let wt_path = wt.path().to_path_buf();`

There is a single assignment site at line 175. The `wt_path` is then consumed in
both the valid (line 201 `path: wt_path`) and invalid (line 220 `path: wt_path`)
`WorktreeInfo` construction arms, so canonicalizing once at the source fixes both
paths. The `unwrap_or` fallback handles invalid worktrees whose directory no
longer exists on disk (canonicalize would fail).

```rust
let wt_path = wt.path().to_path_buf();
let wt_path = wt_path.canonicalize().unwrap_or(wt_path);
```

#### 2. Canonicalize `WorktreeState.path` in the trimmer

**File**: `src/trimmer.rs`

- [x] Line 256: Add canonicalization after `let wt_path = wt.path().to_path_buf();`

```rust
let wt_path = wt.path().to_path_buf();
let wt_path = wt_path.canonicalize().unwrap_or(wt_path);
```

This ensures the worktree paths in all `TrimOutcome` variants that carry a
`worktree_path` field are canonical: `DeletedAfterWorktreeRemoval`,
`WouldDeleteAfterWorktreeRemoval`, `ProtectedDirtyWorktree`, and
`ProtectedLockedWorktree`. All flow from the same `WorktreeState.path` in
`build_worktree_branch_map`.

#### 3. Simplify the existing unit test workaround

**File**: `src/worktrees.rs` (test module)

- [x] In `test_list_worktrees_with_linked_worktree` (lines 357-361): Replace the
  `.canonicalize().ok()` comparison with a direct `==` assertion

Replace:
```rust
assert_eq!(
    wt.path.canonicalize().ok(),
    wt_path.canonicalize().ok(),
    "path should match (after canonicalization)"
);
```

With:
```rust
assert_eq!(
    wt.path,
    wt_path.canonicalize().unwrap_or(wt_path),
    "path should be pre-canonicalized"
);
```

The `wt_path` variable in the test is the raw temp path (not canonicalized), so
we still need to canonicalize the expected side. But `wt.path` should now be
pre-canonicalized, so only one side needs the transform.

#### 4. CLI integration tests for `gityard worktrees`

**File**: `tests/cli.rs`

Add a helper function and new tests at the end of the worktrees section:

- [x] Add `init_repo_with_worktree` helper (creates a real git repo with an
  initial commit and a linked worktree, returns repo path + worktree branch
  name + worktree path). Note: an identically-named helper exists in the
  `worktrees.rs` unit test module (`src/worktrees.rs:269`), but since they are
  in separate crates (`tests/cli.rs` is an integration test binary) there is no
  conflict. The CLI version delegates to `init_real_git_repo` for consistency
  with other CLI test helpers.

```rust
/// Create a real git repo with an initial commit (including a tracked file)
/// and a linked worktree.
///
/// Unlike `init_real_git_repo` (which creates an empty-tree commit), this
/// helper writes `file.txt` so that dirty-worktree tests can modify an
/// existing tracked file (producing a "modified" status, not just "untracked").
///
/// Returns (`repo_path`, `worktree_branch_name`, `worktree_path`).
#[allow(clippy::expect_used)]
fn init_repo_with_worktree(tmp: &Path) -> (PathBuf, String, PathBuf) {
    let repo_path = tmp.join("repo");
    std::fs::create_dir_all(&repo_path).expect("create dir");
    let repo = git2::Repository::init(&repo_path).expect("git init");
    let sig = git2::Signature::now("Test", "test@example.com").expect("sig");
    std::fs::write(repo_path.join("file.txt"), "content").expect("write");
    {
        let mut index = repo.index().expect("index");
        index.add_path(Path::new("file.txt")).expect("add");
        index.write().expect("write index");
        let tree_id = index.write_tree().expect("write_tree");
        let tree = repo.find_tree(tree_id).expect("find_tree");
        repo.commit(Some("HEAD"), &sig, &sig, "initial", &tree, &[])
            .expect("commit");
    }

    let head_commit = repo.head().expect("head").peel_to_commit().expect("peel");

    let wt_branch = "feature-wt";
    repo.branch(wt_branch, &head_commit, false).expect("branch");

    let wt_path = tmp.join("worktree-feature");
    let reference = repo
        .find_reference(&format!("refs/heads/{wt_branch}"))
        .expect("find ref");
    let mut opts = git2::WorktreeAddOptions::new();
    opts.reference(Some(&reference));
    repo.worktree(wt_branch, &wt_path, Some(&opts))
        .expect("create worktree");

    (repo_path, wt_branch.to_string(), wt_path)
}
```

- [x] `test_worktrees_lists_linked_worktree` — Register a repo with a linked
  worktree, run `gityard worktrees`, assert output contains the worktree branch
  name and "clean" status

- [x] `test_worktrees_dirty_filter` — Use `init_repo_with_worktree` (creates
  repo with `feature-wt` worktree). Then create a second branch `feature-wt2`
  from HEAD and a second worktree at `tmp.join("worktree-feature2")`. Dirty
  `feature-wt` by modifying `worktree-feature/file.txt` (tracked file from the
  helper's initial commit). Run `gityard worktrees --dirty`, assert output
  contains `feature-wt` and does NOT contain `feature-wt2`. Note: after
  `init_repo_with_worktree`, HEAD is on the main branch (not `feature-wt`), so
  creating the second branch from HEAD is safe

- [x] `test_worktrees_gone_filter` — Use `init_repo_with_gone_branch` (returns
  `(clone_path, gone_branch_name)`; HEAD is on main after return — see
  `cli.rs:510-511`). Then create a linked worktree on the gone branch: find the
  ref, create `WorktreeAddOptions` with that reference, call
  `repo.worktree(gone_branch_name, wt_path, opts)`. Run
  `gityard worktrees --gone`, assert the gone branch name appears in output
  with "gone" in its REMOTE column

- [x] `test_worktrees_all_includes_main_tree` — Use `init_repo_with_worktree`,
  run `gityard worktrees --all`, assert output contains both the linked worktree
  branch (`feature-wt`) AND the repo's own HEAD branch name. To detect the main
  branch name, read it from the repo after setup (it varies by git version:
  `main` or `master`). The presenter shows the repo's absolute path in the
  WORKTREE column for the main tree, so asserting the output has two data rows
  (one for each tree) and contains both branch names is sufficient

- [x] `test_worktrees_no_linked_worktrees_empty_output` — Register a repo with
  no linked worktrees, run `gityard worktrees`, assert empty/no-output behavior
  (stderr message or empty stdout)

#### 5. CLI integration tests for `gityard trim` with worktrees

**File**: `tests/cli.rs`

- [x] Add `init_repo_with_gone_branch_and_worktree` helper that combines
  `init_repo_with_gone_branch` (creates gone branch) with worktree creation
  (creates a linked worktree checked out on the gone branch). Returns
  `(repo_path, gone_branch_name, worktree_path)`. Modeled on the unit test
  helper at `trimmer.rs:943-962` but with a 3-tuple return (the CLI tests don't
  need `main_branch_name` since they don't assert HEAD movement).

- [x] `test_trim_with_clean_worktree_removes_both` — Use the combined helper,
  run `gityard trim`, assert output contains "deleted" and the gone branch name.
  After trim, verify the worktree directory no longer exists on disk
  (`!wt_path.exists()`) and the branch is gone (`repo.find_branch(...).is_err()`)

- [x] `test_trim_with_dirty_worktree_protects` — Use the combined helper, write
  a file to the worktree to make it dirty, run `gityard trim`, assert output
  indicates the branch was not deleted (e.g., contains "dirty" or "protected").
  Verify the worktree directory and branch still exist

- [x] `test_trim_dry_run_with_worktree` — Same setup as clean worktree test but
  with `--dry-run`, assert "would delete" in output. Verify worktree directory
  and branch still exist afterward

#### 6. Submodule behavior test

**File**: `tests/cli.rs`

- [x] `test_worktrees_ignores_submodules` — Create a repo with a submodule.
  Setup: init a "library" repo with a commit, then in the main repo use
  `repo.submodule(url, path, use_gitlink)` (or shell out to `git submodule add
  <library-path> lib`) to register the submodule, then commit. Run
  `gityard worktrees --all`, assert the submodule directory does not appear as a
  worktree entry. The test codifies that `repo.worktrees()` only returns linked
  worktrees, not submodule directories. If `git2`'s submodule API proves too
  cumbersome, use `std::process::Command` to shell out to `git submodule add`.

#### 7. Bare repo registration test

**File**: `tests/cli.rs`

- [x] `test_add_rejects_bare_repo` — Create a bare repo with
  `git2::Repository::init_bare`, attempt `gityard add <path>`, assert stderr
  contains "not a git repository" and the repo is NOT registered. This codifies
  the registration gate at `registry.rs:79` (bare repos lack a `.git`
  directory — they have the git internals at the top level).
  > **Deviation:** The `add` command exits 0 even when individual paths fail
  > (multi-path design). Changed from `.assert().failure()` to
  > `.assert().success()` plus verifying the repo was not registered via
  > `gityard list`.

### Tests

Tests are inline with each change section above. Summary:

- **Symlink fix verification**: The simplified assertion in
  `test_list_worktrees_with_linked_worktree` directly tests that
  `WorktreeInfo.path` is pre-canonicalized
- **CLI worktrees tests**: 5 new tests covering list, dirty filter, gone filter,
  `--all` flag, and empty-output behavior
- **CLI trim-with-worktree tests**: 1 new helper + 3 new tests covering clean
  worktree removal, dirty worktree protection, and dry-run
- **Edge case tests**: 2 new tests for submodule and bare repo behavior

### Success Criteria

#### Automated Verification

- [x] `just check` passes (format + lint + build + test + license check)
- [x] `cargo test test_list_worktrees_with_linked_worktree` passes with the
  simplified assertion (confirms the symlink fix works)
- [x] All new CLI tests pass: `cargo test --test cli test_worktrees`
- [x] All new CLI tests pass: `cargo test --test cli test_trim_with`
- [x] All new CLI tests pass: `cargo test --test cli test_add_rejects_bare`
- [x] `cargo clippy --all-targets` passes (no new warnings)

## References

- Idea file: `thoughts/ideas/2026-04-13-gityard-worktree-test-coverage-and-symlink-fix.md`
- `src/worktrees.rs` — worktree listing and `WorktreeInfo` construction
- `src/trimmer.rs:240-295` — `build_worktree_branch_map` and `WorktreeState`
- `src/presenter.rs:284-293` — `display_path` function
- `src/registry.rs:73-81` — registration gate and path canonicalization convention
- `tests/cli.rs` — existing CLI integration test patterns and helpers
- `TODO.md` items #2, #3, #4
