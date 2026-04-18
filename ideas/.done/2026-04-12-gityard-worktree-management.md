# gityard: Worktree Management Across Repositories

## Problem Statement

When working with multiple git worktrees across several repositories, there's no centralized way to see what worktrees exist, what state they're in, or clean up stale ones. Each repo's worktrees must be inspected individually with `git worktree list`, and cleaning up a worktree whose tracking branch is gone requires manually removing the worktree, then deleting the branch — across potentially many repos.

Since gityard already provides bulk visibility into repo state (`status`) and branch state (`branches`), and already handles bulk branch cleanup (`trim`), worktree management is a natural extension of its multi-repo oversight role.

## Motivation

- **Visibility gap**: `gityard status` shows per-repo state and `gityard branches` shows per-branch state, but neither surfaces worktree information. Worktrees are a third axis of repo state that's currently invisible to gityard.
- **Stale worktree accumulation**: Worktrees created for feature branches tend to linger after the branch is merged and the remote tracking ref is pruned. Without bulk cleanup, they accumulate disk usage and clutter.
- **Workflow friction**: Checking worktree state across many repos requires running `git worktree list` in each repo individually. This is exactly the kind of repetitive cross-repo operation gityard exists to eliminate.
- **Trim completeness**: The existing `gityard trim` command deletes gone branches but doesn't account for worktrees checked out on those branches. If a gone branch has a worktree, the branch deletion will fail (or leave an orphaned worktree). Trim should handle the full cleanup lifecycle.

## Context

### Current gityard Commands (Relevant)

- **`gityard status`**: One row per repo. Shows branch, dirty/clean, staged/modified/untracked counts, ahead/behind main, ahead/behind remote. Parallel inspection via rayon + git2.
- **`gityard branches`**: One row per branch per repo. Shows tracking state (tracked/gone/untracked), ahead/behind main, last commit date, merged status. Parallel via rayon + git2.
- **`gityard trim`**: Deletes branches matching `--gone` or `--no-remote` filters. Has `--dry-run` mode. Protects HEAD branch and uses safe checkout resolution. Does not currently consider worktrees.

### Git Worktree Model

- A repo can have multiple worktrees, each with its own working directory, index, and HEAD.
- Worktrees share the object store, refs, config, and hooks.
- A branch can only be checked out in one worktree at a time — git enforces this.
- `git worktree list` shows path, HEAD commit, and branch for each worktree.
- `git worktree remove <path>` removes a worktree (fails if dirty unless `--force`).
- `git worktree prune` cleans up stale worktree metadata for worktrees whose paths no longer exist on disk.

### git2 Crate Support

- `Repository::worktrees()` returns a `StringArray` of worktree names (excludes the main working tree).
- `Repository::find_worktree(name)` returns a `Worktree` with path and validation.
- `Worktree::validate()` checks if the worktree path still exists on disk.
- Opening a `Repository` at a worktree path gives access to its HEAD, status, etc. — same APIs used by the existing inspector.

## Goals / Success Criteria

- [ ] `gityard worktrees` lists all worktrees across registered repos, showing per-worktree detail (path, branch, dirty/clean, ahead/behind) in the same tabular style as `status` and `branches`
- [ ] Repos with no additional worktrees (only the main working tree) are excluded by default, with a flag to include them
- [ ] `gityard trim` handles worktrees on gone branches: removes clean worktrees before deleting the branch, skips and warns for dirty worktrees
- [ ] Existing `trim` behavior is unchanged for branches that don't have worktrees
- [ ] `--dry-run` on `trim` reports worktrees that would be removed alongside branches
- [ ] Filtering flags on `worktrees` (e.g., `--gone`, `--dirty`) follow the same pattern as `status` and `branches` filters

## Non-Goals (Out of Scope)

- **Worktree creation**: `gityard` is not a worktree lifecycle manager — creating worktrees remains a per-repo `git worktree add` operation
- **Worktree-based bulk operations**: No `gityard fetch` or `gityard pull` targeting worktrees specifically — the existing commands operate on repos, not worktrees
- **Cross-repo worktree coordination**: No awareness of relationships between worktrees in different repos (e.g., "these three worktrees are all for the same feature")
- **Interactive worktree removal**: No TUI picker for selecting worktrees to remove — `trim` handles gone-branch worktrees automatically, individual removal is `git worktree remove`

## Proposed Approach

### `gityard worktrees` Command

A new top-level command (like `status` and `branches`) that lists worktrees across all registered repos.

**Output columns** (per-worktree row):
- **REPO**: Repository alias (shown on first worktree row per repo, blank on subsequent — same pattern as `branches`)
- **WORKTREE**: Worktree name or path (relative to repo root, or absolute if outside)
- **BRANCH**: Branch checked out in the worktree
- **STATUS**: Clean/dirty with counts (same format as `status` command)
- **REMOTE**: Tracking state of the branch (tracked/gone/untracked — same as `branches`)
- **LAST COMMIT**: Date of the branch tip commit

**Filtering flags**:
- `--gone`: Only show worktrees whose branch has a gone upstream
- `--dirty`: Only show worktrees with uncommitted changes
- `--group`: Filter to repos in a specific group (shared pattern)

**Default behavior**: Only show repos that have at least one linked worktree (beyond the main working tree). A `--all` flag includes repos with no extra worktrees (showing the main working tree as the sole entry).

### `gityard trim` Enhancement

Extend the existing trim command to handle worktrees as part of gone-branch cleanup:

1. When a gone branch is selected for trimming, check if it has a worktree checked out
2. If the worktree is **clean** (no uncommitted changes): remove the worktree first, then delete the branch
3. If the worktree is **dirty**: skip both the worktree removal and the branch deletion, emit a warning with the worktree path and dirty file summary
4. `--dry-run` reports which worktrees would be removed alongside their branches

This preserves the existing safety model — trim already skips HEAD branches and uses safe checkout. Adding worktree-dirty checks is a natural extension of that safety-first approach.

## Constraints

- Must use `git2` for worktree operations (no shelling out to `git worktree`) — consistent with the rest of gityard
- Worktree removal via `git2` may require careful handling — `git2` has `Worktree::prune()` but the ergonomics around force/dirty checks need investigation
- The main working tree is not a "worktree" in git2's model (`Repository::worktrees()` excludes it) — listing it requires special handling if `--all` is used
- Worktree paths can be anywhere on disk (not necessarily inside the repo) — path display needs to handle both relative and absolute cases sensibly

## Resolved Questions

### git2 dirty-checking for worktree prune

`git2::Worktree::prune()` and `WorktreePruneOptions` have **no dirty-awareness**. The three available flags (`valid`, `locked`, `working_tree`) control only structural/administrative pruning. gityard must implement dirty detection separately:

1. `worktree.validate()` — check if the worktree path still exists on disk
2. `Repository::open(worktree.path())` → `repo.statuses(None)` — check for uncommitted changes (same pattern as `inspector::count_statuses()`)
3. If clean: `worktree.prune()` with `.working_tree(true)` to remove both metadata and directory
4. If dirty: skip and warn

The `is_prunable()` companion method checks administrative prunability (invalid, unlocked) but not working-tree cleanliness.

### Worktree path display

**Relative to repo when possible, absolute otherwise.** If the worktree path is inside or a sibling of the repo directory, display it relative to the repo root (e.g., `.worktrees/feature-branch` or `../my-feature/`). If the worktree is elsewhere on the filesystem, display the absolute path. This handles the common `.worktrees/` convention while gracefully supporting scattered layouts.

### Prunable (missing) worktrees

**Show them in the listing with a "missing" marker.** Worktrees whose directory no longer exists on disk (detectable via `worktree.validate()` failure) appear in `gityard worktrees` output with a visual indicator in the STATUS column (e.g., "missing" in red). The `trim` command can also prune these — they're safe to remove since there's no working directory to lose data from.

### Main working tree under `--all`

**Use the repo path.** Consistent with `git worktree list` output. The main working tree is represented by its filesystem path (relative or absolute, same rules as linked worktrees), not a synthetic label like "(main)". The absence of a worktree name naturally distinguishes it from linked worktrees.

## References

- Git worktree documentation: https://git-scm.com/docs/git-worktree
- git2 crate `Worktree` type: https://docs.rs/git2/latest/git2/struct.Worktree.html
- git2 crate `Repository::worktrees()`: https://docs.rs/git2/latest/git2/struct.Repository.html#method.worktrees
- Existing gityard source: `/Users/jasonr/Workspace/jasnross/gityard/src/`
- Related idea: [Parallel Plan Implementation with Worktrees](2026-03-05-parallel-plan-implementation-with-worktrees.md) — different concept (Claude Code workflow) but shares worktree domain knowledge

## Affected Systems

- `gityard` CLI: new `worktrees` command, enhanced `trim` command
- Modules likely affected: `cli.rs` (new command definition), `main.rs` (new handler), new worktree inspector module, `presenter.rs` (new table formatter), `filter.rs` (new filter struct), `trimmer.rs` (worktree-aware cleanup)
