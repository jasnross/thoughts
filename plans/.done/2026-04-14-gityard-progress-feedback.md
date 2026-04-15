# gityard: Progress Feedback for Multi-Repo Operations — Implementation Plan

## Overview

Add an in-place progress counter to all executor-based commands (fetch, pull,
push, checkout) so users see `[3/12] ✓ dotfiles` on stderr as each repo
completes, instead of silence until every repo finishes.

The approach uses `std::sync::mpsc` channels inside `execute()` so the main
thread receives results as threads complete, and an optional progress callback
lets the caller render updates. No new crate dependencies are needed.

## Current State Analysis

The executor (`src/executor.rs:29-110`) spawns one thread per repo via
`thread::scope`, joins all handles in spawn order, and returns an
`ExecutionReport`. The caller `cmd_exec()` (`src/main.rs:257-306`) prints the
report to stdout in one shot after all threads have joined. No output is
produced during execution.

### Key Discoveries

- `execute()` returns `ExecutionReport` with outcomes in spawn order — changing
  to channel-based collection means outcomes arrive in completion order instead.
  This is acceptable (and arguably better — fast repos appear first in the
  report).
- `run_git()` (`src/executor.rs:113-135`) is a pure function with no side
  effects — safe to call from channel-sending threads without modification.
- The `colored` crate's global state is managed via `set_override`/`unset_override`
  with a `Mutex` in presenter tests (`src/presenter.rs:690`). The progress
  callback will not touch color state, so no conflict.
- `unwrap_used`, `expect_used`, and `panic` are denied outside tests
  (`Cargo.toml:36-38`). Channel operations must use `let _ = tx.send(...)` for
  the send side (ignore errors if receiver dropped) and iterate `rx` rather
  than calling `recv().unwrap()`.
- Edition 2024 confirms `std::io::IsTerminal` is available (stable since Rust
  1.70).

## Desired End State

When a user runs an executor-based command against multiple repos, they see
incremental progress on stderr as each repo completes:

```
$ gityard fetch
running: gityard fetch
[1/5] ✓ dotfiles
[2/5] ✓ gityard        ← in-place update, overwrites previous line
[3/5] ✓ blog
[4/5] ✗ api-server
[5/5] ✓ scripts
 ✓ dotfiles             ← final report on stdout (existing behavior)
 ✓ gityard
 ✓ blog
 ✗ api-server: connection timed out
 ✓ scripts

 4 succeeded · 1 failed
```

The progress line updates in-place via `\r` + ANSI clear-to-EOL (`\x1b[K`).
When stderr is not a TTY or there is only one repo, no progress line is shown.

### Verification

- `just check` passes (format, lint, build, test, license check)
- Running `gityard fetch` against 2+ registered repos shows the progress
  counter on stderr, with the final report on stdout
- Running `gityard fetch 2>/dev/null` produces only the final report on stdout
  (progress is suppressed when stderr is not a TTY)
- Running with a single registered repo produces no progress line

## What We're NOT Doing

- Adding `indicatif` or any external progress bar crate
- Showing per-repo git transfer stats (e.g., objects received)
- Adding `--quiet` / `--verbose` flags
- Changing the concurrency model (no async/tokio)
- Sorting the final report — completion order is the new default

## Implementation

### Overview

Three files change: `executor.rs` gains channel-based execution with a progress
callback, `main.rs` wires up TTY detection and the progress closure, and tests
cover the new callback mechanism.

### Changes Required

#### 1. Channel-based execution with progress callback (`src/executor.rs`)

**Modify `execute()` signature** to accept an optional progress callback:

```rust
pub fn execute(
    op: &Operation,
    entries: &[&RepoEntry],
    states: &[RepoState],
    allow_partial: bool,
    on_progress: Option<&dyn Fn(usize, usize, &str, &ExecOutcome)>,
) -> ExecutionReport
```

The callback receives `(completed_count, total_count, alias, outcome)`.

- [x] Add `on_progress` parameter to `execute()`
  > **Deviation:** Extracted a `ProgressCallback<'a>` type alias to satisfy
  > clippy's `type_complexity` lint on the `Option<&dyn Fn(...)>` type.
- [x] Replace the current `thread::scope` block (lines 77-107) with a
      channel-based implementation:
  - Separate preflight results into ready vs. non-ready (blocked/skipped)
      outcomes upfront
  - Create an `mpsc::channel()`
  - Inside `thread::scope`, clone `tx` for each ready repo and spawn threads
      that send `(alias, ExecOutcome)` through the channel
  - Drop the original `tx` after spawning so the receiver terminates when all
      threads complete
  - Receive from `rx` in a loop, calling `on_progress` for each result and
      collecting outcomes
  - Combine non-ready outcomes with ready outcomes into the final
      `ExecutionReport`
- [x] The early-return path for all-or-nothing blocking (lines 50-67) does not
      need channels — it returns immediately with no execution. Pass non-ready
      outcomes through `on_progress` if the callback is provided, so the caller
      sees blocked/skipped repos in the progress count too.

#### 2. Progress display in `cmd_exec()` (`src/main.rs`)

- [x] Add `use std::io::IsTerminal` import
  > **Deviation:** Only `IsTerminal` was needed, not `Write`.
- [x] After the `eprintln!("running: gityard {op}")` line (line 284), determine
      whether to show progress:
  ```rust
  let total = entry_refs.len();
  let show_progress = std::io::stderr().is_terminal() && total > 1;
  ```
- [x] Build the progress callback closure:
  ```rust
  let on_progress: Option<&dyn Fn(usize, usize, &str, &ExecOutcome)> =
      if show_progress {
          &|completed, total, alias, outcome| {
              let symbol = match outcome {
                  ExecOutcome::Success { .. } => "✓",
                  ExecOutcome::Failed { .. } | ExecOutcome::Blocked { .. } => "✗",
                  ExecOutcome::Skipped { .. } => "─",
              };
              eprint!("\r[{completed}/{total}] {symbol} {alias}\x1b[K");
          }
      } else {
          None
      };
  ```
- [x] Pass `on_progress` to `executor::execute()`
- [x] After `execute()` returns, clear the progress line if it was shown:
  ```rust
  if show_progress {
      eprint!("\r\x1b[K");
  }
  ```

#### 3. Update existing call sites

- [x] Update all existing calls to `executor::execute()` to pass `None` for
      `on_progress` if there are any outside `cmd_exec()` (verified — only one
      call site at `main.rs:287`, now updated with the progress callback)

#### Tests

- [x] **Unit test: callback receives correct counts and aliases**
      (`src/executor.rs`). Construct entries with fake paths (they'll fail
      `run_git` but that's fine — we're testing the callback mechanism). Use a
      `Mutex<Vec<(usize, usize, String)>>` to capture callback invocations.
      Assert that `completed` increments from 1 to N and `total` stays constant.
- [x] **Unit test: callback not invoked when `None`**. Call `execute()` with
      `on_progress: None` and verify it still returns a valid `ExecutionReport`
      (regression test for the existing behavior).
- [x] **Unit test: all-or-nothing blocking still calls callback**. With
      `allow_partial=false` and a blocked repo, verify the callback is called
      for all repos (all as `Blocked`).
- [x] **Integration test: final report unchanged** (`tests/cli.rs`). The
      existing `test_fetch_succeeds` and `test_pull_blocked_when_dirty` tests
      already assert stdout content. Since stderr is piped (not a TTY) in
      `assert_cmd`, progress is suppressed, and existing assertions remain
      valid. No changes needed to existing integration tests — all 51 pass.

### Success Criteria

#### Automated Verification

- [x] `just check` passes (includes `cargo fmt --check`, `cargo clippy --all-targets`, `cargo build`, `cargo test`, `cargo deny check licenses`)
- [x] New unit tests pass: callback count/alias assertions, None callback regression, all-or-nothing callback test

#### Manual Verification

- [x] `gityard fetch` with 2+ repos shows `[N/M] ✓ name` progress on stderr, then the final report on stdout
- [x] `gityard fetch` with 1 repo shows no progress line
- [x] `gityard fetch 2>/dev/null` shows only the final report (progress suppressed when stderr is not a TTY)

## Performance Considerations

The `mpsc::channel` adds negligible overhead — each thread sends exactly one
message. The main thread's receive loop is bounded by I/O (git subprocess
completion), not channel throughput. The `eprint!` calls add one stderr write
per repo completion, which is trivially fast compared to the git operations
themselves.

## References

- Idea document: `thoughts/ideas/2026-04-14-gityard-progress-feedback.md`
- Executor implementation: `src/executor.rs:29-135`
- `cmd_exec()` call site: `src/main.rs:257-306`
- `std::sync::mpsc` docs: https://doc.rust-lang.org/std/sync/mpsc/
- `IsTerminal` trait: https://doc.rust-lang.org/std/io/trait.IsTerminal.html
