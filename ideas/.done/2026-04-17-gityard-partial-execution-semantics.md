# gityard: rework partial-execution semantics for bulk commands

## Problem Statement

Two related pain points affect `gityard`'s bulk git operations (`pull`, future
commands sharing `ExecArgs`):

1. **`gityard pull --allow-partial` still errors on branches without an
   upstream.** The pre-flight check in `src/operation.rs:47-61` only catches
   `is_dirty`. Branches with no tracking remote slip through as `Ready`, then
   `git pull` itself fails at runtime (`ExecOutcome::Failed`). Because
   `--allow-partial` only governs the pre-flight *block* phase, it has no effect
   here, and any `Failed` outcome causes `main.rs` to exit non-zero.

2. **`--allow-partial` is opt-in, but the common case wants partial execution.**
   Users have to remember the flag on every run, and the natural inverse
   (`--no-allow-partial`) is awkward.

Reproduction: `gityard pull --allow-partial` on a registry that includes a
branch without upstream yields:

```
 ✗ agentspec: There is no tracking information for the current branch.
```

…as a runtime git failure, and the command exits non-zero even though every
other repo succeeded.

## Motivation

- Everyday bulk operations should "just do what they can" by default, surfacing
  skipped/blocked repos clearly but not aborting the run.
- Branches without upstream are often intentionally local (topic branches,
  scratch work). Treating them as hard failures is noisy and unhelpful.
- Pre-flight blocking and runtime failure should be distinguishable in the
  output: a predictable, pre-detected block should look the same as a dirty
  tree, not like a git crash.
- Inverting the flag gets naming right for the common path — users opt into
  strictness when they want CI-style guarantees, not the reverse.

## Context

- `src/cli.rs:57-66` — `ExecArgs { allow_partial, group }` is shared by `fetch`
  and `pull` subcommands.
- `src/operation.rs:47-61` — `preflight()` for `Pull` checks only `is_dirty`.
  `Fetch` preflight is trivially `Ready`, so `--allow-partial` is effectively a
  no-op for fetch today.
- `src/executor.rs:50-95` — `allow_partial` controls whether a `Blocked`
  pre-flight outcome aborts every other repo (current all-or-nothing default).
- `RepoState` already has `ahead_remote: Option<i32>` and
  `behind_remote: Option<i32>`. `None` is a practical proxy for "no upstream,"
  so the pre-flight check doesn't need new data collection — only a new branch
  in the match.
- Visual language already established: `✗` (red) for blocked, `─` (yellow) for
  skipped, `✓` (green) for success, `✗` (red) for failed. The user prefers the
  no-upstream case to appear as `✗` alongside a descriptive reason, consistent
  with the dirty-tree block.
- Recent commit history (`84b591d`, `602f766`) trimmed surface area (removed
  `push`/`checkout`), so the flag rename is low-churn today — but
  forward-looking, since any future bulk command will inherit `ExecArgs`.

## Goals / Success Criteria

- [ ] `gityard pull` (and any command using `ExecArgs`) defaults to
      partial-allowed semantics — no flag needed.
- [ ] Branches without an upstream are detected in pre-flight and reported as
      `Blocked` with the message "no upstream tracking branch", displayed with
      the existing red-X visual.
- [ ] A new `--strict` flag replaces `--allow-partial`, opting into
      all-or-nothing abort when any repo is `Blocked`.
- [ ] Per-command strict default is configurable in TOML via a `strict` key on
      each command section (e.g. `[pull]\nstrict = false`). CLI `--strict` /
      `--no-strict` override per invocation.
- [ ] Exit-code policy:
  - Exit non-zero on any runtime `Failed` outcome, regardless of mode.
  - Exit non-zero if `--strict` aborts due to a `Blocked` repo.
  - Exit 0 otherwise (including partial runs with `Blocked` repos in non-strict
    mode).
- [ ] Visual output and summary line remain consistent — a blocked repo always
      looks the same regardless of which flag path was taken.
- [ ] `fetch` keeps its current always-`Ready` pre-flight; no new checks added
      as part of this work.

## Non-Goals

- Not redesigning the outcome taxonomy (`Success` / `Failed` / `Blocked` /
  `Skipped` stays).
- Not adding new bulk commands or new pre-flight checks beyond the no-upstream
  case for `pull`.
- Not introducing `--allow-partial` as a deprecated alias. Breaking change is
  acceptable — the user explicitly wants the inversion.
- Not changing how `git pull` itself is invoked (no `--ff-only`, no autostash,
  no merge strategy changes).

## Proposed Direction (sketch — detailed design belongs in planning)

1. Add a `Pull` pre-flight branch that returns `Blocked { reason: "no upstream
   tracking branch" }` when the current branch has no tracking remote
   (detectable via `RepoState.ahead_remote.is_none() &&
   behind_remote.is_none()`, or by a direct check on the branch's upstream).
2. Rename `ExecArgs::allow_partial` → `ExecArgs::strict` and invert the
   semantics in `executor::execute()`. Support `--strict` and `--no-strict` at
   the CLI.
3. Add a per-command config section with a `strict` key (e.g. `[pull] strict =
   false`). CLI flag overrides config; config overrides built-in default.
   Built-in default: `false` (partial allowed).
4. Update `main.rs` exit-code policy:
   - Any `Failed` outcome → exit non-zero.
   - `--strict` aborted with `Blocked` → exit non-zero.
   - Otherwise → exit 0.
5. Document the breaking change in `README.md`; use a `feat!:` conventional
   commit (or `feat:` + `BREAKING CHANGE:` trailer) when landing.

No `BlockReason` enum is needed — strict mode fires uniformly on any `Blocked`,
so the existing free-form `reason: String` is sufficient. This keeps the
change surface small.

## Constraints

- This is a breaking change for users/scripts relying on `--allow-partial` or
  on current exit-code behavior. README update and a `feat!:` / `BREAKING
  CHANGE:` commit are mandatory.
- `Operation::Pull` is a unit variant — pre-flight sees only the operation
  kind, no args. The new block reason can be derived purely from `RepoState`,
  so no shape change to `Operation` is needed.
- Per-command config section needs to coexist with the existing config
  structure; verify current `src/config.rs` layout can absorb `[pull]` and
  `[fetch]` sections cleanly.

## Affected Components

- `src/cli.rs` — rename `allow_partial` → `strict` on `ExecArgs`; add
  `--no-strict` for override symmetry.
- `src/operation.rs` — add no-upstream branch to `preflight()` for `Pull`.
- `src/executor.rs` — invert `allow_partial` → `strict`; update doc comments
  and tests.
- `src/config.rs` — add `[pull]` and `[fetch]` sections with a `strict: bool`
  field (defaults to `false`).
- `src/main.rs` — exit-code policy; wire config → `ExecArgs.strict` default.
- `tests/cli.rs`, `src/executor.rs` tests, `src/operation.rs` tests — update
  existing coverage, add cases for no-upstream pre-flight and strict/non-strict
  executor behavior.
- `README.md` — flag rename, new default behavior, new config fields.

## References

- `src/operation.rs:47-61` — current pre-flight logic.
- `src/executor.rs:74-95` — current all-or-nothing gate.
- `src/cli.rs:59-66` — `ExecArgs` definition.
- Recent commits `84b591d`, `602f766` — trimmed surface area (push/checkout
  removed), making now a clean moment to adjust shared flag semantics.
