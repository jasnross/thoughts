# mdview: Filter Hidden Files and Directories by Default

## Problem Statement

mdview's file watcher does not filter hidden files and directories. While the
initial directory scan uses `ignore::WalkBuilder` (which skips dotfiles/dotdirs
by default), the `notify`-based file watcher that runs during the session has no
such filtering. When a hidden markdown file is created or modified at runtime,
`handle_markdown_file_change` picks it up and adds it to the tracked file set,
causing it to appear in the sidebar and be servable via URL.

This is inconsistent: hidden files are excluded at startup but included if they
change while mdview is running.

## Motivation

- **Consistency**: The same filtering rules should apply regardless of when a
  file appears. Users expect dotfiles to stay hidden throughout the session, not
  just at startup.
- **Convention alignment**: Tools like `ripgrep`, `fd`, and `ls` all exclude
  hidden files by default. mdview should follow the same convention end-to-end.
- **Practical use case**: Documentation directories often contain hidden files
  (editor swap files, `.DS_Store`, dotfile-based configs) that should never show
  up in a viewer.

## Context

- `scan_markdown_files` (`app.rs:72`) uses `ignore::WalkBuilder` which skips
  hidden entries by default — the initial scan is already correct.
- `handle_markdown_file_change` (`app.rs:313`) processes any `notify` event for
  markdown files with no hidden-path check. In directory mode, it adds new files
  to tracking unconditionally.
- `serve_file` (`app.rs:635`) serves any tracked file — it has no independent
  hidden-path guard, but this is acceptable if the tracking layer filters
  correctly.
- The `notify` watcher (`app.rs:460`) watches the entire `base_dir` recursively,
  so it receives events for hidden paths that `WalkBuilder` would have skipped.

## Goals / Success Criteria

- [ ] Hidden markdown files created/modified at runtime are not added to
      tracking or shown in the sidebar
- [ ] A `--hidden` CLI flag opts in to including hidden files (both at scan time
      and during the watcher session)
- [ ] When `--hidden` is passed, `WalkBuilder` includes hidden entries and the
      watcher does not filter them out
- [ ] Existing tests for hidden-file exclusion continue to pass
- [ ] New test coverage for the watcher path (hidden file created at runtime is
      not tracked)

## Non-Goals (Out of Scope)

- Filtering based on `.gitignore` rules in the watcher (already handled by
  `WalkBuilder` for initial scan; watcher parity for gitignore is a separate
  concern)
- Filtering non-markdown hidden files (images, etc.) — only markdown tracking
  is affected
- Any UI to toggle hidden-file visibility at runtime

## Constraints

- The `--hidden` flag should follow the convention used by `ripgrep` and `fd`:
  short form `-H` is not available (already used for `--hostname`), so
  `--hidden` is long-form only
- `ignore::WalkBuilder` has a `.hidden(bool)` method that controls dotfile
  inclusion — the scan path can use this directly
- The watcher filtering needs a simple path-component check (any component
  starting with `.`) rather than reimplementing `WalkBuilder`'s logic

## Resolved Questions

- **`--hidden` vs `--no-ignore`**: `--hidden` controls dotfile visibility only.
  `.gitignore` behavior remains independent and unchanged. This matches the
  convention used by ripgrep and fd, where `--hidden` and `--no-ignore` are
  separate, composable flags. A `--no-ignore` flag could be added separately in
  the future if needed.
