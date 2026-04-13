# Git-Town Integration Skill

## Problem Statement

The `split-branch` command creates stacked branches (where one branch depends on
another) using plain `git branch` + worktrees, but has no awareness of git-town.
When git-town is active in a repository, these stacked branches exist outside
git-town's parent-child tracking, so git-town's stack-aware commands (`sync
--stack`, `propose --stack`, `branch`, `ship`) don't work on them. The user must
manually register each relationship with `git town set-parent` after a split —
tedious and error-prone for multi-branch stacks.

## Motivation

- **git-town is already in use.** The dotfiles include global git-town config
  (`git-town.toml`), a setup script (`setup-git-town.sh`) for personal and work
  repos, shell aliases (`gt`, `gtb`), and completions. It's the active branch
  management tool.
- **Stacked branches are git-town's core strength.** Once registered, git-town
  automatically rebases through the hierarchy on `sync`, creates correctly-
  targeted PRs on `propose`, and enforces merge order on `ship`.
- **split-branch already models the same relationships.** It tracks which groups
  are independent (branching from `$BASE_BRANCH`) and which stack on a previous
  group. The dependency graph is already computed — it just isn't communicated to
  git-town.
- **Reusable beyond split-branch.** Other commands that create branches with
  dependencies (or future commands) could use the same git-town integration
  logic.

## Context

### Current split-branch behavior (Phase 4)

Split-branch creates branches with `git branch` and processes them in dependency
order using worktrees:

```bash
# Independent group: branches from $BRANCH_POINT (origin/main HEAD or merge-base)
git branch "feature/foo--part-1-models" $BRANCH_POINT

# Stacked group: branches from a previous split branch
git branch "feature/foo--part-2-api" "feature/foo--part-1-models"
```

git-town has no knowledge of these relationships. Running `git town branch`
shows them as unrelated feature branches.

### git-town's parent tracking

git-town stores parent-child relationships in git config:

```
git-town-branch.feature/foo--part-2-api.parent=feature/foo--part-1-models
```

The `git town set-parent` command sets this interactively (prompts for parent),
or it can be set directly via `git config`. Once set, git-town treats the branch
as part of a stack.

### Existing git-town setup in dotfiles

| Component | Location |
|-----------|----------|
| Global config template | `~/dotfiles/git-town.toml` |
| Setup script | `~/dotfiles/scripts/setup-git-town.sh` |
| Shell aliases | `gt` = `git-town`, `gtb` = `git-town branch` |
| Completions | `_git-town` zsh completion |
| Auth | `GITHUB_TOKEN` from `gh auth token` |

Personal repos use committed `git-town.toml`; work repos use local-only
`.git-town.toml` with `ship.strategy = "api"` and GitHub Enterprise forge.

### Research context

A [comparative analysis of git-town vs git-branchless](../research/2026-03-01-git-town-vs-git-branchless.md)
was completed on 2026-03-01. Key findings relevant here:
- git-town is the healthier, more practical choice (v22.6.0, active releases)
- git-town explicitly does not support DAG/multi-parent branches (tree model only)
- `set-parent` is the mechanism for registering relationships post-hoc

## Goals / Success Criteria

- [ ] A standalone skill (`spec/skills/git-town/`) that other commands can load
- [ ] Detection of git-town availability (binary in PATH + config in repo)
- [ ] Post-hoc parent registration for stacked branches via `git town set-parent`
      or direct `git config` (whichever is more reliable for non-interactive use)
- [ ] Only stacked branches are registered; independent branches (based on main)
      are left as default feature branches
- [ ] Verification step: `git town branch` shows the expected hierarchy after
      registration
- [ ] split-branch's Phase 4 integrates with the skill when git-town is detected
- [ ] No changes to split-branch's branch creation logic (stays `git branch` +
      worktrees)
- [ ] No changes to split-branch's PR creation logic (stays `gh`-based, not
      `git town propose`)

## Non-Goals (Out of Scope)

- **Replacing `git branch` with `git town append/hack`** — The worktree-based
  execution model works well; we only need post-hoc registration.
- **PR creation via `git town propose`** — The current `gh pr create` approach
  generates richer PR descriptions (file summaries, split context, repo
  templates). `git town propose` is too basic.
- **DAG/multi-parent support** — git-town doesn't support this (issue #3232
  closed as "not planned"). The skill operates within git-town's tree model.
- **Managing git-town config** — The setup script (`setup-git-town.sh`) already
  handles config. This skill only reads config to detect git-town, it doesn't
  write it.
- **Replacing git-town CLI** — The skill wraps/invokes git-town, it doesn't
  reimplement its logic.

## Proposed Solution

### Skill structure

A standalone skill at `spec/skills/git-town/SKILL.md` that provides:

1. **Detection logic** — How to check if git-town is available and active:
   - `git-town` binary exists in `$PATH`
   - Repo has `git-town.toml` or `.git-town.toml`, or `git-town` section in
     `.git/config`

2. **Parent registration procedure** — For each stacked branch in a split:
   - Determine the correct parent (the branch it was created from)
   - Register via `git config git-town-branch.<branch>.parent <parent>` (direct
     config write, avoiding `set-parent`'s interactive prompt)
   - Alternatively, if git-town has a non-interactive `set-parent` flag, use that

3. **Verification** — After registration, run `git town branch` to confirm the
   hierarchy matches the expected dependency graph

4. **Skill metadata** — `user_invocable: false` (loaded by other commands, not
   invoked directly)

### Integration with split-branch

Split-branch would reference the git-town skill at two points:

- **Phase 1 (pre-flight):** Detect git-town. If present, note it in the
  pre-flight summary (e.g., "git-town: active, will register stacked branches").
- **Phase 4 (execution):** After all branches are created and verified, register
  parent-child relationships for stacked branches. Run `git town branch` to
  verify.

### Example flow

Given a split producing:
```
main ─── part-1-models ─── part-2-api
     └── part-3-cleanup (independent)
```

The skill would:
```bash
# For each stacked branch, checkout the branch and set its parent.
# set-parent operates on the current branch, so we must checkout first.
git checkout feature/foo--part-2-api
git town set-parent feature/foo--part-1-models

# part-1-models: no registration needed (parent is main, git-town's default)
# part-3-cleanup: no registration needed (independent, parent is main)

# Verify
git town branch
# Should show:
#   main
#   ├── feature/foo--part-1-models
#   │   └── feature/foo--part-2-api
#   └── feature/foo--part-3-cleanup
```

## Constraints

- **git-town's tree model.** Only single-parent relationships. If split-branch
  ever supports DAG dependencies, the git-town skill can't represent them.

## Resolved Questions

Research conducted 2026-03-02. Sources: git-town v22.6 docs, CLI help output,
GitHub issues #4705 and #1636.

### 1. `git config` vs `git town set-parent` for registration

**Answer: Use `git town set-parent <parent>` (positional argument).**

The `git-town-branch.<name>.parent` config key is documented but explicitly
described as internal: "You can ignore these configuration entries, Git Town
maintains them as it creates and removes feature branches." Writing it directly
would work today but is not a stable public API.

Since v19.0.0 (April 2025), `set-parent` accepts the parent branch name as a
**positional argument**, making it fully non-interactive:

```
git town set-parent [branch] [flags]
```

Example: `git town set-parent feature/foo--part-1-models` — sets the parent of
the *current* branch to `feature/foo--part-1-models` without any prompt.

The `--none` flag (v22.0.0) removes the parent, converting to perennial.

Locally verified with git-town v22.6.0:
```
$ git-town set-parent --help
Usage: git-town set-parent [branch] [flags]
```

Sources:
- https://www.git-town.com/commands/set-parent.html
- https://www.git-town.com/preferences/parent
- https://github.com/git-town/git-town/issues/4705

### 2. Should the skill run `git town sync --stack` after registration?

**Answer: No.** `sync --stack` would trigger rebases if the base has advanced,
which could introduce conflicts. Better to leave syncing as a user decision
after the split. The skill should only register parents and verify with
`git town branch`.

### 3. Does pushing branches confuse git-town's tracking?

**Answer: No, it works fine.**

git-town tracks two separate concerns: (a) parent relationships (in git config),
and (b) remote tracking (standard git `@{upstream}`). Manually pushing with
`git push -u origin <branch>` sets up the upstream tracking correctly, and
git-town respects it.

The only "missing piece" for manually-created branches is the parent
relationship. Without it, git-town classifies the branch as "unknown" and
prompts for a parent on first `sync`. Using `set-parent` before push (or before
the user runs sync) avoids the prompt entirely.

A historical bug where `set-parent` failed on manually-created branches (exit
status 5 from trying to `--unset` a non-existent config key) was fixed in v14.1,
so this is not a concern on v22.x.

Sources:
- https://www.git-town.com/branch-types.html
- https://www.git-town.com/preferences/push-branches
- https://github.com/git-town/git-town/issues/1636

### 4. Should the skill also set branch type?

**Answer: No.** git-town's `unknown-type = "feature"` config (present in the
dotfiles template) means manually-created branches default to feature type. Split
branches are features, so no explicit type setting is needed.

## Remaining Open Questions

- **Checkout requirement for `set-parent`.** The command operates on the current
  branch. In split-branch's Phase 4, the worktrees are still active. Should
  registration happen inside each worktree (where the branch is checked out), or
  after worktree cleanup (requiring checkout of each branch in the main tree)?
  The former avoids extra checkouts but means registration happens before build
  verification completes for all branches.

## References

- [split-branch spec](../../spec/commands/split-branch.md) — The command that
  would consume this skill
- [gh-safe skill](../../spec/skills/gh-safe/) — Pattern for standalone skills in
  this repo
- [git-town research](../research/2026-03-01-git-town-vs-git-branchless.md) —
  Comparative analysis of git-town vs git-branchless
- [git-town stacked changes guide](https://www.git-town.com/stacked-changes.html)
- [git-town configuration](https://www.git-town.com/configuration.html)
- [git-town set-parent command](https://www.git-town.com/commands/set-parent.html)
- [git-town parent preference](https://www.git-town.com/preferences/parent)
- [git-town.toml template](../../git-town.toml) — via `~/dotfiles/git-town.toml`
- [setup-git-town.sh](../../scripts/setup-git-town.sh) — Repo setup script

## Affected Systems/Services

- `spec/skills/git-town/` — New skill to create
- `spec/commands/split-branch.md` — Will need updates to reference the skill
- `mappings/` — May need adapter updates if the skill needs provider-specific
  behavior (likely not — skill is provider-neutral)
