# Worktree/trim test coverage and display_path symlink fix

## Problem Statement

The `gityard worktrees` and `gityard trim` commands have thorough unit test
coverage (11 trim scenarios, 7 worktree scenarios), but no CLI integration
tests exercising end-to-end behavior with actual worktrees. The existing CLI
tests only cover no-repos and `--help` cases. This means wiring bugs (argument
parsing, module composition, output formatting) and behavior regressions could
slip through undetected.

Separately, `display_path` in `presenter.rs` computes relative paths using
`pathdiff::diff_paths` without canonicalizing inputs first. When a worktree
path and its repo path traverse different symlink resolutions (e.g., macOS
`/var` vs `/private/var`, or any user-created symlink), the relative path
computation fails silently, producing an incorrect or overly long display path.

## Motivation

- **CLI integration tests catch what unit tests miss**: unit tests validate
  logic in isolation, but the CLI is the user-facing contract. Argument parsing,
  cross-module wiring, and output formatting are only exercised end-to-end.
- **Regression safety net**: as worktree and trim features evolve, CLI tests
  provide stable behavioral assertions that protect against accidental changes.
- **Edge case robustness**: behavior with submodules and bare repositories is
  now understood (see Resolved Questions below), and tests should codify the
  expected behavior.
- **Correct display paths**: even though the symlink issue is display-only,
  incorrect paths in CLI output erode user trust and make it harder to navigate
  to worktree directories.

## Context

### Current test landscape

- **Unit tests** (`trimmer.rs`, `worktrees.rs`): 18 tests covering trim
  pipeline, worktree inspection, edge cases (dirty, locked, invalid worktrees,
  HEAD-on-gone-branch, etc.)
- **Presenter tests** (`presenter.rs`): golden tests for table formatting
  including worktree and trim report tables
- **CLI tests** (`tests/cli.rs`): 25 tests covering most commands, but
  `worktrees` only has no-repos and help tests; `trim` has no
  worktree-interaction tests

### The symlink issue

`entry.path` (from `Registry::add`) is canonicalized at registration time
(`registry.rs:74-76`). But linked worktree paths come from
`git2::Worktree::path()`, which may return a differently-resolved path on
systems with symlinks. `display_path` compares these paths with `==`
(`presenter.rs:286`) and passes them to `pathdiff::diff_paths` (line 289) —
both operations produce incorrect results when paths disagree on symlink
resolution.

The fix is eager canonicalization of `WorktreeInfo.path` at construction time,
consistent with the existing convention for `entry.path`. Three lines to change
in `worktrees.rs` (lines 175, 200, 220), using
`.canonicalize().unwrap_or(wt_path)` to handle invalid worktrees gracefully.

## Goals / Success Criteria

- [ ] CLI integration tests exercise `gityard worktrees` with actual linked
      worktrees (at least: list worktrees, filter by dirty/gone, `--all` flag)
- [ ] CLI integration tests exercise `gityard trim` on repos that have linked
      worktrees (at least: trim with clean worktree, trim with dirty worktree
      protection)
- [ ] Submodule behavior is tested: worktree listing ignores submodule
      directories; modified submodules count as 1 dirty entry
- [ ] Bare repo behavior is tested: registration rejects them; `worktrees`
      with `--all` does not report a phantom main tree (fix or document)
- [ ] `WorktreeInfo.path` is canonicalized at construction time
- [ ] A test verifies `display_path` handles symlinked paths correctly
- [ ] The worktrees.rs test workaround (comparing `.canonicalize()` on both
      sides) is simplified now that paths are pre-canonicalized

## Non-Goals (Out of Scope)

- Full first-class support for submodules or bare repos — current behavior is
  acceptable; tests codify it
- Expanding the existing presenter golden tests (unless the symlink fix changes
  existing output)
- The `--all`/filter composition issue (TODO item #1) is a separate design
  decision

## Constraints

- CLI integration tests create real git repos in temp directories. Worktree
  tests will need to create linked worktrees, which requires careful cleanup
  (worktrees leave metadata in `.git/worktrees/`).
- The existing test helpers in `tests/cli.rs` create repos with specific
  structures. New helpers may be needed for worktree setup, but should follow
  the existing patterns.
- Invalid worktrees (directory deleted) cannot be canonicalized — the fallback
  to the raw path is intentional and acceptable.

## Resolved Questions

### Submodule behavior (investigated 2026-04-13)

gityard handles submodules correctly by default:
- `repo.worktrees()` returns only linked worktrees, not submodule directories
- `count_statuses()` reports a modified submodule as 1 dirty entry (the
  submodule path), without recursing into its contents
- No code path recurses into submodules
- `Repository::open()` on a submodule path treats it as a standalone repo

**Conclusion**: No code changes needed. Add tests to codify this behavior.

### Bare repository behavior (investigated 2026-04-13)

| Command | Behavior |
|---------|----------|
| `gityard add` | Rejects: checks `.git/` existence (`registry.rs:79`) |
| `gityard status` | Shows error: `count_statuses` fails with "cannot scan working directory for a bare repository" |
| `gityard trim` | Mostly works: branch ops are ref-only. HEAD switch fails gracefully (branch protected). |
| `gityard worktrees --all` | **Misleading**: reports a phantom "Main" working tree (`worktrees.rs:146-158`) |
| `gityard worktrees` (no `--all`) | Fine: returns empty list |

The registration gate (`registry.rs:79`) uses a `.git/` existence heuristic
rather than `is_bare()`. A bare repo could enter `repos.toml` via manual
editing.

**Conclusion**: The `worktrees --all` phantom entry is the one issue worth
fixing. Consider adding an `is_bare()` check in `list_worktrees` to skip the
main-tree entry, or strengthening the registration gate.

### display_path canonicalization (investigated 2026-04-13)

Eager canonicalization at `WorktreeInfo` construction time is the right
approach:
- `entry.path` is already canonicalized eagerly at `registry.rs:74-76` — this
  is the project convention
- Linked worktree paths from `git2::Worktree::path()` are the only exception
- Canonicalizing at construction fixes both `display_path` and trimmer display
  paths without patching each consumer
- Invalid worktrees use `.canonicalize().unwrap_or(wt_path)` since their
  directory is missing
- No consumers would be broken — paths are only used for display and filtering
  (which ignores the path field)

**Conclusion**: Canonicalize `WorktreeInfo.path` at 3 sites in `worktrees.rs`
(lines 175, 200, 220).

## References

- `tests/cli.rs` — existing CLI integration test patterns
- `src/worktrees.rs` — worktree listing and inspection
- `src/trimmer.rs` — trim pipeline with worktree-aware deletion
- `src/presenter.rs:display_path` — the symlink-affected function
- `src/registry.rs:73-81` — registration gate and path canonicalization
- TODO.md items #2, #3, #4
