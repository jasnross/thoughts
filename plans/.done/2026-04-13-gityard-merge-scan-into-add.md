# Merge `scan` Into `add` Command — Implementation Plan

## Overview

Merge the `gityard scan` subcommand into `gityard add` so that `add` auto-detects
whether each path is a git repo (adds directly) or a plain directory (scans for
repos underneath). The `scan` subcommand is removed entirely.

## Current State Analysis

- `cmd_add` (`src/main.rs:64-86`) iterates paths, calls `Registry::add()` for
  each, prints per-path feedback, and saves. No scanning, no summary.
- `cmd_scan` (`src/main.rs:88-125`) calls `scanner::scan()` with a depth limit,
  registers discovered repos, prints per-path "added:" messages plus an aggregate
  summary line.
- `Registry::add()` (`src/registry.rs:73-101`) canonicalizes the path, validates
  `.git` exists, deduplicates, and returns `AddResult::Added` or
  `AddResult::AlreadyRegistered`.
- `scanner::scan()` (`src/scanner.rs:11-41`) walks a directory tree with
  `walkdir` up to `max_depth`, returning canonical paths to directories
  containing `.git`.
- The `Scan` CLI variant (`src/cli.rs:22-29`) accepts a single `dir: PathBuf`
  and an optional `--depth` flag defaulting to `config.scan.depth` (default 3).
- Two help-text strings in `main.rs` (lines 132, 465) reference
  `` `gityard scan` ``.

### Key Discoveries

- `cmd_add` accepts `&[impl AsRef<Path>]` — already supports multiple paths.
- `cmd_scan` accepts only a single `dir: PathBuf` — the merged `add` will
  handle mixed paths (repos + directories) naturally since each path is
  classified independently.
- `scanner::scan()` is a pure function with no side effects beyond filesystem
  reads — safe to call from `cmd_add` without any changes to `scanner.rs`.
- `cmd_add` currently exits 0 even when individual paths error (multi-path
  design) — this behavior should be preserved for the merged command.
- The `ScanConfig` struct and `config.scan.depth` field remain valid config
  since `add` will use them as the default depth. The doc comment on
  `ScanConfig` says "Configuration for the `scan` command" and should be updated.

## Desired End State

`gityard add <path>...` works for any mix of git repositories and plain
directories:

- Paths containing `.git` are added directly (current `add` behavior).
- Paths without `.git` are scanned for repos underneath (current `scan`
  behavior), using `--depth` or the configured default.
- The `scan` subcommand no longer exists.
- Output is per-path ("added:", "already registered:", "error:") with an
  aggregate summary when scanning was triggered.

### Verification

```sh
# Direct repo add still works
gityard add ~/projects/my-repo

# Directory scanning works through add
gityard add ~/projects --depth 2

# Mixed paths work
gityard add ~/projects/my-repo ~/other-projects

# scan subcommand is gone
gityard scan ~/projects  # exits with "unrecognized subcommand" error
```

## What We're NOT Doing

- Glob pattern support (`gityard add ~/projects/*/`)
- Changes to `Registry::add()` internals
- Changes to `scanner::scan()` internals
- Changes to the registry format (`repos.toml`)
- A `--no-scan` escape hatch flag
- Renaming `ScanConfig` or `config.scan` in the config file (that would be a
  breaking config change for no user-facing benefit)

## Implementation Approach

Each path passed to `cmd_add` is classified independently: if `path.join(".git")`
exists, add directly; otherwise, treat it as a scan directory. This reuses
`Registry::add()` for direct adds and `scanner::scan()` for directory scanning,
keeping both modules unchanged. The `--depth` flag moves from `Scan` to `Add`
and is only meaningful when at least one path triggers scanning.

## Implementation

### Overview

Remove the `Scan` subcommand, add `--depth` to `Add`, merge `cmd_scan` logic
into `cmd_add`, remove `cmd_scan`, and update integration tests.

### Changes Required

#### 1. CLI Definition

**File**: `src/cli.rs`

- [x] Remove the `Scan` variant from the `Command` enum (lines 22-29)
- [x] Add `--depth` flag to the `Add` variant:
  ```rust
  /// Register one or more repositories (directories without .git are scanned for repos)
  Add {
      /// Paths to git repositories or directories to scan
      paths: Vec<PathBuf>,
      /// Maximum directory depth when scanning a non-repo directory
      /// (defaults to config value, typically 3)
      #[arg(long)]
      depth: Option<usize>,
  },
  ```

#### 2. Command Handler

**File**: `src/main.rs`

- [x] Update the `Command::Add` match arm to destructure `depth`:
  ```rust
  Command::Add { paths, depth } => cmd_add(&config_dir, &config, &paths, depth),
  ```
- [x] Remove the `Command::Scan` match arm (line 36)
- [x] Rewrite `cmd_add` to accept `&Config` and `depth: Option<usize>`, with
  auto-detection logic:
  ```rust
  fn cmd_add(
      config_dir: &Path,
      config: &Config,
      paths: &[impl AsRef<Path>],
      depth: Option<usize>,
  ) -> Result<()> {
      let mut registry = Registry::load(config_dir)?;
      let mut scanned = false;
      let mut added = 0usize;
      let mut already_registered = 0usize;

      for path in paths {
          let path = path.as_ref();

          if path.join(".git").exists() {
              // Direct repo add
              match registry.add(path) {
                  Ok(AddResult::Added(canonical)) => {
                      eprintln!("added: {}", canonical.display());
                      added += 1;
                  }
                  Ok(AddResult::AlreadyRegistered) => {
                      eprintln!("already registered: {}", path.display());
                      already_registered += 1;
                  }
                  Err(e) => {
                      eprintln!("error: {}: {e}", path.display());
                  }
              }
          } else if path.is_dir() {
              // Scan directory for repos
              scanned = true;
              let effective_depth = depth.unwrap_or(config.scan.depth);
              let discovered = scanner::scan(path, effective_depth)
                  .with_context(|| format!("failed to scan {}", path.display()))?;

              if discovered.is_empty() {
                  eprintln!("no git repositories found under {}", path.display());
                  continue;
              }

              for repo_path in &discovered {
                  match registry.add(repo_path) {
                      Ok(AddResult::Added(_)) => {
                          eprintln!("added: {}", repo_path.display());
                          added += 1;
                      }
                      Ok(AddResult::AlreadyRegistered) => {
                          already_registered += 1;
                      }
                      Err(e) => {
                          eprintln!("error: {}: {e}", repo_path.display());
                      }
                  }
              }
          } else {
              eprintln!("error: {}: not a git repository or directory", path.display());
          }
      }

      registry
          .save(config_dir)
          .context("failed to save registry")?;

      if scanned {
          eprintln!(
              "scan complete: {added} added, {already_registered} already registered, {} total",
              registry.list().len()
          );
      }

      Ok(())
  }
  ```
- [x] Delete `cmd_scan` function entirely (lines 88-125)
- [x] Update help text at line 132 and line 465: replace
  `` `gityard add` or `gityard scan` `` with `` `gityard add` ``

#### 3. Config Doc Comment

**File**: `src/config.rs`

- [x] Update the doc comment on `ScanConfig` (line 64) from
  `Configuration for the \`scan\` command.` to
  `Configuration for directory scanning (used by \`add\` when a path is not a git repo).`

#### 4. No Changes Needed

- `src/scanner.rs` — called from `cmd_add` instead of `cmd_scan`, no API change
- `src/registry.rs` — `Registry::add()` unchanged
- `src/lib.rs` — `pub mod scanner;` still needed

#### Tests

**File**: `tests/cli.rs`

- [x] Update `test_scan_discovers_repos` (line 107) to use `add` instead of
  `scan`:
  ```rust
  gityard_with_config(&config_dir)
      .args(["add", scan_dir.to_str().expect("utf8")])
      .assert()
      .success()
      .stderr(
          predicate::str::contains("added:")
              .and(predicate::str::contains("2 added"))
              .and(predicate::str::contains("2 total")),
      );
  ```
- [x] Add test: `add` with explicit `--depth` flag limits scan depth
  ```rust
  #[test]
  fn test_add_scan_respects_depth_flag() {
      let tmp = TempDir::new().expect("tempdir");
      let config_dir = tmp.path().join("config");
      let scan_dir = tmp.path().join("workspace");
      // depth 1: directly under scan dir
      init_git_repo(&scan_dir.join("shallow"));
      // depth 3: too deep for --depth 1
      let deep = scan_dir.join("a").join("b").join("deep-repo");
      init_git_repo(&deep);

      gityard_with_config(&config_dir)
          .args(["add", "--depth", "1", scan_dir.to_str().expect("utf8")])
          .assert()
          .success()
          .stderr(predicate::str::contains("1 added"));
  }
  ```
- [x] Add test: mixed paths (one direct repo, one scan directory)
  ```rust
  #[test]
  fn test_add_mixed_repo_and_directory() {
      let tmp = TempDir::new().expect("tempdir");
      let config_dir = tmp.path().join("config");
      let direct_repo = tmp.path().join("direct-repo");
      init_git_repo(&direct_repo);
      let scan_dir = tmp.path().join("workspace");
      init_git_repo(&scan_dir.join("scanned-repo"));

      gityard_with_config(&config_dir)
          .args([
              "add",
              direct_repo.to_str().expect("utf8"),
              scan_dir.to_str().expect("utf8"),
          ])
          .assert()
          .success()
          .stderr(
              predicate::str::contains("added: ")
                  .count(2),
          );

      // Both repos should be registered
      gityard_with_config(&config_dir)
          .args(["list"])
          .assert()
          .success()
          .stdout(
              predicate::str::contains("direct-repo")
                  .and(predicate::str::contains("scanned-repo")),
          );
  }
  ```
- [x] Add test: path that is neither a repo nor a directory produces an error
  ```rust
  #[test]
  fn test_add_nonexistent_path() {
      let tmp = TempDir::new().expect("tempdir");
      let config_dir = tmp.path().join("config");

      gityard_with_config(&config_dir)
          .args(["add", "/nonexistent/path"])
          .assert()
          .success()
          .stderr(predicate::str::contains("error:"));
  }
  ```
- [x] Add test: `scan` subcommand no longer exists
  ```rust
  #[test]
  fn test_scan_subcommand_removed() {
      gityard()
          .args(["scan", "/tmp"])
          .assert()
          .failure()
          .stderr(predicate::str::contains("unrecognized subcommand"));
  }
  ```
- [x] Verify `test_add_rejects_bare_repo` (line 1255) still passes — bare repos
  don't have a `.git` directory, so they'll now be treated as scan directories
  (finding nothing), which changes the error message. Update the assertion if
  needed: the current test expects `"not a git repository"` from
  `Registry::add()`, but after this change a bare repo directory will be scanned
  (finding no repos) and print `"no git repositories found under"` instead.
  Update the expected stderr:
  ```rust
  .stderr(predicate::str::contains("no git repositories found under"));
  ```
  Then verify the bare repo is still NOT registered (this part of the test
  stays the same).

### Success Criteria

#### Automated Verification

- [x] All tests pass: `just test`
- [x] Clippy passes: `just lint`
- [x] Formatting passes: `cargo fmt --check`
- [x] Full suite passes: `just check`

#### Manual Verification

None required — all behavior is covered by automated tests.

## References

- Idea document: `thoughts/ideas/2026-04-12-gityard-merge-scan-into-add.md`
- Learnings applied: atomic commits for interdependent changes (`thoughts/learnings/git-commit-strategies.md`) — all changes should land in a single commit since any intermediate state would leave the CLI broken
