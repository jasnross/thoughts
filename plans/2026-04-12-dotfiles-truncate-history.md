# Dotfiles Repository History Truncation Plan

## Overview

Truncate the dotfiles repository history by squashing all 870 commits up to and
including Dec 30, 2023 into a single root commit, preserving the 956
post-cutoff commits individually. Also remove the `fonts/` directory (59M) from
the entire history. This eliminates ~30M+ of dead binary blobs (deleted
`windows/`, `.atom/` directories) from the packfile, removes 59M of unused font
files from the working tree, and reduces total commit count by ~52%.

## Current State Analysis

- **1,826 commits** spanning Oct 2015 – Apr 2026
- **107M `.git` directory** — bloated by deleted binaries in old history
- **Cutoff commit**: `60d22959` (Dec 30, 2023) — last commit before 2024
- **Dead weight**: `windows/` (vim .exe files, gnuwin32 DLLs) and `.atom/`
  packages were deleted years ago but still occupy ~30M+ in packfiles. Neither
  directory exists at the cutoff point, so squashing eliminates them.
- **1 merge commit** after cutoff (`1ccc9569 Merge branch 'jaross/mason-rust'`)
  — will be linearized during rebase, which is acceptable for a personal
  dotfiles repo.
- **Stale branch**: `hello` — has an active worktree at
  `.worktrees/hello`. Must remove the worktree before deleting the branch.
- **Remote**: `origin` on Bitbucket (`git@bitbucket.org:jasnross/dotfiles.git`)

### Key Discoveries

- `windows/` and `.atom/` directories do not exist in the tree at the cutoff
  commit, so the squashed root commit's tree will not reference those blobs.
  After gc, they will be pruned.
- `fonts/` (59M) exists at the cutoff point and in the current tree, but **no
  post-cutoff commit touches it**. Removing it from the root commit's tree is
  sufficient — the rebase replays diffs, so `fonts/` will not reappear in any
  rebased commit.
- No tags exist in the repository.
- `git-filter-repo` is not installed, but the native `git commit-tree` +
  `git rebase` approach requires no external tools.

## Desired End State

- A single root commit containing the full tree state as of Dec 30, 2023,
  minus the `fonts/` directory
- All 956 post-cutoff commits replayed on top with their original messages,
  authors, and dates preserved
- `fonts/` directory absent from every commit in the repository
- `.git` directory significantly smaller after dead blobs are pruned
- Remote updated to match the rewritten history
- Stale `hello` branch deleted locally and remotely

### Verification

- `git log --oneline | wc -l` returns ~957 (1 root + 956 replayed)
- `git ls-tree -r HEAD --name-only | grep fonts` returns nothing
- `du -sh .git` is significantly smaller than 107M
- `git log --oneline | tail -1` shows the squashed root commit message
- Working tree no longer contains `fonts/`

## What We're NOT Doing

- Migrating to a different hosting provider
- Changing the repository structure or file organization
- Using `git-filter-repo` or BFG — the native git approach is sufficient
- Preserving the single merge commit's topology — linearizing is acceptable

## Implementation Approach

Use git's index manipulation to build a modified tree (cutoff snapshot minus
`fonts/`), then `git commit-tree` to create a parentless root commit from it,
and `git rebase --onto` to replay all subsequent commits on top. Since no
post-cutoff commit touches `fonts/`, the rebase replays diffs that never
reference it — so `fonts/` is cleanly absent from every commit in the rewritten
history. This is a pure-git approach requiring no external tools.

## Implementation

### Overview

Build a modified tree from the cutoff snapshot with `fonts/` removed, create a
new root commit from it, rebase post-cutoff history onto it, verify correctness,
then force-push and clean up.

### Changes Required

#### 1. Create a Full Backup

Before any destructive operations, create a complete backup.

- [x] Clone the repo to a backup location:
  ```bash
  git clone --mirror /Users/jasonr/Workspace/jasnross/dotfiles /Users/jasonr/Workspace/jasnross/dotfiles-backup.git
  ```

#### 2. Remove the Stale Worktree and Branch

- [x] Remove the `hello` worktree:
  ```bash
  git worktree remove .worktrees/hello
  ```
- [x] Delete the `hello` branch locally:
  ```bash
  git branch -D hello
  ```
- [x] Delete the `hello` branch on the remote (if it exists — ignore errors):
  ```bash
  git push origin --delete hello 2>/dev/null || true
  ```

#### 3. Create the Squashed Root Commit (without `fonts/`)

Build a tree from the cutoff snapshot with `fonts/` removed, then create a
parentless commit from it.

- [x] Load the cutoff tree into a temporary index, remove `fonts/`, and create
  the root commit:
  ```bash
  TMPIDX=$(mktemp)
  TREE=$(GIT_INDEX_FILE="$TMPIDX" sh -c '
    git read-tree 60d22959^{tree} &&
    git rm -r --cached --quiet fonts &&
    git write-tree
  ')
  rm -f "$TMPIDX"
  NEW_ROOT=$(git commit-tree "$TREE" -m "Initial commit (squashed history before 2024)")
  echo "New root: $NEW_ROOT"
  ```
  This uses a temporary index file so the working tree and real index are
  untouched. `git read-tree` loads the cutoff snapshot, `git rm --cached`
  strips `fonts/`, `git write-tree` captures the modified tree, and
  `git commit-tree` creates the parentless root commit from it. The temp
  file is cleaned up immediately after use.

#### 4. Rebase Post-Cutoff History onto the New Root

- [x] Rebase all commits after the cutoff onto the new root:
  ```bash
  git rebase --committer-date-is-author-date --onto $NEW_ROOT 60d22959 master
  ```
  This replays all 956 commits that descend from `60d22959` onto `$NEW_ROOT`.
  The `--committer-date-is-author-date` flag preserves original timestamps.
  The single merge commit (`1ccc9569`) will be linearized automatically.

- [x] Verify the rebase completed successfully.
  > **Deviation:** Two issues encountered during rebase: (1) 20 untracked
  > `.zsh/completions/` files had to be moved to `/tmp` before the rebase
  > could proceed, then restored afterward. (2) One merge conflict in
  > `cspell/custom-words.txt` from linearizing the merge commit — resolved
  > by keeping both sides' additions. The empty merge commit was dropped,
  > resulting in 956 total commits (955 replayed + 1 root) instead of 957.

#### 5. Verify Correctness

- [x] Confirm `fonts/` is absent from the working tree, root commit tree, and
  all history:
  ```bash
  ls fonts/ 2>&1          # should say "No such file or directory"
  git ls-tree $(git rev-list --max-parents=0 HEAD) --name-only | grep fonts  # no output
  git log --all -- fonts/  # should produce no output
  ```
- [x] Confirm the working tree is clean:
  ```bash
  git status  # should show nothing to commit
  ```
- [x] Confirm commit count is ~957:
  ```bash
  git log --oneline | wc -l  # returns 956 (merge commit dropped)
  ```
- [x] Confirm the root commit has the correct tree:
  ```bash
  git log --oneline | tail -1  # 6b407831 Initial commit (squashed history before 2024)
  ```
- [x] Confirm the latest commit matches the original HEAD:
  ```bash
  git log --oneline -1  # 6967c0a5 chore(agent-config): regenerate compiled output with resolved spec names
  ```
- [x] Spot-check a few mid-range commits to verify messages and authorship:
  ```bash
  git log --format="%h %ai %an %s" | head -20  # all dates and authors preserved
  ```
- [x] Run fsck to check repository integrity:
  ```bash
  git fsck  # no errors, only dangling objects (expected, cleaned by gc)
  ```

#### 6. Force-Push to Remote

Force-push before pruning so the local reflog remains available as a safety net
if the push fails.

- [x] Force-push the rewritten master branch:
  ```bash
  git push --force origin master
  ```
- [x] Verify the remote matches local:
  ```bash
  git log --oneline origin/master | wc -l   # 956
  git log --oneline origin/master | head -5  # matches local HEAD
  ```

#### 7. Prune Unreachable Objects

- [x] Expire the reflog to make old objects unreachable:
  ```bash
  git reflog expire --expire=now --all
  ```
- [x] Run aggressive garbage collection to prune dead blobs:
  ```bash
  git gc --prune=now --aggressive
  ```
- [x] Also pruned Git LFS cache (59M of font objects):
  ```bash
  git lfs prune  # removed 66 unreferenced LFS objects
  ```
  > **Deviation:** The fonts were stored via Git LFS, not regular git objects.
  > `git gc` alone only reduced `.git` to 62M. `git lfs prune` was needed to
  > remove the 59M LFS cache, bringing `.git` to 2.6M.
- [x] Verify size reduction:
  ```bash
  du -sh .git  # 2.6M (down from 107M — 97% reduction)
  du -sh .     # 6.3M (down from 234M — 97% reduction)
  ```

#### 8. Clean Up (Manual)

- [ ] Remove the backup once fully satisfied everything works:
  ```bash
  rm -rf /Users/jasonr/Workspace/jasnross/dotfiles-backup.git
  ```
  **Do not automate this step** — delete the backup manually after verifying
  the repo over a period of normal use.

### Rollback

If anything goes wrong, restore from the mirror backup:

```bash
# Delete the broken repo
rm -rf /Users/jasonr/Workspace/jasnross/dotfiles

# Restore from backup
git clone /Users/jasonr/Workspace/jasnross/dotfiles-backup.git /Users/jasonr/Workspace/jasnross/dotfiles

# Restore the remote
cd /Users/jasonr/Workspace/jasnross/dotfiles
git remote set-url origin git@bitbucket.org:jasnross/dotfiles.git
git push --force --all origin
```

### Success Criteria

#### Automated Verification

- [x] `git log --oneline | wc -l` returns 956 (957 minus 1 dropped empty merge)
- [x] `git log --oneline | tail -1` shows the squashed root commit
- [x] `git status` shows clean working tree
- [x] `git ls-tree $(git rev-list --max-parents=0 HEAD) --name-only | grep fonts` produces no output
- [x] `git log --all -- fonts/` produces no output
- [x] `ls fonts/` reports "No such file or directory"
- [x] `du -sh .git` is 2.6M (down from 107M)
- [x] `git fsck` reports no errors
- [x] `git log --oneline origin/master | wc -l` matches local count (956)

#### Manual Verification

- [ ] Spot-check that recent commit messages and authorship are preserved
