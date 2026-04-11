# Git Integration Workflows

<!-- migrated-from: /Users/jasonr/.config/agent-docs/git-integration-workflows.md -->

## Merge Without Merge Commit

<!-- learned: 2026-02-12 -->

To merge a feature branch into `master` without creating a merge commit, rebase the feature branch onto `origin/master`, then switch to `master` and run `git merge --ff-only <feature-branch>`. If fast-forward fails, fetch updates, rebase again, and retry the fast-forward merge.

## Continue rebase without launching interactive editor

<!-- learned: 2026-03-08 -->

In non-interactive shells, `git rebase --continue` may open `nvim` and hang. If no commit message edit is needed, run `GIT_EDITOR=true git rebase --continue` to continue safely without launching an editor.
