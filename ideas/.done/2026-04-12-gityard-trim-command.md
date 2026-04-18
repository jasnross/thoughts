# gityard trim: Bulk Branch Cleanup Across Repositories

## Problem Statement

When working across many repositories, local branches accumulate over time as
remote branches are merged and deleted. Cleaning these up today requires
cd-ing into each repository and running custom scripts (`git-delete-gone`,
`git-delete-noremote`) one repo at a time. This is tedious and easy to forget,
leading to cluttered branch lists that make it harder to find active work.

gityard already provides bulk operations (`fetch`, `pull`, `push`, `checkout`)
across registered repositories, but has no equivalent for branch cleanup.

## Motivation

- **Reduce manual effort**: Cleaning up stale branches across 10+ repos is a
  recurring chore that should be a single command.
- **Consistency with existing workflow**: gityard already handles bulk git
  operations with pre-flight validation — branch cleanup fits naturally.
- **Better visibility**: The `branches` command shows branch info across repos
  but doesn't currently highlight branches that are stale candidates for
  deletion (gone upstream or no remote tracking).

## Context

### Existing single-repo tools

Two custom shell scripts handle this today for individual repositories:

- **`git-delete-gone`**: Fetches with `--prune`, finds branches whose upstream
  shows `[: gone]` in `git branch -vv`, and force-deletes them.
- **`git-delete-noremote`**: Fetches with `--prune`, finds branches with no
  upstream tracking branch at all, and force-deletes them.

### Two distinct branch states

1. **"Gone" branches**: A local branch that *has* an upstream configured, but
   that remote branch no longer exists (was merged and deleted on the remote).
   These are almost always safe to delete.

2. **"No remote" branches**: A local branch with *no* upstream tracking branch
   configured. These may be local-only work branches that were never pushed, so
   they require more caution.

### gityard's current capabilities

- `BranchInfo` in `branches.rs` already tracks `upstream: Option<String>` per
  branch, which can detect "no remote" branches (`upstream` is `None`).
- Detecting "gone" branches requires checking whether the configured upstream
  ref still exists — this is a small addition to the existing branch inspection
  logic.
- The `--group` scoping pattern is well-established across commands.
- The executor module provides a pattern for parallel operations with
  pre-flight validation and structured reporting.

## Goals / Success Criteria

- [ ] `gityard trim` deletes "gone" branches across all (or scoped) repositories
- [ ] `--no-remote` flag extends deletion to branches with no upstream tracking
- [ ] `--dry-run` flag shows what would be deleted without deleting anything
- [ ] `--group` scoping works consistently with other commands
- [ ] Current HEAD branch and configured main branches are never deleted
- [ ] Output clearly reports per-repo results: what was deleted, what failed,
      what was skipped
- [ ] `gityard branches` gains visual indicators for gone/no-remote branch states
- [ ] `gityard branches` gains filter flags (`--gone`, `--no-remote`) to show
      only branches in those states

## Non-Goals (Out of Scope)

- **No auto-fetch**: `trim` operates on current local state. Users run
  `gityard fetch` first if they want up-to-date remote state. This keeps the
  command fast, offline, and consistent with how other gityard commands work.
- **No interactive confirmation**: Deletes immediately, matching the behavior of
  the existing `git-delete-gone` script. Users preview with `--dry-run` first.
- **No "merged into main" detection as a deletion criterion**: The `branches`
  command already shows `is_merged` status, but `trim` focuses on
  remote-tracking state, not merge state. These are orthogonal — a branch can
  be merged but still have a live remote, or be gone but not merged.
- **No remote branch deletion**: Only local branches are affected.

## Proposed Behavior

### `gityard trim`

```
gityard trim [--group <name>] [--no-remote] [--dry-run]
```

**Default (no flags)**: Delete branches whose upstream is "gone" (remote branch
was deleted).

**`--no-remote`**: Also delete branches with no upstream tracking branch
configured. This is additive — gone branches are always included.

**`--dry-run`**: Print what would be deleted without deleting anything.

**Safety rules** (always enforced):
- Never delete the current HEAD branch
- Never delete configured main branches (from `config.defaults.main_branches`)

**Output format**: Per-repo results grouped by repository, similar to other
gityard command output. Each branch shows its state (gone/no-remote) and whether
it was deleted, skipped, or failed.

**Exit code**: Non-zero (1) if any branch deletion fails. Matches Unix
conventions and the existing `git-delete-gone` behavior. The structured output
still reports exactly what happened per branch.

### `gityard branches` enhancements

- Add a new **REMOTE** column showing the upstream tracking branch name (e.g.,
  `origin/feature/login`), `gone` (highlighted) when the upstream was deleted,
  or `─` when no upstream is configured.
- Add `--gone` filter flag: show only branches whose upstream is gone.
- Add `--no-remote` filter flag: show only branches with no upstream.
- Flags can be combined (OR semantics, matching `status` command's filter
  pattern).

## Constraints

- Must use `git2` (libgit2) for branch inspection, consistent with the rest of
  gityard — no shelling out to `git branch -vv`.
- Detecting "gone" state via libgit2 requires checking whether the configured
  upstream ref resolves to a valid reference. If a branch has
  `branch.<name>.remote` and `branch.<name>.merge` configured but the
  remote-tracking ref doesn't exist, the upstream is "gone."
- Branch deletion via `git2` uses `Branch::delete()`, which requires the branch
  to not be checked out (HEAD).

## Resolved Questions

- **Flag design**: Additive flag (`--no-remote`) rather than an enum
  `--target` flag. Simpler, consistent with gityard's existing OR-semantics
  filter pattern. If more states are added later, they become additional flags.
- **Exit code**: Non-zero (1) on any deletion failure. Matches `git-delete-gone`
  and Unix conventions.
- **Branches column**: New dedicated REMOTE column showing upstream name, `gone`,
  or `─`. Explicit and scannable, consistent with the table's existing style.

## References

- Existing scripts: `dotfiles/scripts/git-delete-gone`,
  `dotfiles/scripts/git-list-gone`, `dotfiles/scripts/git-delete-noremote`,
  `dotfiles/scripts/git-list-noremote`
- gityard branch inspection: `src/branches.rs` (`BranchInfo` struct,
  `list_branches()`)
- gityard executor pattern: `src/executor.rs` (parallel execution with
  pre-flight validation)
- gityard filter pattern: `src/filter.rs` (OR-semantics flag filtering)

## Affected Systems

- **`src/branches.rs`**: Add "gone" detection to `BranchInfo`
- **`src/cli.rs`**: New `trim` command and args; new filter flags on `branches`
- **`src/main.rs`**: Command dispatch for `trim`; updated `branches` handling
- **`src/presenter.rs`**: Output formatting for trim results and branches
  tracking state
- New module (e.g., `src/trimmer.rs`): Branch deletion logic with safety checks
