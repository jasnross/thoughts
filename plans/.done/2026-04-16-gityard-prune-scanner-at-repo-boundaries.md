# Prune Scanner at Repo Boundaries Implementation Plan

## Overview

Change the scanner to stop descending into directories once they are identified
as git repositories. This prevents submodules and other nested repos from being
discovered and registered as independent tracked repos during `gityard add
<directory>`. A "skipping" message is printed for each nested repo that would
have been found, so the user knows what was excluded.

## Current State Analysis

The scanner (`src/scanner.rs:11-41`) walks a directory tree with `walkdir` and
collects every directory that contains a `.git` entry. Its only filter is
skipping entries named `.git` (to avoid descending into git internals). It does
**not** prune the subtree when it finds a repo — it continues walking into that
repo's subdirectories, discovering submodules and nested repos.

### Key Discoveries:

- `filter_entry` at `scanner.rs:17-20` controls both filtering *and* pruning —
  returning `false` prevents descending into that entry's subtree
- `cmd_add` at `main.rs:83` only enters the scan path when
  `path.join(".git").exists()` is false, so the walker's root directory is
  guaranteed to not be a repo — we don't need to handle the edge case of
  pruning the scan root
- `walkdir` visits parents before children (depth-first pre-order), so a repo
  discovered at depth N is added to the known set before its children at depth
  N+1 are evaluated by `filter_entry`
- The scanner returns `Vec<PathBuf>` — this needs to expand to also report
  skipped nested repos, following the project's "separate data collection from
  presentation" principle

## Desired End State

After this change:

1. `gityard add <directory>` discovers only top-level repos within the scanned
   directory — repos nested inside other repos (submodules, clones-within-clones)
   are excluded
2. Each excluded nested repo produces a stderr message:
   `skipping nested repo: <path>`
3. `gityard add <explicit-path>` (direct repo add) is unaffected — users can
   still explicitly add any repo, including submodules
4. Existing scanner unit tests are updated and new tests cover the pruning
   behavior

### Verification:

```sh
just check   # format + lint + build + test + license check
```

## What We're NOT Doing

- Adding `--include-submodules` flag or config option — explicit `gityard add
  <path>` covers the opt-in case
- Removing already-registered submodules from existing registries
- Changing fetch `--recurse-submodules` behavior
- Adding submodule-aware display to `gityard status`

## Implementation Approach

Use `RefCell<HashSet<PathBuf>>` to share state between `filter_entry` and the
main loop body. When a repo is discovered in the loop body, its walked path is
inserted into the `HashSet`. When `filter_entry` evaluates a child entry, it
checks whether the entry's parent is in the set — if so, it returns `false` to
prune the subtree. This is safe because `filter_entry`'s `RefCell::borrow()` is
dropped before the loop body calls `RefCell::borrow_mut()`.

This approach relies on `walkdir`'s default depth-first pre-order traversal
(parents yielded before children). Do not set `contents_first(true)` on the
walker, as that would reverse the order and break the pruning logic.

For skipped repo reporting: inside `filter_entry`, when an entry is being pruned
(parent is a discovered repo), check whether the entry itself has a `.git` — if
so, record it in a separate `RefCell<Vec<PathBuf>>`. This filesystem check is
only performed on immediate children of discovered repos (since deeper
descendants are already pruned), keeping the I/O cost minimal.

## Implementation

### Overview

This is a single atomic change: modify the scanner to prune at repo boundaries,
update its return type to report skipped repos, and update `cmd_add` to print
skipping messages.

### Changes Required:

#### 1. Scanner return type

**File**: `src/scanner.rs`

- [x] Add a `ScanResult` struct above the `scan` function:

```rust
/// Results from scanning a directory tree for git repositories.
pub struct ScanResult {
    /// Canonical paths to discovered top-level repositories.
    pub repos: Vec<PathBuf>,
    /// Non-canonical (walked) paths to nested repositories that were skipped
    /// (immediate children of discovered repos only — deeper transitive nesting
    /// is not reported because the entire subtree is pruned).
    ///
    /// Unlike `repos`, these are not canonicalized — walked paths are shorter
    /// and more readable in user-facing "skipping" messages.
    pub skipped: Vec<PathBuf>,
}
```

- [x] Change `scan` return type from `Result<Vec<PathBuf>>` to `Result<ScanResult>`

#### 2. Scanner pruning logic

**File**: `src/scanner.rs`

- [x] Add module-level imports: `use std::cell::RefCell;` and `use std::collections::HashSet;` (alongside existing imports at lines 1-4)
- [x] Inside `scan`, create shared state before the walker loop:

```rust
// RefCell needed for discovered/skipped because filter_entry captures them;
// repos is only mutated in the loop body so a plain Vec suffices.
let discovered = RefCell::new(HashSet::new());
let skipped = RefCell::new(Vec::new());
```

- [x] Expand the `filter_entry` closure to prune children of discovered repos:
  > **Deviation:** Clippy's `collapsible_if` lint required collapsing the nested
  > `if let Some(parent)` + `if discovered.borrow().contains(parent)` into a
  > single `if let ... && ...` let-chain.

```rust
.filter_entry(|e| {
    if e.file_name() == ".git" {
        return false;
    }
    if let Some(parent) = e.path().parent()
        && discovered.borrow().contains(parent)
    {
        if e.path().join(".git").exists() {
            skipped.borrow_mut().push(e.path().to_path_buf());
        }
        return false;
    }
    true
})
```

- [x] In the loop body, preserve the existing `is_dir()` guard and expand the `.git` block to also insert the walked path into the `discovered` set:

```rust
// existing guard unchanged
if !entry.file_type().is_dir() {
    continue;
}
if entry.path().join(".git").exists()
    && let Ok(canonical) = entry.path().canonicalize()
{
    // Insert the walked (non-canonical) path — filter_entry checks against
    // walked paths, not canonical ones. repos gets the canonical path.
    discovered.borrow_mut().insert(entry.path().to_path_buf());
    repos.push(canonical);
}
```

- [x] Sort both `repos` and `skipped` before returning, then return `ScanResult { repos, skipped: skipped.into_inner() }` (sorting ensures deterministic output for tests and user-facing messages)

#### 3. Update `cmd_add` to handle `ScanResult`

**File**: `src/main.rs`

- [x] Update the scan call site (line ~102) to destructure `ScanResult`:

```rust
let scan_result = scanner::scan(path, effective_depth)
    .with_context(|| format!("failed to scan {}", path.display()))?;

for skipped_path in &scan_result.skipped {
    eprintln!("skipping nested repo: {}", skipped_path.display());
}

if scan_result.repos.is_empty() {
    eprintln!("no git repositories found under {}", path.display());
    continue;
}

for repo_path in &scan_result.repos {
    // ... existing registration logic unchanged ...
}
```

#### 4. Update doc comment

**File**: `src/scanner.rs`

- [x] Update the `scan` doc comment to reflect the new behavior:

```rust
/// Scan a directory tree for git repositories.
///
/// Walks up to `max_depth` levels below `dir`, returning canonical paths
/// to directories that contain a `.git` entry. Once a repository is found,
/// its subtree is not descended into — any nested repositories (submodules,
/// clones-within-clones) are recorded in `ScanResult::skipped` instead.
```

#### Tests

**File**: `src/scanner.rs` (unit tests)

- [x] Update `test_scan_discovers_repos` to use `ScanResult` field access (`result.repos.len()`)
- [x] Update `test_scan_respects_depth_limit` to use `ScanResult` field access
- [x] Update `test_scan_skips_non_repo_directories` to use `ScanResult` field access
- [x] Add `test_scan_prunes_nested_repos`:

```rust
#[test]
fn test_scan_prunes_nested_repos() {
    let tmp = tempfile::tempdir().expect("tempdir");
    let parent = tmp.path().join("parent");
    init_git_repo(&parent);
    init_git_repo(&parent.join("nested-child"));
    init_git_repo(&parent.join("another-nested"));

    let result = scan(tmp.path(), 5).expect("scan");
    assert_eq!(result.repos.len(), 1);
    assert!(result.repos[0].ends_with("parent"));
    assert_eq!(result.skipped.len(), 2);
    assert!(result.skipped.iter().any(|p| p.ends_with("nested-child")));
    assert!(result.skipped.iter().any(|p| p.ends_with("another-nested")));
}
```

- [x] Add `test_scan_prunes_does_not_affect_siblings`:

```rust
#[test]
fn test_scan_prunes_does_not_affect_siblings() {
    let tmp = tempfile::tempdir().expect("tempdir");
    init_git_repo(&tmp.path().join("repo-a"));
    init_git_repo(&tmp.path().join("repo-b"));
    init_git_repo(&tmp.path().join("repo-a").join("nested"));

    let result = scan(tmp.path(), 5).expect("scan");
    assert_eq!(result.repos.len(), 2);
    assert!(result.repos.iter().any(|p| p.ends_with("repo-a")));
    assert!(result.repos.iter().any(|p| p.ends_with("repo-b")));
    assert_eq!(result.skipped.len(), 1);
}
```

**File**: `tests/cli.rs` (integration test)

- [x] Add `test_add_scan_skips_nested_repos`:

```rust
#[test]
fn test_add_scan_skips_nested_repos() {
    let tmp = TempDir::new().expect("tempdir");
    let config_dir = tmp.path().join("config");
    let scan_dir = tmp.path().join("workspace");
    init_git_repo(&scan_dir.join("parent-repo"));
    init_git_repo(&scan_dir.join("parent-repo").join("submodule"));
    init_git_repo(&scan_dir.join("sibling-repo"));

    gityard_with_config(&config_dir)
        .args(["add", scan_dir.to_str().expect("utf8")])
        .assert()
        .success()
        .stderr(
            predicate::str::contains("scan complete: 2 added")
                .and(predicate::str::contains("skipping nested repo:"))
                .and(predicate::str::contains("submodule")),
        );
}
```

### Success Criteria:

#### Automated Verification:

- [x] `just check` passes (format + lint + build + test + license check)
  > **Note:** `test_format_toml_omits_none_fields` in `config.rs` fails on
  > `main` before this change — pre-existing, unrelated to our work.
- [x] All existing scanner tests pass (updated for `ScanResult`)
- [x] New unit tests verify pruning behavior and sibling independence
- [x] New integration test verifies end-to-end skip messaging

## References

- Idea document: `thoughts/ideas/2026-04-16-gityard-exclude-submodules-from-add.md`
- Scanner implementation: `src/scanner.rs:11-41`
- `cmd_add` scan path: `src/main.rs:98-123`
- `walkdir` `filter_entry` docs: returns `false` to both exclude the entry and
  skip its subtree
