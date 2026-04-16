# Prune Scanner at Repo Boundaries During `gityard add`

## Problem Statement

When running `gityard add <directory>` on a directory containing repositories,
the scanner continues walking into discovered repos' subdirectories. This causes
it to discover and register submodules (and any other nested repos) as
independent tracked repositories alongside their parent repos. This is
unexpected — submodules are pinned dependencies managed by their parent, and
repos nested inside other repos are not typically independent projects the user
wants to track separately.

The result is a cluttered registry where parent repos and their nested repos
appear as peers in `gityard status`, and bulk operations like `gityard fetch` act
on them independently rather than letting the parent repo manage them.

## Motivation

- **Submodules aren't independent repos.** Their checked-out commit and branch
  state are controlled by the parent. Tracking them separately creates a
  confusing duplicate view.
- **Bulk operations become problematic.** Running `gityard fetch` on both a
  parent and its submodules independently can put submodules at different commits
  than the parent expects. The parent's `--recurse-submodules` (already supported
  in gityard's fetch config) is the correct mechanism.
- **User expectations.** Multi-repo tools typically distinguish top-level
  projects from their submodules. Adding submodules by default is surprising.

## Context

The scanner (`scanner.rs`) uses a simple heuristic: any directory containing a
`.git` entry (file or directory) is a repository. Git submodules have a `.git`
*file* (not directory) containing a `gitdir:` pointer to the parent's
`.git/modules/` directory. The scanner doesn't distinguish between these cases.

The scanner also does not prune the walk tree after finding a repo, so it
continues descending into discovered repos and finds submodules within the depth
limit.

gityard already has submodule-aware features in the fetch path
(`recurse_submodules` config and `--recurse-submodules` CLI flag), so there's
precedent for treating submodules as subordinate to their parent.

## Goals / Success Criteria

- [ ] Scanner prunes subtrees at repo boundaries — once a directory is
      identified as a repo, its contents are not walked further
- [ ] Skipped nested repos (submodules or otherwise) produce a visible message
      (e.g., "Skipping nested repo: path/to/submodule") so the user knows what
      was found and excluded
- [ ] `gityard add <path-to-submodule>` (explicit direct path) still works
      silently — pruning only applies to scanning/discovery, not explicit adds

## Non-Goals (Out of Scope)

- Removing already-registered submodules from existing registries (that's a
  migration/cleanup concern, not part of this change)
- Changing how `gityard fetch --recurse-submodules` works
- Detecting or handling nested submodules differently from top-level submodules
- Adding submodule-aware display to `gityard status` (e.g., showing submodule
  state under the parent)

## Constraints

- The scanner currently returns a flat `Vec<PathBuf>` — pruning should work at
  the scanner level via `walkdir`'s `filter_entry` rather than requiring the
  caller to post-filter.

## Resolved Questions

- **Should the scanner stop descending into discovered repos?** Yes — prune at
  repo boundaries. This is simpler than submodule-specific filtering and handles
  the broader case (nested independent repos are also excluded from scan).
  Submodule detection becomes unnecessary for the scan path since submodules are
  never reached.
- **Should `gityard add <direct-submodule-path>` warn?** No — add it silently.
  The user explicitly chose that path; pruning is a scan/discovery concern, not
  a direct-add concern.

## References

- Scanner implementation: `src/scanner.rs:11-41`
- Registry add: `src/registry.rs:73-101`
- Existing submodule config: `src/config.rs:111-112` (`recurse_submodules`)
- Fetch submodule flag: `src/cli.rs:127-129`
- Git submodule `.git` file format: `gitdir: <path>` (since Git 1.7.8)
