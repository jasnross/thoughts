# Auto-switch HEAD off gone branches during `gityard trim`

## Problem Statement

When running `gityard trim`, if HEAD is pointing to a branch whose remote-tracking reference has been pruned ("gone"), that branch is marked as `Protected` with reason `"HEAD"` and silently skipped. The user sees output like `─ myrepo: protected feature-gone (HEAD)` but receives no guidance on what to do.

This means the gone branch survives every future `trim` run until the user manually switches to another branch. In a multi-repo workflow — gityard's primary use case — this creates a recurring papercut: the user has to notice the protected output, remember which repo it applies to, navigate there, switch branches, and re-run trim.

## Motivation

- **Ergonomics**: `trim` is meant to be a fire-and-forget cleanup command. A gone branch stuck on HEAD undermines that by requiring manual follow-up.
- **Multi-repo friction**: With many repos, it's easy to miss the protected output for one repo among dozens. The stale branch lingers indefinitely.
- **User expectation**: If the remote-tracking ref is gone, the user almost certainly wants the local branch cleaned up too. The only reason it isn't deleted is a mechanical limitation (can't delete the checked-out branch), not user intent.
- **Architectural alignment**: The `trim` command is the only mutation command that interleaves data collection and action in a single loop. Every other command — `status`, `branches`, `fetch`, `pull`, `push`, `checkout` — separates collection from action. This feature is a natural forcing function to bring `trim` in line with the established pattern.

## Context

### Current implementation

- `trimmer.rs:80-89`: The `is_head()` check runs after candidacy filtering but before deletion. Gone branches on HEAD are pushed as `BranchTrimResult` with `TrimOutcome::Protected { reason: "HEAD" }` and skipped via `continue`.
- `presenter.rs:362-369`: Protected outcomes are displayed with a yellow dash prefix and the reason in parentheses.
- `branches.rs:144-165`: `detect_tracking_state()` classifies branches as `Gone`, `Tracking`, or `Untracked` by checking upstream config and resolving the remote-tracking ref.
- The trim command already identifies the main branch for each repo (to protect it from deletion), so the target branch for switching is already known.

### Architectural gap in `trim`

The `trim` command is the only mutation command that interleaves collection and action in a single loop (`trimmer.rs:58-123`). All other commands follow a phased approach:

- **Read-only commands** (`status`, `branches`): collect state into structs (`RepoState`, `BranchInfo`), then present.
- **Mutation commands** (`fetch`, `pull`, `push`, `checkout`): collect state (`RepoState`), run preflight validation (`PreflightResult`), then execute.

The trimmer independently re-derives branch state (tracking via `detect_tracking_state()`, HEAD via `branch.is_head()`, main-branch membership via `main_branches.contains()`) that `BranchInfo` in `branches.rs:20-38` already models. Splitting `trim_repo` into collect-then-act would:

- Align with the codebase's established architecture.
- Enable the auto-switch feature cleanly: assess all branches first, detect and resolve HEAD-is-gone before acting, then iterate normally.
- Improve testability: decision logic becomes a pure function over collected state, testable without real git repos.
- Support future features like preview-before-act or richer dry-run output.

### How git handles this

Git itself refuses to delete the checked-out branch (`git branch -d` errors with "Cannot delete branch 'X' checked out at..."). The standard manual workflow is `git checkout main && git branch -d feature-gone`.

## Goals / Success Criteria

- [ ] `trim_repo` is refactored into collect-then-act phases, consistent with the rest of the codebase
- [ ] When HEAD is on a gone branch, `trim` automatically switches to the repo's main branch before deleting the gone branch
- [ ] The switch and subsequent deletion are reported clearly in trim output
- [ ] If the switch fails (e.g. uncommitted changes, merge conflicts), the branch is protected as today with a clear warning explaining why
- [ ] `--dry-run` reports that a switch *would* happen without actually performing it
- [ ] No new CLI flags required — this is the default behavior
- [ ] Existing trim tests continue to pass; new tests cover the auto-switch and switch-failure paths

## Non-Goals (Out of Scope)

- **Smart branch detection**: No reflog analysis or "most recently used" branch heuristics. Always switch to the repo's main branch.
- **Interactive confirmation**: No prompts or confirmation dialogs. Gityard is non-interactive by design, and the user already opted into cleanup by running `trim`.
- **Handling detached HEAD**: If HEAD is detached (not on any branch), current behavior is fine — no branch to protect or switch away from.
- **Switching for non-gone protected branches**: The main branch is also protected; this change only applies to the HEAD-is-gone case.

## Proposed Approach

### Phase 1: Refactor `trim_repo` to collect-then-act

Split the current single-loop `trim_repo()` into two phases, aligning it with the pattern used by every other command in the codebase:

1. **Collect**: Iterate all local branches and gather their state (name, tracking state, is_head, main-branch membership) into a vec of assessment structs. No mutations.
2. **Act**: Iterate the collected assessments and perform deletions based on the already-computed decisions.

This refactor is a prerequisite for the auto-switch feature but has independent value: it aligns `trim` with the established architecture, improves testability, and makes dry-run reporting trivially accurate.

### Phase 2: Auto-switch HEAD off gone branches

With all branch states collected upfront, the auto-switch slots in naturally between collection and action:

1. After collecting all branch states, check if HEAD is on a gone branch.
2. If so, attempt to switch HEAD to the repo's main branch (via libgit2, no shell-out).
3. **If the switch succeeds**: Mark the branch as "was HEAD, switched to main" in the collected state. The action phase deletes it normally. Output uses a single line: `✓ myrepo: deleted feature-gone (gone, was HEAD → main)`.
4. **If the switch fails** (e.g. dirty worktree conflicts): Mark the branch as protected with a clear warning: `✗ myrepo: cannot trim feature-gone (HEAD, dirty worktree prevents switch to main)`.
5. **In dry-run mode**: Report that a switch would be needed but do not perform it.
6. **No auto-stash**: If uncommitted changes prevent the switch, fail gracefully. The user can stash manually and re-run trim. Auto-stashing adds risk (stash pop conflicts, forgotten stashes) for a rare edge case.

## Constraints

- The switch must use libgit2 (via git2 crate) rather than shelling out to `git`, consistent with how gityard performs all other git operations.
- The main branch must already be known/detected — no new heuristics for determining which branch is "main".
- The operation must be safe with a dirty working tree: if there are uncommitted changes that would conflict with the switch, fail gracefully rather than losing work.

## Resolved Questions

- **Output format**: Single line per branch — `✓ myrepo: deleted feature-gone (gone, was HEAD → main)`. Keeps output compact and counts the branch once in the summary. The switch is a means to an end, not a separate outcome.
- **Stash behavior**: No auto-stash. If the worktree is dirty and conflicts with the switch, protect the branch and warn. The user can stash manually and re-run. Auto-stashing adds risk for a rare edge case.
- **Order of operations**: Switch before the branch iteration loop. The gone branch is no longer HEAD when encountered, so it flows through the existing deletion path without special-case logic mid-loop.

## References

- Current HEAD protection: `trimmer.rs:80-89`
- Trim single-loop implementation: `trimmer.rs:42-129`
- Tracking state detection: `branches.rs:144-165`
- `BranchInfo` struct (existing collect model): `branches.rs:20-38`
- Trim output formatting: `presenter.rs:322-409`
- CLI args: `cli.rs:148-160`
- Command wiring: `main.rs:356-393`
- Executor collect-validate-act pattern: `executor.rs:29-110`
- Preflight validation pattern: `operation.rs:43-86`
- Inspector collect pattern: `inspector.rs:46-96`
