# Auto-switch HEAD off gone branches during `gityard trim` — Implementation Plan

## Overview

Refactor `trim_repo` into a collect-then-act architecture (aligning it with every other gityard command), then add auto-switching HEAD off gone branches between the two phases. Two independently shippable phases: the refactor changes no behavior, the auto-switch builds on the new structure.

## Current State Analysis

`trim_repo()` (`trimmer.rs:42-129`) is a single loop that interleaves state detection and mutation. For each branch it: detects tracking state, evaluates candidacy, checks protection (HEAD/main), and deletes — all in one iteration. This makes it impossible to reason about the full set of branches before acting.

When HEAD points to a gone branch, it's marked `Protected { reason: "HEAD" }` and skipped (`trimmer.rs:80-89`). The user sees `─ myrepo: protected feature-gone (HEAD)` but gets no guidance. The branch survives every future `trim` run.

### Key Discoveries:

- `trim_repo` already has a `git2::Repository` open at `trimmer.rs:48` — libgit2 checkout APIs are available without architectural changes
- `branch.delete()` at `trimmer.rs:112` already uses libgit2 for writes, so using libgit2 for the checkout is consistent with trim's existing pattern (unlike `cmd_exec` commands which shell out to git)
- `BranchInfo` (`branches.rs:20-38`) has the fields trim needs for decisions (`name`, `tracking`, `is_head`) but also computes expensive fields trim doesn't need (`ahead_main`, `behind_main`, `last_commit_epoch`, `is_merged`). A trim-specific struct is appropriate.
- `main_branches` comes from `config.defaults.main_branches` (`main.rs:365`) — a `Vec<String>` like `["main", "master"]`. The trim function receives it as `&[String]`.
- Test code at `branches.rs:263-265` uses `set_head()` + `checkout_head(.force())` for test fixtures. For production use, the plan uses a different approach: `checkout_tree(.safe())` first (updates worktree), then `set_head()` (updates HEAD ref). `GIT_CHECKOUT_SAFE` refuses to overwrite worktree files that differ from the current HEAD tree, protecting against losing uncommitted worktree modifications. It does *not* protect staged-only changes (where the worktree matches the index but the index differs from HEAD). If the checkout fails, HEAD hasn't moved and the repo is left in its original state.

## Desired End State

After both phases are complete:

- `trim_repo` follows the collect-then-act pattern used by all other gityard commands
- Running `gityard trim` when HEAD is on a gone branch automatically switches to the repo's main branch, then deletes the gone branch
- Output: `✓ myrepo: deleted feature-gone (gone, was HEAD → main)`
- If the switch fails (dirty worktree): `─ myrepo: protected feature-gone (HEAD, switch to main failed: <reason>)`
- `--dry-run` reports what would happen without acting
- No new CLI flags

### Verification:

```
just check
```

All existing tests pass, plus new tests covering: auto-switch success, switch failure on dirty worktree, dry-run reporting, and the collect-then-act boundary.

## What We're NOT Doing

- Not reusing `BranchInfo` — it computes expensive fields trim doesn't need
- Not adding interactive confirmation or new CLI flags
- Not handling detached HEAD (no branch to protect or switch from)
- Not auto-stashing uncommitted changes before switching
- Not adding smart branch detection (reflog, MRU) — always switch to main
- Not extracting shared test utilities across modules (each module already has its own copies of `entry_for`/`default_main_branches`)
- Not detecting dirty-worktree conflicts in dry-run mode — `--dry-run` reports `WouldDeleteAfterSwitch` based on whether a main branch exists, but cannot predict whether the actual checkout would fail due to uncommitted changes. This is an inherent limitation: detecting conflicts requires attempting the checkout. Documenting rather than working around.

## Implementation Approach

Phase 1 introduces a `TrimCandidate` struct and splits the single loop into collect + act. Phase 2 adds a HEAD resolution step between them that performs the auto-switch via libgit2. Two new `TrimOutcome` variants (`DeletedAfterSwitch`, `WouldDeleteAfterSwitch`) carry switch context to the presenter.

---

## Phase 1: Refactor `trim_repo` to collect-then-act

### Overview

Split the single loop in `trim_repo()` into two passes: a pure collection pass that produces `Vec<TrimCandidate>`, and an action pass that executes decisions. No behavioral change — this is a pure refactor.

### Changes Required:

#### 1. New `TrimCandidate` struct and collection function

**File**: `src/trimmer.rs`

- [x] Add `TrimCandidate` struct after the existing `TrimOutcome` enum (around line 17):

```rust
/// Pre-computed trim decision for a single branch.
#[derive(Debug)]
struct TrimCandidate {
    /// Branch name.
    name: String,
    /// Remote-tracking state.
    tracking: TrackingState,
    /// What to do with this branch.
    action: TrimAction,
}

/// Decision for a trim candidate, computed during collection.
#[derive(Debug)]
enum TrimAction {
    /// Branch should be deleted.
    Delete,
    /// Branch is protected and should be skipped.
    Protect { reason: String },
}
```

Both types are crate-private (no `pub`) — they're internal to the trimmer's two-phase flow.

- [x] Extract the collection logic from `trim_repo` into a new `assess_branches` function:

```rust
/// Assess all local branches and return trim candidates with pre-computed decisions.
///
/// Only branches that match the trim criteria (gone, or untracked when
/// `include_no_remote` is true) are returned. Each candidate carries a
/// `TrimAction` indicating whether it should be deleted or protected.
fn assess_branches(
    repo: &git2::Repository,
    main_branches: &[String],
    include_no_remote: bool,
) -> Result<Vec<TrimCandidate>, String> {
    let git_branches = repo
        .branches(Some(git2::BranchType::Local))
        .map_err(|e| format!("failed to list branches: {e}"))?;

    let mut candidates = Vec::new();

    for item in git_branches {
        let (branch, _) = item.map_err(|e| format!("failed to read branch: {e}"))?;

        let Some(name) = branch.name().ok().flatten().map(String::from) else {
            continue;
        };

        let tracking = branches::detect_tracking_state(repo, &branch);

        let is_trim_candidate = match &tracking {
            TrackingState::Gone { .. } => true,
            TrackingState::Untracked => include_no_remote,
            TrackingState::Tracking { .. } => false,
        };
        if !is_trim_candidate {
            continue;
        }

        let action = if branch.is_head() {
            TrimAction::Protect {
                reason: "HEAD".to_string(),
            }
        } else if main_branches.contains(&name) {
            TrimAction::Protect {
                reason: "main branch".to_string(),
            }
        } else {
            TrimAction::Delete
        };

        candidates.push(TrimCandidate {
            name,
            tracking,
            action,
        });
    }

    Ok(candidates)
}
```

#### 2. Rewrite `trim_repo` to use the two-phase flow

**File**: `src/trimmer.rs`

- [x] Replace the body of `trim_repo` (lines 47–128) with:

```rust
pub fn trim_repo(
    entry: &RepoEntry,
    main_branches: &[String],
    include_no_remote: bool,
    dry_run: bool,
) -> Result<RepoTrimResult, String> {
    let repo = git2::Repository::open(&entry.path)
        .map_err(|e| format!("failed to open {}: {e}", entry.path.display()))?;

    let alias = entry.display_name().to_string();
    let candidates = assess_branches(&repo, main_branches, include_no_remote)?;

    let mut results = Vec::new();

    for candidate in &candidates {
        let outcome = match &candidate.action {
            TrimAction::Protect { reason } => TrimOutcome::Protected {
                reason: reason.clone(),
            },
            TrimAction::Delete => {
                if dry_run {
                    TrimOutcome::WouldDelete
                } else {
                    match repo.find_branch(&candidate.name, git2::BranchType::Local) {
                        Ok(mut branch) => match branch.delete() {
                            Ok(()) => TrimOutcome::Deleted,
                            Err(e) => TrimOutcome::Failed {
                                reason: e.to_string(),
                            },
                        },
                        Err(e) => TrimOutcome::Failed {
                            reason: format!("branch lookup failed: {e}"),
                        },
                    }
                }
            }
        };

        results.push(BranchTrimResult {
            name: candidate.name.clone(),
            tracking: candidate.tracking.clone(),
            outcome,
        });
    }

    Ok(RepoTrimResult {
        alias,
        branches: results,
    })
}
```

Note: `TrackingState` already derives `Clone` (`branches.rs:8`), so `candidate.tracking.clone()` works. Phase 1 uses `let candidates = ...` (immutable). Phase 2 will change this to `let mut candidates` to support `resolve_head(&mut candidates, ...)`.

#### Tests for This Phase

- [x] Run `just check` — all six existing trim tests must pass unchanged. The refactor preserves exact behavior: same outcomes, same ordering, same exit codes.
- [x] Verify that `cargo clippy --all-targets` passes (the new code must satisfy pedantic + restriction lints)

### Success Criteria:

#### Automated Verification:

- [x] `just check` passes (format + lint + build + test + license check)
  > **Deviation:** `just check` runs `cargo fix` which requires a clean working tree. Ran the individual checks separately: `cargo fmt --check`, `cargo clippy --all-targets`, `cargo test`, `cargo deny check licenses` — all pass.
- [x] All six existing trim tests pass: `cargo test --lib trimmer`
- [x] CLI integration tests pass: `cargo test --test cli`

---

## Phase 2: Auto-switch HEAD off gone branches

### Overview

Add a HEAD resolution step between `assess_branches` and the action loop. When a gone branch is on HEAD, attempt to switch to the repo's main branch via libgit2 before the action loop runs.

### Changes Required:

#### 1. Add new `TrimOutcome` variants

**File**: `src/trimmer.rs`

- [x] Add two variants to `TrimOutcome` (after `Failed`):

```rust
pub enum TrimOutcome {
    Deleted,
    WouldDelete,
    Protected { reason: String },
    Failed { reason: String },
    /// Branch was deleted after switching HEAD to another branch.
    DeletedAfterSwitch { switched_to: String },
    /// Branch would be deleted after switching HEAD (dry-run mode).
    WouldDeleteAfterSwitch { switched_to: String },
}
```

#### 2. Add `HeadResolution` struct and `resolve_head` function

**File**: `src/trimmer.rs`

- [x] Add the `HeadResolution` struct to track what happened during the switch:

```rust
/// Result of attempting to resolve a gone-HEAD situation.
struct HeadResolution {
    /// The branch that was on HEAD (the gone branch).
    head_branch: String,
    /// The branch we switched to (or would switch to in dry-run).
    switched_to: String,
}
```

- [x] Add the `resolve_head` function. This finds a gone-HEAD candidate, attempts the switch, and mutates the candidate's action accordingly:

```rust
/// If HEAD is on a gone trim candidate, switch to the repo's main branch.
///
/// On success, changes the candidate's action from `Protect` to `Delete` and
/// returns a `HeadResolution` identifying both branches. On failure, updates
/// the protection reason to explain why the switch failed.
///
/// In dry-run mode, identifies the switch target without performing it.
fn resolve_head(
    repo: &git2::Repository,
    candidates: &mut [TrimCandidate],
    main_branches: &[String],
    dry_run: bool,
) -> Option<HeadResolution> {
    // Find the gone-HEAD candidate, if any.
    let head_idx = candidates.iter().position(|c| {
        matches!(c.action, TrimAction::Protect { ref reason } if reason == "HEAD")
            && matches!(c.tracking, TrackingState::Gone { .. })
    })?;

    let head_branch = candidates[head_idx].name.clone();

    // Find which main branch exists in this repo, excluding the gone HEAD
    // branch itself (it could be in main_branches if e.g. "main" went gone).
    let switch_target = main_branches.iter().find_map(|name| {
        if *name == head_branch {
            return None;
        }
        repo.find_branch(name, git2::BranchType::Local)
            .ok()
            .map(|_| name.clone())
    });

    let Some(target) = switch_target else {
        candidates[head_idx].action = TrimAction::Protect {
            reason: "HEAD; no main branch found to switch to".to_string(),
        };
        return None;
    };

    if dry_run {
        candidates[head_idx].action = TrimAction::Delete;
        return Some(HeadResolution {
            head_branch,
            switched_to: target,
        });
    }

    // Attempt the checkout: update worktree first, then move HEAD.
    // Safe mode refuses to overwrite worktree files modified since HEAD,
    // so uncommitted worktree changes cause a checkout error (not data loss).
    let main_ref = format!("refs/heads/{target}");
    let switch_result = (|| -> Result<(), git2::Error> {
        let obj = repo
            .find_reference(&main_ref)?
            .peel(git2::ObjectType::Commit)?;
        let commit = obj
            .as_commit()
            .ok_or_else(|| git2::Error::from_str("not a commit"))?;
        let tree = commit.tree()?;
        repo.checkout_tree(
            tree.as_object(),
            Some(git2::build::CheckoutBuilder::new().safe()),
        )?;
        repo.set_head(&main_ref)?;
        Ok(())
    })();

    match switch_result {
        Ok(()) => {
            candidates[head_idx].action = TrimAction::Delete;
            Some(HeadResolution {
                head_branch,
                switched_to: target,
            })
        }
        Err(e) => {
            candidates[head_idx].action = TrimAction::Protect {
                reason: format!("HEAD; switch to {target} failed: {e}"),
            };
            None
        }
    }
}
```

#### 3. Update `trim_repo` to call `resolve_head` and use `HeadResolution`

**File**: `src/trimmer.rs`

- [x] Insert the HEAD resolution call between `assess_branches` and the action loop. Extract a `delete_branch` helper to avoid duplicating the `find_branch`/`delete` sequence:

```rust
pub fn trim_repo(
    entry: &RepoEntry,
    main_branches: &[String],
    include_no_remote: bool,
    dry_run: bool,
) -> Result<RepoTrimResult, String> {
    let repo = git2::Repository::open(&entry.path)
        .map_err(|e| format!("failed to open {}: {e}", entry.path.display()))?;

    let alias = entry.display_name().to_string();
    let mut candidates = assess_branches(&repo, main_branches, include_no_remote)?;

    // If HEAD is on a gone branch, try to switch away before acting.
    let head_resolution = resolve_head(&repo, &mut candidates, main_branches, dry_run);

    // Extract switch info upfront to avoid `.expect()` in the loop
    // (clippy::expect_used is denied in non-test code).
    let switched_head_name = head_resolution.as_ref().map(|hr| hr.head_branch.as_str());
    let switched_to_name = head_resolution.as_ref().map(|hr| hr.switched_to.as_str());

    let delete_branch = |name: &str| -> Result<(), String> {
        let mut branch = repo
            .find_branch(name, git2::BranchType::Local)
            .map_err(|e| format!("branch lookup failed: {e}"))?;
        branch.delete().map_err(|e| e.to_string())
    };

    let mut results = Vec::new();

    for candidate in &candidates {
        // `is_switched_head` is only meaningful when `action` is `Delete` —
        // if resolve_head failed, the action stays `Protect` and we never
        // reach the Delete arm where switched_to_name is used.
        let is_switched_head = switched_head_name == Some(candidate.name.as_str());

        let outcome = match &candidate.action {
            TrimAction::Protect { reason } => TrimOutcome::Protected {
                reason: reason.clone(),
            },
            TrimAction::Delete => {
                if dry_run {
                    match switched_to_name {
                        Some(target) if is_switched_head => {
                            TrimOutcome::WouldDeleteAfterSwitch {
                                switched_to: target.to_string(),
                            }
                        }
                        _ => TrimOutcome::WouldDelete,
                    }
                } else {
                    match delete_branch(&candidate.name) {
                        Ok(()) => match switched_to_name {
                            Some(target) if is_switched_head => {
                                TrimOutcome::DeletedAfterSwitch {
                                    switched_to: target.to_string(),
                                }
                            }
                            _ => TrimOutcome::Deleted,
                        },
                        Err(reason) => TrimOutcome::Failed { reason },
                    }
                }
            }
        };

        results.push(BranchTrimResult {
            name: candidate.name.clone(),
            tracking: candidate.tracking.clone(),
            outcome,
        });
    }

    Ok(RepoTrimResult {
        alias,
        branches: results,
    })
}
```

#### 4. Update presenter for new outcome variants

**File**: `src/presenter.rs`

- [x] Add match arms for `DeletedAfterSwitch` and `WouldDeleteAfterSwitch` in `format_trim_report()` (after line 360, before the `Protected` arm):

```rust
TrimOutcome::DeletedAfterSwitch { switched_to } => {
    deleted += 1;
    format!(
        " {} {}: deleted {} ({tracking_label}, was HEAD \u{2192} {switched_to})",
        "✓".green(),
        report.alias,
        branch.name,
    )
}
TrimOutcome::WouldDeleteAfterSwitch { switched_to } => {
    would_delete += 1;
    format!(
        " {} {}: would delete {} ({tracking_label}, was HEAD \u{2192} {switched_to})",
        "✓".cyan(),
        report.alias,
        branch.name,
    )
}
```

The `→` character is `\u{2192}` (Unicode rightwards arrow).

#### 5. Verify exhaustiveness across `TrimOutcome` match sites

**Files**: `src/main.rs`, `src/presenter.rs`

- [x] The `has_failures` check at `main.rs:383-386` uses `matches!(b.outcome, TrimOutcome::Failed { .. })`, which returns `false` for the new variants — correct behavior (a successful switch-and-delete is not a failure). No code change needed.

- [x] The `wildcard_enum_match_arm` restriction lint in `Cargo.toml` means any explicit `match` on `TrimOutcome` with a `_` arm will fail to compile when new variants are added. The `format_trim_report` match block in `presenter.rs:343-379` is the primary site — adding the new arms (step 4) satisfies this. The `matches!()` macro in `main.rs:386` is not affected by this lint (clippy's `wildcard_enum_match_arm` targets explicit `match` blocks in user code, not macro-generated fallthrough arms). Confirmed with `cargo clippy --all-targets`.

- [x] If existing presenter tests cover the summary footer (e.g., `test_trim_report_summary_footer`), verify they still pass. The new variants increment the same counters (`deleted`, `would_delete`) as their non-switch counterparts, so existing summary assertions should hold. Add a test case that includes a `DeletedAfterSwitch` result and verifies it contributes to the `deleted` count in the summary.
  > **Deviation:** Existing presenter tests pass unchanged. Skipped adding a dedicated presenter test for `DeletedAfterSwitch` summary counting — the `test_trim_gone_head_auto_switched` unit test and the `test_trim_protects_head` CLI integration test both verify the end-to-end behavior including output formatting.

#### Tests for This Phase

**File**: `src/trimmer.rs` (add to the existing `#[cfg(test)]` module)

- [x] `test_trim_gone_head_auto_switched`: Set up a repo with a gone branch on HEAD (reuse `init_repo_with_gone_branch`, then reopen the repo via `Repository::open` and call `set_head()` — matching the pattern in `test_trim_head_branch_protected`). Run `trim_repo`. Assert:
  > **Deviation:** This test replaces the original `test_trim_head_branch_protected` which now verifies the auto-switch behavior instead of protection. The CLI integration test `test_trim_protects_head` was also updated to match.
  - The gone branch appears in results with `TrimOutcome::DeletedAfterSwitch { switched_to: main_name }`
  - The branch no longer exists in the repo
  - HEAD now points to the main branch (`repo.head().shorthand()` == main_name)

- [x] `test_trim_gone_head_dirty_worktree_protected`: Set up a gone branch on HEAD with a **divergent tree** from main — after creating the gone branch, add a commit on main that modifies a tracked file (so the main tree differs from the gone branch tree). Then switch HEAD to the gone branch, modify the same tracked file in the worktree without committing. Run `trim_repo`. The `safe()` checkout will refuse because the worktree file differs from the current HEAD tree and the target tree wants to change it. Assert:
  - The gone branch appears with `TrimOutcome::Protected { reason }` where `reason` contains "switch to" and "failed"
  - The branch still exists
  - HEAD still points to the gone branch

- [x] `test_trim_gone_head_dry_run`: Set up a gone branch on HEAD. Run `trim_repo` with `dry_run: true`. Assert:
  - The gone branch appears with `TrimOutcome::WouldDeleteAfterSwitch { switched_to: main_name }`
  - The branch still exists
  - HEAD still points to the gone branch (no switch performed)

- [x] `test_trim_gone_head_no_main_branch_protected`: Set up a gone branch on HEAD in a repo where no main branch exists (e.g., the only branches are "feature-gone" on HEAD and "other"). Run `trim_repo` with `main_branches: ["main", "master"]`. Assert:
  - The gone branch appears with `TrimOutcome::Protected { reason }` where reason contains "no main branch found"

### Success Criteria:

#### Automated Verification:

- [x] `just check` passes (format + lint + build + test + license check)
  > **Deviation:** Ran individual checks (same as Phase 1): `cargo fmt --check`, `cargo clippy --all-targets`, `cargo test`, `cargo deny check licenses` — all pass.
- [x] All existing trim tests pass: `cargo test --lib trimmer` (updated `test_trim_head_branch_protected` → `test_trim_gone_head_auto_switched`)
- [x] All four new tests pass: `cargo test --lib trimmer` (3 new + 1 updated = 9 total trimmer tests)
- [x] CLI integration tests pass: `cargo test --test cli` (updated `test_trim_protects_head` for auto-switch behavior)
- [x] `cargo clippy --all-targets` passes (including `wildcard_enum_match_arm` enforcement on new variants)

---

## Performance Considerations

The refactor adds a second branch lookup (`repo.find_branch()`) during the action phase for branches that will be deleted. This is a hash table lookup in libgit2's reference store — negligible compared to the I/O of the deletion itself. The collection phase iterates branches exactly once, same as today.

The libgit2 checkout in `resolve_head` involves tree comparison and worktree updates, but this runs at most once per repo (only when HEAD is on a gone branch), making it effectively free in the common case.

## References

- Idea document: `thoughts/ideas/2026-04-12-gityard-trim-auto-switch-gone-head.md`
- Current trim implementation: `src/trimmer.rs:42-129`
- Tracking state detection: `src/branches.rs:144-165`
- `BranchInfo` struct: `src/branches.rs:20-38`
- Trim output formatting: `src/presenter.rs:322-410`
- CLI args: `src/cli.rs:148-160`
- Command wiring: `src/main.rs:356-393`
- Executor collect-validate-act pattern: `src/executor.rs:29-110`
- libgit2 checkout in tests: `src/branches.rs:263-265`
- `wildcard_enum_match_arm` lint: `Cargo.toml` restriction lints section
