# mdview: Hidden File Filtering — Implementation Plan

## Overview

Add consistent hidden-file filtering across mdview's entire file lifecycle (scan
and watcher), and a `--hidden` CLI flag to opt in to including hidden files. The
initial scan already excludes hidden files via `ignore::WalkBuilder`'s default
behavior, but the `notify`-based file watcher has no such filtering — hidden
markdown files created at runtime are picked up and added to tracking.

## Current State Analysis

- `scan_markdown_files` (`app.rs:72-88`) uses `WalkBuilder` with no explicit
  `.hidden()` call — it relies on the default (`true`, meaning hidden files are
  skipped). This works but is implicit.
- `handle_markdown_file_change` (`app.rs:313-340`) processes all `notify` events
  for markdown files. When `is_directory_mode` is true and the file is new, it
  calls `add_tracked_file` unconditionally — no hidden-path check.
- `add_tracked_file` (`app.rs:251-275`) canonicalizes the path and inserts into
  `tracked_files` with no hidden-path check.
- `MarkdownState` (`app.rs:163-168`) stores `base_dir`, `tracked_files`,
  `is_directory_mode`, and `change_tx`. No `show_hidden` field exists.
- Boolean config threading pattern: `Args` field → `main()` → `serve_markdown()`
  → `new_router()` → `MarkdownState::new()` → struct field. The `is_directory_mode`
  bool follows this exact path and is the model for `show_hidden`.
- One existing test (`app.rs:1070-1102`) verifies `scan_markdown_files` skips a
  `.hidden/` directory. No tests cover the watcher path.

### Key Discoveries:

- `WalkBuilder` has a `.hidden(bool)` method: `true` (default) means skip hidden
  files, `false` means include them. To support `--hidden`, call
  `.hidden(!show_hidden)`.
- The `-H` short flag is taken by `--hostname` (`main.rs:18`), so `--hidden` is
  long-form only.
- Test helpers `create_test_server_impl` (`app.rs:1322`) and
  `create_directory_server_impl` (`app.rs:1364`) call `new_router` directly and
  will need updating when `new_router`'s signature changes.

## Desired End State

- Hidden markdown files are excluded from both the initial scan and runtime
  watcher events, consistently.
- A `--hidden` CLI flag opts in to including hidden files everywhere.
- `scan_markdown_files` explicitly configures `WalkBuilder::hidden()` instead of
  relying on the implicit default.
- All affected code paths have test coverage.

**Verification:** `cargo test` passes, and manually running `mdview` against a
directory containing hidden markdown files confirms they are excluded from the
sidebar (and included when `--hidden` is passed).

## What We're NOT Doing

- Filtering hidden non-markdown files (images served via `serve_static_file_inner`)
- Adding `.gitignore` parity to the watcher (separate concern)
- Adding a `--no-ignore` flag (future work if needed)
- Runtime UI toggle for hidden-file visibility

## Implementation Approach

Thread a `show_hidden: bool` value from the CLI through to `MarkdownState`,
following the same path as `is_directory_mode`. Use it in two places: the
`WalkBuilder` configuration (explicit `.hidden()` call) and a new
`has_hidden_component` helper called in the watcher path.

## Implementation

### Overview

A single atomic change: add the CLI flag, thread it through, add the filter
helper, and wire it into the watcher path. All changes are in `src/main.rs` and
`src/app.rs`.

### Changes Required:

#### 1. CLI Flag

**File**: `src/main.rs`

- [x] Add `show_hidden: bool` field to the `Args` struct:

```rust
/// Show hidden files and directories (those starting with a dot)
#[arg(long = "hidden")]
hidden: bool,
```

- [x] Pass `args.hidden` to `scan_markdown_files` and `serve_markdown`:

```rust
// In the directory-mode branch:
let tracked_files = scan_markdown_files(&absolute_path, args.hidden)?;

// In the serve_markdown call:
serve_markdown(
    base_dir,
    tracked_files,
    is_directory_mode,
    args.hidden,
    args.hostname,
    args.port,
    args.open,
)
```

#### 2. `scan_markdown_files` — Accept `show_hidden` Parameter

**File**: `src/app.rs`

- [x] Add `show_hidden: bool` parameter to `scan_markdown_files` signature
- [x] Configure `WalkBuilder` explicitly:

```rust
pub(crate) fn scan_markdown_files(dir: &Path, show_hidden: bool) -> Result<Vec<PathBuf>> {
    let mut md_files = Vec::new();
    for entry in WalkBuilder::new(dir)
        .follow_links(false)
        .hidden(!show_hidden)
        .build()
    {
```

- [x] Update the comment at line 75-76 to reflect explicit configuration

#### 3. Hidden-Path Helper

**File**: `src/app.rs`

- [x] Add a `has_hidden_component` function (place near `is_markdown_file` at
  line 99 for locality):

```rust
/// Returns true if any component of `path` relative to `base` starts with a dot.
fn has_hidden_component(path: &Path, base: &Path) -> bool {
    path.strip_prefix(base)
        .unwrap_or(path)
        .components()
        .any(|c| {
            c.as_os_str()
                .to_str()
                .map_or(false, |s| s.starts_with('.'))
        })
}
```

This checks the relative path components (not the absolute prefix) so that a
`base_dir` like `/Users/foo/.notes/` doesn't cause everything to be filtered.

#### 4. Thread `show_hidden` Through State

**File**: `src/app.rs`

- [x] Add `show_hidden: bool` field to `MarkdownState` (line 163-168)
- [x] Add `show_hidden: bool` parameter to `MarkdownState::new` (line 171),
  store it in the struct
- [x] Add `show_hidden: bool` parameter to `serve_markdown` (line 513), pass
  it through to `new_router`
- [x] Add `show_hidden: bool` parameter to `new_router` (line 435), pass it
  through to `MarkdownState::new`

#### 5. Watcher Filtering

**File**: `src/app.rs`

- [x] Add hidden-path guard in the watcher path

  > **Deviation:** Instead of adding the check in `handle_markdown_file_change`
  > before the lock (as originally planned), the check was placed in
  > `add_tracked_file` instead. This is because the original approach would have
  > blocked refreshing already-tracked files whose names happen to start with a
  > dot (e.g., temp files created by `NamedTempFile` like `.tmpXXXXXX.md`). By
  > placing the check in `add_tracked_file`, only *new* file additions are
  > gated — files explicitly added at startup are always refreshable. A
  > secondary check was also added in `handle_markdown_file_change` after the
  > lock but only for the "new file in directory mode" branch, to skip
  > unnecessary work early.

#### 6. Update Test Helpers

**File**: `src/app.rs`

- [x] Update `create_test_server_impl` (line 1322) — pass `false` for
  `show_hidden` to `new_router`
- [x] Update `create_directory_server_impl` (line 1364) — pass `false` for
  `show_hidden` to `new_router`
- [x] Update any other call sites of `scan_markdown_files` or `new_router` to
  pass `show_hidden: false`

#### Tests

- [x] Update existing `test_scan_markdown_files_recursive` (line 1070) to pass
  `show_hidden: false` explicitly
- [x] Add `test_scan_markdown_files_show_hidden`: same setup as the existing
  test but passes `show_hidden: true` and asserts that `secret.md` IS included
- [x] Add `test_scan_hidden_file_at_root`: create a `.hidden.md` file directly
  in the temp dir (not in a hidden directory), verify it's excluded with
  `show_hidden: false` and included with `show_hidden: true`
- [x] Add `test_has_hidden_component`: unit test the helper with cases including
  hidden file at root, hidden directory with visible file, nested hidden
  directory, and no hidden components
- [x] Add `test_add_tracked_file_skips_hidden`: create a `MarkdownState` with
  `show_hidden: false`, call `add_tracked_file` with a path containing a hidden
  component, verify it's not added to `tracked_files`

### Success Criteria:

#### Automated Verification:

- [x] `cargo test` — all existing tests pass (no regressions)
- [x] `cargo test` — new hidden-file tests pass
- [x] `cargo build --release` — no warnings or errors
- [x] `cargo clippy` — no new warnings

#### Manual Verification:

- [x] Run `mdview <dir>` against a directory with hidden markdown files —
  confirm they don't appear in the sidebar
- [x] Run `mdview --hidden <dir>` — confirm hidden files DO appear
- [x] While mdview is running (without `--hidden`), create a `.test.md` file in
  the served directory — confirm it does NOT appear in the sidebar after
  live reload

## References

- Idea document: `thoughts/ideas/2026-04-13-mdview-hidden-file-filtering.md`
- `ignore` crate `WalkBuilder::hidden()` API
- Existing hidden-file test: `src/app.rs:1070-1102`
