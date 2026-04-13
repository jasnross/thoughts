# Extract General-Purpose Conflict Resolution Skill

## Problem Statement

The `gt-sync` command contains a detailed conflict resolution workflow (Phase 3:
steps 1–8) that is tightly coupled to the git-town sync loop. This logic —
identify conflicted files, analyze per-hunk semantics via subagents, present a
summary with confidence levels, collect user approval, write resolved files,
verify no markers remain, and stage — is universally useful whenever git leaves
unmerged files in the working tree. Today there is no way to invoke this
workflow outside of a `git-town sync`.

## Motivation

- **Ad-hoc conflict resolution.** After a `git merge`, `git rebase`,
  `git cherry-pick`, or `git stash pop` that hits conflicts, the user currently
  has to resolve files manually. A `/resolve-conflicts` command would bring the
  same analyze-and-approve UX to all of these scenarios.
- **DRY across commands.** `gt-sync` currently inlines the full conflict
  resolution procedure. As more commands encounter conflicts (e.g., a future
  `gt-ship` or `gt-rebase`), duplicating this logic becomes a maintenance
  burden. A shared skill keeps the procedure in one place.
- **Consistent UX.** Users get the same summary table, approval flow, and
  marker verification regardless of what operation caused the conflict.

## Context

### What exists today

- **`spec/commands/gt-sync.md` Phase 3** — The full conflict resolution
  workflow: identify → analyze (parallel subagents) → present summary → user
  approval → write → verify markers → stage. This is the logic to extract.
- **`spec/skills/git-town/SKILL.md`** — Non-invocable skill for git-town
  operations. The conflict resolution skill would sit alongside this as another
  non-invocable skill, but git-agnostic (not git-town-specific).
- **`spec/commands/split-branch.md`** — Architectural reference for the
  skill-loading and subagent-dispatch patterns.

### Git conflict states to support

All states where `git diff --name-only --diff-filter=U` returns unmerged files:

| Operation         | Continue command             |
|-------------------|------------------------------|
| `git merge`       | `git merge --continue`       |
| `git rebase`      | `git rebase --continue`      |
| `git cherry-pick` | `git cherry-pick --continue` |
| `git stash pop`   | (no continue — already done) |
| `git-town sync`   | `git-town continue`          |
| `git-town ship`   | `git-town continue`          |

The skill should detect which operation is in progress and offer the
appropriate continue action. Detection can use:

```bash
# Rebase in progress
test -d "$(git rev-parse --git-dir)/rebase-merge" || \
  test -d "$(git rev-parse --git-dir)/rebase-apply"

# Merge in progress
test -f "$(git rev-parse --git-dir)/MERGE_HEAD"

# Cherry-pick in progress
test -f "$(git rev-parse --git-dir)/CHERRY_PICK_HEAD"
```

For `git-town` operations, the caller (`gt-sync`) already knows the context and
can pass it in — the skill doesn't need to detect git-town state itself.

## Goals / Success Criteria

- [ ] A non-invocable `resolve-conflicts` skill exists in `spec/skills/` with
  the full resolve-verify-stage cycle
- [ ] A thin invocable `resolve-conflicts` command exists in `spec/commands/`
  that detects the current conflict state, loads the skill, resolves, and offers
  to continue the interrupted operation
- [ ] `gt-sync` is updated to load the `resolve-conflicts` skill and delegate
  Phase 3 steps 1–8 to it, keeping only the gt-sync-specific loop (conflict
  detection, `git-town continue`, round counter)
- [ ] The skill's analyze → present → approve → write → verify → stage flow is
  identical to what `gt-sync` has today (no behavior change for existing users)
- [ ] Build passes for all four provider targets
- [ ] Schema validates with no errors

## Non-Goals (Out of Scope)

- **Automatic resolution without user approval.** The skill only suggests — the
  user must approve every file.
- **Three-way merge tooling.** The skill works from conflict markers in the
  working tree, not from a merge tool interface.
- **Conflict prevention.** No pre-merge checks, no "rebase before merge"
  recommendations.
- **Resolving conflicts in binary files.** Binary conflicts should be flagged
  and deferred to the user with clear instructions.
- **Retry/undo after continue.** If the continue command fails, that's the
  caller's problem (or the standalone command surfaces the error and stops).

## Proposed Architecture

### Two-piece design

1. **`spec/skills/resolve-conflicts/SKILL.md`** (non-invocable)
   - Core procedure: identify conflicted files, dispatch parallel subagent
     analysis, present summary table, collect approval, write resolved files,
     verify markers, stage
   - Accepts context from the caller: optional operation name (for display),
     optional list of files to restrict to
   - **Partial resolution loop:** after staging approved files, re-checks for
     remaining unmerged files. If any remain (user chose "Resolve manually"),
     shows them and asks: "Re-analyze now (if you've edited it)" or "Stop here
     (you'll continue manually)". Loops until all files are resolved or the user
     stops.
   - Returns to the caller: list of resolved files, count of conflicts resolved,
     whether any files were skipped (manual resolution)

2. **`spec/commands/resolve-conflicts.md`** (user-invocable)
   - Pre-flight: verify unmerged files exist, detect which operation is in
     progress (rebase, merge, cherry-pick, or stash pop — each detected via
     sentinel files in `.git/`)
   - Loads the `resolve-conflicts` skill and invokes it
   - Post-resolution:
     - **rebase / merge / cherry-pick:** offer to run the appropriate
       `--continue` command
     - **stash pop:** offer `git stash drop stash@{0}` with an explanation
       (pop = apply + drop; apply succeeded but drop was skipped due to the
       conflict)
     - **git-town operations:** caller (`gt-sync`) handles the continue —
       the skill just stages and returns
   - If no conflicts detected, shows a clean message and stops

### How gt-sync changes

`gt-sync` Phase 3 shrinks to:

1. Detect conflicts (unchanged)
2. Report conflict location and round counter (gt-sync-specific)
3. **Load and invoke the `resolve-conflicts` skill** (replaces steps 3–8)
4. Run `git-town continue` (gt-sync-specific)

The skill handles everything between "here are unmerged files" and "files are
staged and marker-free."

## Constraints

- Must follow the canonical spec schema (`spec/schema/canonical.schema.json`)
- Provider-neutral prose throughout (no tool name capitalization in body)
- The skill's subagent dispatch pattern must match existing conventions
  (`split-branch`, `gt-sync`)
- The standalone command should work in any git repo — no git-town dependency

## Decisions

- **Skill loading mechanism:** The schema has no `requires` or `dependencies`
  field. Skill loading is prose-only — "Load the `resolve-conflicts` skill" —
  consistent with how `gt-sync` and `split-branch` reference the `git-town`
  skill today.
- **Stash pop edge case:** After resolving a stash pop conflict, offer
  `git stash drop stash@{0}` with an explanation: "The stash entry still exists
  because pop stops on conflict — drop it now that the conflict is resolved?"
  The user can accept or decline.
- **Partial resolution:** After staging approved files, the skill re-checks for
  remaining unmerged files and loops: "Re-analyze now (if you've edited it)" or
  "Stop here (you'll continue manually)". This lets the user resolve in their
  editor and then hand back to the skill for analysis.

## References

- Existing conflict resolution logic: `spec/commands/gt-sync.md` (Phase 3)
- Skill pattern reference: `spec/skills/git-town/SKILL.md`
- Command pattern reference: `spec/commands/split-branch.md`
- Schema: `spec/schema/canonical.schema.json`
- Naming conventions: `agent-docs/canonical-command-conventions.md`
