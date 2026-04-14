# Default "Hide Clean" Filtering for `status` and `branches`

## Problem Statement

When managing many repositories with `gityard status` and `gityard branches`,
the output includes every registered repo and every local branch -- even those
that are in a completely clean state with nothing actionable. As the number of
managed repos grows, the signal-to-noise ratio drops: the repos and branches
that need attention get buried among those that don't.

Users scanning output are almost always looking for things that need action --
dirty repos, branches that haven't been merged, repos that are out of sync. The
"everything is fine" entries are noise in the common case.

## Motivation

- **Faster scanning**: When you have 20+ repos, most are often clean. Showing
  only the ones that need attention makes the output immediately actionable.
- **Consistent UX**: `gityard worktrees` already hides main working trees by
  default (requiring `--all` to show them). Applying the same pattern to
  `status` and `branches` makes the CLI feel consistent.
- **Progressive disclosure**: Default to the useful view, let users opt into the
  full view when needed.

## Context

### Current `gityard status` behavior

Shows all registered repos with: branch name, dirty state (staged/modified/
untracked counts), ahead/behind main, ahead/behind remote, and stash count.
Existing filter flags (`--dirty`, `--clean`, `--ahead`, `--behind`, `--diverged`)
use OR semantics -- with no flags, all repos are shown.

### Current `gityard branches` behavior

Shows all local branches across repos with: name, HEAD marker, ahead/behind
main, tracking state (tracking/gone/untracked), last commit time, and merged
status. Existing filter flags (`--gone`, `--no-remote`) use OR semantics -- with
no flags, all branches are shown.

### Precedent: `gityard worktrees --all`

The `worktrees` subcommand hides main working trees by default, showing only
linked worktrees. The `--all` flag includes main working trees. This is the UX
pattern to follow.

## Goals / Success Criteria

- [ ] `gityard status` hides "totally clean" repos by default
- [ ] `gityard branches` hides "totally clean" branches by default
- [ ] `--all` flag on both commands shows everything (including clean items)
- [ ] `--clean` flag on both commands shows only clean items
- [ ] `--topic` flag on both commands shows only topic (non-main) items
- [ ] Existing filter flags (`--dirty`, `--ahead`, `--topic`, etc.) override the
      default hiding -- when any explicit filter is passed, the default
      clean-hiding is disabled and only the explicit filter applies
- [ ] Output indicates how many items were hidden (e.g., "3 clean repos hidden,
      use --all to show")
- [ ] Behavior is consistent with the existing `worktrees --all` pattern

## Non-Goals (Out of Scope)

- Changing what data is collected or displayed per repo/branch (only filtering
  which rows appear)
- Adding new filter flags beyond `--all`, `--clean`, and `--topic`
- Changing the `worktrees` subcommand behavior (already has `--all`)
- Persisting filter preferences in config (always use CLI flags)

## Definitions

### "Totally clean" for `gityard status`

A repo is considered totally clean when ALL of these are true:

1. **On a main branch** -- current branch matches one of the configured
   `main_branches` (default: `["main", "master"]`)
2. **Not dirty** -- `staged_count`, `modified_count`, and `untracked_count` are
   all zero
3. **Not ahead/behind main** -- `ahead_main` and `behind_main` are both zero (or
   N/A because already on main)
4. **Not ahead/behind remote** -- `ahead_remote` and `behind_remote` are both
   zero (tracking branch is in sync)

### "Totally clean" for `gityard branches`

A branch is considered totally clean when BOTH of these are true:

1. **Merged into main** -- the branch tip is an ancestor of the main branch tip
2. **Has active upstream tracking** -- the branch has a configured upstream that
   still exists (not `Gone` or `Untracked`)

## Proposed Solution

### Flag design

| Flag | Commands | Behavior |
|------|----------|----------|
| _(no flags)_ | both | Hide clean items (new default) |
| `--all` | both | Show everything, including clean items |
| `--clean` | both | Show only clean items |
| `--topic` | both | Show only topic (non-main) items. On `status`: repos checked out on a non-main branch. On `branches`: branches that aren't a configured main branch. |
| `--dirty`, `--ahead`, etc. | status | Override default hiding; apply explicit filter to full set |
| `--gone`, `--no-remote` | branches | Override default hiding; apply explicit filter to full set |

### Interaction with existing filters

When any existing filter flag is passed (e.g., `--dirty`, `--ahead`, `--behind`,
`--diverged`, `--gone`, `--no-remote`, `--topic`), the default clean-hiding is
disabled. The explicit flag is applied to the full unfiltered set, preserving
current behavior. This means existing scripts and muscle memory are unaffected.

The `--all` flag is effectively "show everything" and could be thought of as
disabling all filtering.

The `--clean` flag inverts the default -- instead of hiding clean items, it shows
only clean items.

### Hidden-items summary

When items are hidden by the default filter, display a footer line:

```
(3 clean repos hidden — use --all to show)
```

This follows the principle of least surprise: users see that filtering happened
and know how to disable it.

## Constraints

- Must not break existing CLI usage -- all current flag combinations should
  produce the same results they do today
- The `--all` and `--clean` flags should be mutually exclusive (showing
  everything vs. showing only clean)
- `--clean` and `--dirty` (on status) should be mutually exclusive

## Resolved Questions

- **Stash count and "clean" for status**: No. Stashes are intentionally parked
  work and don't affect the clean definition. The clean definition focuses on
  sync state (branch, dirty, ahead/behind).
- **Main branch in `branches` output**: Yes, hide it by default. It meets the
  clean criteria (merged into itself, typically tracked) and showing it adds no
  actionable information. Consistent with the strict clean definition.
- **Suppressible hidden-items summary**: No `--quiet` flag needed. The summary
  is a single line and only appears when items are hidden. If scripting needs
  arise later, it can be added then.

## Affected Systems

- `src/cli.rs` -- Add `--all` and `--clean` flags to `StatusArgs` and
  `BranchesArgs`
- `src/filter.rs` -- Add clean-detection logic to `StatusFilter` and
  `BranchFilter`; handle interaction between default hiding and explicit flags
- `src/main.rs` -- Wire up new filter logic in `cmd_status()` and
  `cmd_branches()`
- `src/presenter.rs` -- Add hidden-items summary footer
