# gityard: Progress Feedback for Multi-Repo Operations

## Problem Statement

When running `gityard fetch`, `gityard pull`, or other executor-based commands
across many repositories, the user sees **zero output** until every repo has
finished. For network-bound operations like fetch and pull, this can mean
several seconds (or longer on slow connections) of silence with no indication
that the tool is working. The user cannot distinguish "working" from "stuck."

## Motivation

- **Perceived responsiveness**: Even if total wall-clock time doesn't change,
  incremental feedback makes the tool feel faster and more trustworthy.
- **Confidence the tool is working**: Without output, users may Ctrl-C and
  re-run, or assume something went wrong.
- **Debugging slow repos**: Seeing which repo just completed (or is currently
  running) helps identify bottlenecks — e.g., a repo on a slow remote.
- **Consistency with expectations**: Most CLI tools that perform multiple
  network operations provide some form of progress indication (e.g., `cargo`,
  `apt`, `brew`).

## Context

The executor (`src/executor.rs`) spawns all repos in parallel via
`std::thread::scope`, collects results into an `ExecutionReport`, and only then
calls `format_report()` to print the outcome table. There is no mechanism for
threads to report completion as they finish — the main thread blocks on
`join()` for all handles before producing any output.

The `run_git()` function captures stdout/stderr into `ExecOutcome` variants,
so individual repo results are available as soon as each thread completes,
but nothing currently consumes them incrementally.

## Goals / Success Criteria

- [ ] User sees in-place progress updates as repos complete (e.g.,
      `[3/12] ✓ dotfiles`)
- [ ] Progress feedback applies to **all** executor-based commands (fetch,
      pull, push, checkout, etc.), not just a subset
- [ ] The final outcome report is still printed after all repos complete
      (existing behavior preserved)
- [ ] Progress updates write to stderr so they don't interfere with stdout
      content (important for future `--json` or piped usage)
- [ ] No progress line is shown when operating on a single repo (unnecessary
      noise)

## Non-Goals (Out of Scope)

- Full progress bars with ETA/throughput (indicatif-style) — the counter
  approach is sufficient for now
- Per-repo streaming of git's own progress output (e.g., `git fetch` transfer
  stats)
- Async/tokio migration — the current `std::thread::scope` approach is fine;
  this is about reporting, not concurrency model changes
- Quiet/verbose mode flags — may come later but not part of this idea

## Proposed Solutions

### Option A: Channel-based incremental reporting

Threads send `(alias, ExecOutcome)` through a channel as they complete. The
main thread receives from the channel in a loop, updates an in-place progress
line on stderr, and collects results for the final report.

**Pros**: Clean separation, no shared mutable state, idiomatic Rust.
**Cons**: Adds a channel dependency (though `std::sync::mpsc` is in stdlib).

### Option B: Shared atomic counter with periodic polling

A shared `AtomicUsize` tracks completed count. The main thread polls it on a
short interval and updates the display.

**Pros**: Very simple, no channel overhead.
**Cons**: Doesn't naturally provide the repo name for the progress line;
polling adds a small delay before updates appear.

## Constraints

- Must work with the existing `std::thread::scope` parallelism model
- Progress output must go to stderr (per gityard's convention of using stderr
  for status and stdout for data)
- Terminal cursor manipulation (carriage return for in-place updates) should
  degrade gracefully when stderr is not a TTY (e.g., piped to a file) — in
  that case, either suppress the progress line or print non-updating lines

## Decisions

- **Progress line shows the most recently completed repo name** (e.g.,
  `[3/12] ✓ dotfiles`). Simpler than tracking in-flight threads, and the
  name is naturally available from each thread's completion message.
- **Suppress progress entirely when stderr is not a TTY.** The final report
  still prints. This follows the convention of cargo, git, and docker — no
  carriage-return noise in piped/logged output.

## Affected Systems/Services

- `src/executor.rs` — primary change site (thread communication and progress
  reporting)
- `src/main.rs` — may need to pass a writer/callback for progress output
- `src/presenter.rs` — may gain a progress formatting function, or this may
  live in executor
